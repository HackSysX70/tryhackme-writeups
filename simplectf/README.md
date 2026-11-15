# Simple CTF - TryHackMe Write-up

A detailed write-up on how to solve the **Simple CTF** machine on TryHackMe. This lab covers network reconnaissance, enumerating an anonymous FTP service, targeted brute-force attacks against SSH on a non-standard port, and local privilege escalation by abusing permitted Sudo privileges (Vim).

* **Author:** HackSysX70
* **Platform:** TryHackMe
* **Difficulty:** Easy
* **OS:** Linux

---

## 1. Reconnaissance (Recon)

We begin by performing a full port scan on the target machine using **Nmap** to identify all exposed services.

```bash
nmap -p- -sS -T4 -Pn -n -vvv -sVC <TARGET_IP> -oN escaneo.txt
```

### Explanation of parameters used:
* `-p-`: Scans **all 65,535 available ports**. This is used because administrators often hide services on non-standard ports. If we only scan the default top 1,000 ports, we might miss critical entry vectors.
* `-sS`: Performs a *SYN Stealth* scan (half-open). It is fast, efficient, and does not complete the 3-way handshake, making it more stealthy against simple logging.
* `-T4`: Sets the timing template to level 4 (aggressive). This speeds up the scan to save time without losing too much reliability on stable networks like lab environments.
* `-Pn`: Treats the host as active, **skipping ICMP ping discovery**. This prevents Nmap from dropping the target if it blocks ping requests via a firewall.
* `-n`: Disables DNS name resolution, significantly reducing the total scan time.
* `-vvv`: Increases the verbosity level to the maximum. It displays open ports on the screen in real-time as they are discovered, rather than waiting for the entire process to finish.
* `-sVC`: Combines `-sV` (determines the exact service version running on the port) and `-sC` (runs Nmap default scripts to look for common vulnerabilities in those services).
* `-oN escaneo.txt`: Saves the complete output in normal format into a file named `escaneo.txt` for future reference.

### Scan Results:
* **Port 21/TCP:** Open - FTP Service.
* **Port 80/TCP:** Open - HTTP Service (Apache Web Server).
* **Port 2222/TCP:** Open - SSH Service (Relocated non-standard port).

---

## 2. Enumeration & Discovery (Anonymous FTP)

Upon detecting port 21 open, we accessed the FTP service using the default `anonymous` account. Initially, data transmission failed due to network constraints within the TryHackMe environment, forcing us to execute the **`passive`** command to enable passive transfer mode.

Once inside, we located a directory named `pub/` containing a text file named `ForMitch.txt`. We downloaded it to our local machine, revealing the following developer note:

```text
Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, 
and the password is so weak... i cracked it in seconds. Gosh... what a mess!
```

### Critical Analysis:
The leaked note exposed critical operational security (OPSEC) flaws:
1. A valid system user named **`mitch`** exists.
2. The user reused their web platform password for the operating system account (credential reuse).
3. The password is highly weak and vulnerable to common wordlist attacks.

---

## 3. Exploitation (SSH Brute-Force & Initial Access)

Armed with a confirmed username (`mitch`), knowing that SSH was running on the alternative port **2222**, and knowing the password was weak, we launched a targeted brute-force attack using **Hydra** paired with the standard `rockyou.txt` wordlist.

```bash
sudo hydra -l mitch -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>:2222 -t 16 -f -I
```

### Explanation of parameters used:
* `-l mitch`: Specifies the single target username strictly.
* `-P /.../rockyou.txt`: Loads the path to the standard Kali Linux password wordlist.
* `ssh://<TARGET_IP>:2222`: Directs the attack specifically to the SSH protocol on the alternative port **2222**.
* `-t 16`: Runs 16 parallel tasks to accelerate the attack without dropping connections.
* `-f`: Forward mode. Instructs Hydra to stop immediately once the first valid credential set is found.
* `-I`: Ignores previous session restore attempts, forcing a clean attack from scratch.

### Brute-Force Results:
Hydra successfully retrieved the valid credential set:
* **Username:** `mitch`
* **Password:** `secret`

We established a remote terminal session via SSH, explicitly defining the non-standard port using the `-p` parameter:
```bash
ssh mitch@<TARGET_IP> -p 2222
```

### Terminal Stabilization (TTY Spawn)
Upon logging in, the shell provided by the server was highly limited and non-interactive. To stabilize our session and spawn a fully functional Bash TTY, we executed:
```bash
script /dev/null -c bash
```

We then navigated to mitch's home directory and read the first proof token:
```bash
cat user.txt
```
* **User Flag:** `THM{REDACTED_USER_FLAG}` (Hidden due to platform write-up policies).

---

## 4. Privilege Escalation (Abusing Sudo Privileges)

Operating inside the infrastructure as the restricted `mitch` user, we performed local enumeration to check our current Sudo privileges by executing:

```bash
sudo -l
```

### Vector Discovery:
The command returned the following allowed rule within the system's sudoers configuration:
```text
(root) NOPASSWD: /usr/bin/vim
```
This misconfiguration explicitly implies that the user `mitch` can execute the **Vim** text editor with the highest administrative privileges (`root`) **without providing any password**.

### Exploitation (GTFOBins Evasion)
Vim includes native features to execute system commands directly from its internal text-editing interface. To abuse this vector, we initiated Vim utilizing sudo:

```bash
sudo /usr/bin/vim
```

Once inside Vim's visual interface, we pressed **`Shift + :`** to enter the editor's command-line interface, and typed the exclamation mark `!` (used in Vim to execute external shell commands) followed by the system shell path:

```text
!/bin/bash
```

Upon hitting Enter, Vim spawned a system terminal. Since Vim was initiated as root, the newly spawned shell automatically inherited the supreme administrative context.

We verified our escalated identity on the host:
```bash
whoami
# Output: root
```

Finally, we shifted directories to the administrator's restricted home space to capture the final proof token of the laboratory:

```bash
cd /root
cat root.txt
```
* **Root Flag:** `THM{REDACTED_ROOT_FLAG}` (Hidden due to platform write-up policies).

---

## Conclusions and Mitigation
1. **Credential Weakness and Reuse:** The initial critical point of failure was the reuse of an extremely weak password (`secret`) across multiple system layers. Password policies must enforce complexity and length, and administrative credentials should never match standard application accounts.
2. **Anonymous Service Exposure:** Leaving open FTP shares with anonymous read permissions allows attackers to extract critical development notes or infrastructure details. Anonymous FTP access should be entirely disabled in production environments.
3. **Excessive Sudoers Privileges:** Granting unauthenticated Sudo access (`NOPASSWD`) to powerful interactive binaries like Vim or Nano is equivalent to revealing the root password. Since Vim natively allows shell execution, users needing to edit specific files should be restricted via `sudoedit` instead of granting full execution rights over the entire editor binary.
