# Simple CTF - TryHackMe Write-up

A detailed write-up on how to solve the **Simple CTF** machine on TryHackMe. This lab covers network reconnaissance, enumerating an anonymous FTP service, targeted brute-force attacks against SSH on a non-standard port, and local privilege escalation by abusing permitted Sudo privileges (`vim`).

* **Author:** HackSysX70
* **Platform:** TryHackMe
* **Difficulty:** Easy
* **OS:** Linux

---

## 1. Reconnaissance (Recon)

We begin by performing a full port scan on the target machine using **Nmap** to identify all exposed services and running ports.

```bash
nmap -p- -sS -T4 -Pn -n -vvv -sVC <TARGET_IP> -oN escaneo.txt
```

### Explanation of Parameters Used

| Parameter | Description |
| :--- | :--- |
| `-p-` | Scans **all 65,535 TCP ports** to ensure services running on non-standard ports are not missed. |
| `-sS` | Performs a *SYN Stealth* scan (half-open), which is fast and does not complete the full TCP 3-way handshake. |
| `-T4` | Sets the timing template to level 4 (aggressive) to speed up the scan in stable lab environments. |
| `-Pn` | Treats the host as online, **skipping ICMP ping discovery** in case a firewall blocks ping requests. |
| `-n` | Disables DNS name resolution, significantly reducing overall scan time. |
| `-vvv` | Enables maximum verbosity to display open ports in real-time as they are discovered. |
| `-sVC` | Combines `-sV` (service version detection) and `-sC` (default NSE scripts for common vulnerabilities). |
| `-oN escaneo.txt` | Saves the complete output in normal format into `escaneo.txt` for future reference. |

### Scan Results Summary

| Port | State | Service | Details |
| :--- | :--- | :--- | :--- |
| **21/TCP** | Open | `ftp` | FTP Service (Anonymous login allowed) |
| **80/TCP** | Open | `http` | HTTP Service (Apache Web Server) |
| **2222/TCP** | Open | `ssh` | SSH Service (Relocated non-standard port) |

---

## 2. Enumeration & Discovery (Anonymous FTP)

Upon detecting port `21` open, we accessed the FTP service using the default `anonymous` account. Initially, data transmission failed due to network constraints within the TryHackMe environment, requiring us to execute the **`passive`** command to enable passive transfer mode.

Once inside, we located a directory named `pub/` containing a text file named `ForMitch.txt`. Downloading and reading the file revealed the following developer note:

```text
Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, 
and the password is so weak... i cracked it in seconds. Gosh... what a mess!
```

### Key Takeaways
* A valid system user named **`mitch`** exists on the target host.
* The user reused their password across multiple services (credential reuse).
* The password is weak and vulnerable to dictionary-based brute-force attacks.

---

## 3. Exploitation (SSH Brute-Force & Initial Access)

Armed with a confirmed username (`mitch`), knowing that SSH was running on the alternative port **2222**, and knowing the password was weak, we launched a targeted brute-force attack using **Hydra** paired with the `rockyou.txt` wordlist.

```bash
sudo hydra -l mitch -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>:2222 -t 16 -f -I
```

### Explanation of Parameters Used

| Parameter | Description |
| :--- | :--- |
| `-l mitch` | Targets the single specific username `mitch`. |
| `-P /.../rockyou.txt` | Loads the path to the password wordlist used for the attack. |
| `ssh://<TARGET_IP>:2222` | Directs the brute-force attack against the SSH service on alternative port `2222`. |
| `-t 16` | Runs 16 parallel tasks to speed up the attack while avoiding connection drops. |
| `-f` | Instructs Hydra to stop immediately once the first valid credential pair is found. |
| `-I` | Ignores any previous session restore files, forcing a clean run from scratch. |

### Discovered Credentials

| Service | Port | Username | Password |
| :--- | :--- | :--- | :--- |
| **SSH** | `2222` | `mitch` | `secret` |

We established a remote terminal session via SSH, explicitly specifying the custom port with `-p`:

```bash
ssh mitch@<TARGET_IP> -p 2222
```

### Terminal Stabilization (TTY Spawn)
Upon logging in, the shell provided by the server was limited and non-interactive. To stabilize our session and spawn a fully functional Bash TTY, we executed:

```bash
script /dev/null -c bash
```

We then navigated to `mitch`'s home directory and read the user flag:

```bash
cat user.txt
```

* **User Flag:** `THM{REDACTED_USER_FLAG}` *(Hidden in accordance with TryHackMe write-up policies).*

---

## 4. Privilege Escalation (Abusing Sudo Privileges)

Operating inside the infrastructure as the restricted `mitch` user, we checked our current Sudo privileges:

```bash
sudo -l
```

### Vector Discovery
The command returned the following rule inside the system's sudoers configuration:

```text
(root) NOPASSWD: /usr/bin/vim
```

This misconfiguration allows the user `mitch` to execute the **Vim** text editor with full administrative privileges (`root`) **without providing a password**.

### Exploitation (GTFOBins Evasion)
Vim includes native functionality to execute system shell commands directly from its internal command interface. To abuse this vector, we launched Vim with `sudo`:

```bash
sudo /usr/bin/vim
```

Once inside Vim's interface, we pressed **`Shift + :`** to open the command prompt and typed the exclamation mark `!` (used to run external shell commands) followed by the Bash binary path:

```text
!/bin/bash
```

Upon pressing Enter, Vim spawned an interactive root shell. We verified our escalated privileges and captured the final flag:

```bash
whoami
# Output: root

cd /root
cat root.txt
```

* **Root Flag:** `THM{REDACTED_ROOT_FLAG}` *(Hidden in accordance with TryHackMe write-up policies).*

---

## 5. Conclusions and Mitigation

1. **Credential Weakness and Reuse:** Reusing a weak password (`secret`) across different system layers enabled direct remote access. Password policies must enforce strong complexity and length, and system accounts should never share passwords with web applications.
2. **Anonymous FTP Exposure:** Leaving open FTP shares with anonymous read access allows attackers to gather sensitive internal notes and usernames. Anonymous FTP login should be disabled in production environments.
3. **Excessive Sudoers Privileges:** Granting `NOPASSWD` sudo rights to interactive editors like `vim` or `nano` allows trivial privilege escalation to root. Users who need to edit specific files should be restricted via `sudoedit` rather than full binary execution.
