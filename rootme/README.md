# RootMe - TryHackMe Write-up

A detailed write-up on how to solve the **RootMe** machine on TryHackMe. This lab covers advanced web reconnaissance techniques, bypassing upload filters using extension manipulation (Arbitrary File Upload), and local privilege escalation by abusing misconfigured SUID binaries.

* **Author:** HackSysX70
* **Platform:** TryHackMe
* **Difficulty:** Easy
* **OS:** Linux

---

## 1. Reconnaissance (Recon)

We begin by performing a full port scan on the target machine using **Nmap** to identify exposed services.

```bash
nmap -p- -sS -T4 -Pn -n -vvv -sVC <TARGET_IP> -oN scan.txt
```

### Explanation of parameters used:
* `-p-`: Scans **all 65,535 available ports**. This is used because administrators often hide services on non-standard ports. If we only scan the default top 1,000 ports, we might miss critical entry vectors.
* `-sS`: Performs a *SYN Stealth* scan (half-open). It is fast, efficient, and does not complete the 3-way handshake, making it more stealthy against simple logging.
* `-T4`: Sets the timing template to level 4 (aggressive). This speeds up the scan to save time without losing too much reliability on stable networks like lab environments.
* `-Pn`: Treats the host as active, **skipping ICMP ping discovery**. This prevents Nmap from dropping the target if it blocks ping requests via a firewall.
* `-n`: Disables DNS name resolution, significantly reducing the total scan time.
* `-vvv`: Increases the verbosity level to the maximum. It displays open ports on the screen in real-time as they are discovered, rather than waiting for the entire process to finish.
* `-sVC`: Combines `-sV` (determines the exact service version running on the port) and `-sC` (runs Nmap default scripts to look for common vulnerabilities in those services).
* `-oN scan.txt`: Saves the complete output in normal format into a file named `scan.txt` for future reference.

### Scan Results:
* **Port 22/TCP:** Open - OpenSSH Server.
* **Port 80/TCP:** Open - Apache Web Server (HTTP).

---

## 2. Web Enumeration

Since port 80 is hosting a web server, we proceed to perform directory brute-forcing using **Gobuster** to uncover hidden structures or administration pages.

```bash
sudo gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt -t 70 -x php,html,txt,bak,css
```

### Explanation of parameters used:
* `dir`: Instructs Gobuster to use directory and file enumeration mode.
* `-u http://<TARGET_IP>`: Specifies the target URL.
* `-w /usr/share/wordlists/...`: Sets the path to the wordlist containing common directory names.
* `-t 70`: Runs the execution using 70 concurrent threads to optimize scanning speed.
* `-x php,html,txt,bak,css`: Instructs Gobuster to append these specific extensions to every word in the list, searching for actual backend script files or backups.

### Key Discoveries:
* `/panel/` `(Status: 301)` -> A hidden directory hosting a file upload form.
* `/uploads/` `(Status: 301)` -> A public folder where all files submitted to the panel are stored.

---

## 3. Exploitation (Gaining Initial Access)

Upon inspecting the `/panel/` directory, we locate an upload feature. Testing shows that the web application implements a blacklist filtering standard `.php` extensions to prevent the execution of malicious scripts.

### Vulnerability Analysis & Extension Bypass
To evade this restriction, we can exploit standard Apache configurations. Web servers often support multiple extensions for PHP processing (such as `.php3`, `.php4`, `.php5`, or `.phtml`). If the application's blacklist only checks for `.php`, alternative extensions will bypass the filter but will still be processed as code by the backend engine.

### Preparation and Execution
We prepared a standard PHP reverse shell payload and renamed the file to append an allowed extension: `revshell.php5`. 

We set up a Netcat listener on our attacker machine (Kali Linux) using port `4444`:

```bash
nc -lvnp 4444
```

Next, we successfully uploaded `revshell.php5` through the `/panel/` page. To trigger execution, we forced the server to load the script by requesting its direct path inside the storage directory:

```bash
curl http://<TARGET_IP>/uploads/revshell.php5
```

The script successfully connected back to our listener, granting an initial shell session under the context of the web server service account:
```bash
whoami
# Output: www-data
```

With initial access established, we navigated to the user's home space to retrieve the first flag:
```bash
cat /var/www/user.txt
```
* **User Flag:** `THM{REDACTED_USER_FLAG}` (Hidden due to platform write-up policies).

---

## 4. Privilege Escalation

Operating inside the system as the restricted `www-data` user, we initiated a local enumeration process to find a vertical escalation vector to become `root`.

We searched for binaries that have the **SUID** (*Set User ID*) bit active. This special permission allows a regular user to run an executable file with the privileges of the file owner (which is usually root).

```bash
find / -perm -4000 2>/dev/null
```

### Explanation of parameters used:
* `/`: Starts the search from the root of the file system.
* `-perm -4000`: Filters the search to show only files with SUID execution permissions.
* `2>/dev/null`: Redirects standard error messages (like "Permission Denied") to the null device, cleaning the terminal output.

### Vector Discovery
Among the standard binaries, we spotted an unusual entry:
```text
/usr/bin/python2.7
```
Leaving a programming interpreter like Python with SUID permissions is a fatal misconfiguration. It allows an attacker to run custom Python code that inherits full administrative capabilities.

### Gaining Root Access
Referencing GTFOBins evasion methods, we used Python's native operating system modules to spawn a system terminal (`/bin/sh`). We included the critical **`-p`** (*privileged mode*) flag to force the shell to preserve the inherited root context instead of dropping it:

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

Upon executing the command, the terminal instantly elevated, giving us absolute control over the infrastructure:

```bash
whoami
# Output: root
```

We proceeded to change directories to the administrator's restricted home space to capture the final proof token:

```bash
cd /root
cat root.txt
```
* **Root Flag:** `THM{REDACTED_ROOT_FLAG}` (Hidden due to platform write-up policies).

---

## Conclusions and Mitigation
1. **Insecure File Upload Controls:** Implementing blacklists to filter dangerous extensions is an unsafe approach. Attackers will always find alternative extensions (`.php5`, `.phtml`, etc.) or execution tricks. The solution is to use **whitelisting**, permitting only strictly necessary non-executable formats (like `.png`, `.jpg`, `.pdf`) and storing files outside the web root.
2. **Misconfigured SUID Permissions:** Powerful system tools, compilers, or language interpreters (like Python, Bash, Perl or Find) should never have the SUID bit enabled. Regular security audits should be performed using `find` queries to ensure low-privileged users cannot run administrative binaries.
3. **Web Infrastructure Isolation:** Service accounts like `www-data` should be restricted to the minimum required environment permissions, avoiding access to unnecessary tools inside `/usr/bin/` to prevent post-exploitation escalation.
