# HackTheBox - Cap - Easy - ctxzero

---

### How many TCP ports are open?

nmap -sS 10.10.10.245    
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-28 17:07 EDT
Nmap scan report for 10.10.10.245
Host is up (0.044s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```
Answer: 3

---

### After running a "Security Snapshot", the browser is redirected to a path of the format /[something]/[id], where [id] represents the id number of the scan. What is the [something]?
so we visited the website and clicked on that:

<img width="237" height="56" alt="image" src="https://github.com/user-attachments/assets/1174be80-edef-4c1e-84de-ec43ad4b8bb9" />

and got an URL like:

<img width="150" height="28" alt="image" src="https://github.com/user-attachments/assets/16632f67-b38e-4444-91a6-2b8276250d8d" />

Answer: data

---

### Are you able to get to other users' scans?

So we took a look at:

<img width="229" height="51" alt="image" src="https://github.com/user-attachments/assets/95493ef5-cd66-459e-96ab-3bd40f1b721c" />

and we saw the other guy his scans in the netstat

Answer: yes

---

### What is the ID of the PCAP file that contains sensative data?

So what we did then was went back to this Tab:

<img width="213" height="53" alt="image" src="https://github.com/user-attachments/assets/f5cd26cd-0847-4e4c-b1fa-2296a8ffcf75" />

our URL path was: /data/[id] so we started checking for every number, luckily at the first number 0 we got our result,
so when we changed our path to /data/0 we got informations like in the picture below

<img width="438" height="600" alt="image" src="https://github.com/user-attachments/assets/14973b30-3a71-466c-83ef-e5aa82cf5e31" />

we clicked on Download and got a .pcap file, we opened it in Wireshark and we quickly saw login credentials for FTP in plaintext.

<img width="1690" height="386" alt="image" src="https://github.com/user-attachments/assets/d3f71d80-9df1-4b4d-9c24-c00ece9cce72" />

Answer: 0

---

### Which application layer protocol in the pcap file can the sensetive data be found in?

We saw that in our Wireshark Capture.

Answer: FTP

---

### We've managed to collect nathan's FTP password. On what other service does this password work?

so we knew it only can be ssh, because of our nmap scan, so we logged in via ssh with the credentials from the Wireshark capture.

<img width="655" height="141" alt="image" src="https://github.com/user-attachments/assets/e7ca2c50-a52a-4070-a117-e1a7ca18ab6b" />

Answer: SSH

---

### Submit the flag located in the nathan user's home directory.

<img width="265" height="30" alt="image" src="https://github.com/user-attachments/assets/554834ee-0c6f-4130-9cad-b6e979a2e8e9" />

---

### What is the full path to the binary on this machine has special capabilities that can be abused to obtain root privileges?

```
nathan@cap:~$ getcap -r / 2>/dev/null
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

I immediately knew it's python, but to verify that we could just look it up on GTFOBins.

so we copied the payload from GTFOBins, and changed it like that:

<img width="692" height="49" alt="image" src="https://github.com/user-attachments/assets/cb3f8aee-0012-45ab-bff7-513731aa6066" />

Answer: /usr/bin/python3.8

---

### Submit the flag located in root's home directory.

<img width="274" height="55" alt="image" src="https://github.com/user-attachments/assets/21038292-bee6-486a-9019-3fc22e63ff73" />













