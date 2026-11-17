# Room 404 - TryHackMe Write-up

A detailed write-up on how to solve the **Room 404** machine on TryHackMe. This lab focuses on web enumeration, identifying exposed version control repositories (`.git`), and reconstructing source code to recover sensitive information.

* **Author:** HackSysX70
* **Platform:** TryHackMe
* **Difficulty:** Easy
* **OS:** Linux (Ubuntu)

---

## 1. Reconnaissance (Recon)

We begin by performing a full port scan on the target machine using **Nmap** to identify open ports, running services, and potential entry vectors.

```bash
nmap -p- -sS -T4 -Pn -n -vvv -sVC --open -oN scan.txt <TARGET_IP>
```

### Explanation of Parameters Used

| Parameter | Description |
| :--- | :--- |
| `-p-` | Scans **all 65,535 TCP ports** to ensure no services running on non-standard ports are missed. |
| `-sS` | Performs a *SYN Stealth* scan (half-open), which is fast and does not complete the full TCP 3-way handshake. |
| `-T4` | Sets the timing template to level 4 (aggressive) to speed up the scan in stable lab environments. |
| `-Pn` | Treats the host as online, **skipping ICMP ping discovery** in case a firewall blocks ping probes. |
| `-n` | Disables DNS name resolution, significantly reducing overall scan time. |
| `-vvv` | Enables maximum verbosity to display discovered ports in real-time during the scan. |
| `-sVC` | Combines `-sV` (service version detection) and `-sC` (default NSE scripts for basic vulnerability checks). |
| `--open` | Filters the output to display only ports that are actively open. |
| `-oN scan.txt` | Saves the scan output in normal format to `scan.txt` for documentation and future reference. |

### Scan Results Summary

| Port | State | Service | Version / Details |
| :--- | :--- | :--- | :--- |
| **22/TCP** | Open | `ssh` | OpenSSH 9.6p1 Ubuntu |
| **8080/TCP** | Open | `http` | Werkzeug httpd 3.0.1 (Python 3.12.3) — *Byte Lotus — Stay Noticed* |

During the script scan (`-sC`), Nmap's **`http-git`** module automatically detected an exposed Git repository on the web server:

```text
| http-git: 
|   <TARGET_IP>:8080/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: initial Byte Lotus guest platform
```

---

## 2. Web Enumeration

Navigating to `http://<TARGET_IP>:8080` in a web browser reveals a minimal interface without obvious interactive features or visible links. To confirm Nmap's findings and rule out any additional hidden paths on the web server, we performed directory brute-forcing using **Gobuster**.

```bash
sudo gobuster dir -u http://<TARGET_IP>:8080/ -w /usr/share/wordlists/dirb/common.txt -t 70 -x php,html,txt,bak,css
```

### Explanation of Parameters Used

| Parameter | Description |
| :--- | :--- |
| `dir` | Instructs Gobuster to use directory and file enumeration mode. |
| `-u http://<TARGET_IP>:8080/` | Specifies the target URL including port `8080`. |
| `-w /usr/share/wordlists/...` | Sets the path to the wordlist containing common directory and file names. |
| `-t 70` | Runs the execution using 70 concurrent threads to optimize scanning speed. |
| `-x php,html,txt,bak,css` | Instructs Gobuster to append these specific extensions to every word in the list, searching for web files or backups. |

### Key Discoveries
The scan completed 100% of the wordlist and confirmed the full exposure of the internal Git repository structure:

| Discovered Path | HTTP Status | Size | Description |
| :--- | :--- | :--- | :--- |
| **`.git/HEAD`** | `200` | `21` | Confirms the active branch reference (`ref: refs/heads/main`). |
| **`/.git/config`** | `200` | `92` | Local Git repository configuration file. |
| **`/.git/hooks`** | `200` | `897` | Directory containing automated Git hook scripts. |
| **`/.git/index`** | `200` | `289` | Binary staging file tracking the state of repository files. |
| **`/.git/info`** | `200` | `145` | Folder containing additional repository metadata. |
| **`/.git/logs`** | `200` | `165` | History of reference and commit updates (`HEAD`). |
| **`/.git/objects`** | `200` | `283` | Internal compressed object database storing project files. |

Navigating directly to `http://<TARGET_IP>:8080/.git/` also confirmed that directory listing (*Index of /.git/*) was enabled on the server.

---

## 3. Exploitation (Source Code Reconstruction)

### Vulnerability Analysis: Exposed `.git` Directory
When a `.git/` folder is left publicly accessible on a web server, an attacker can download the internal Git object database and reconstruct the entire source code of the application—even if the original files are not directly served or listed in the public web root.

### Dumping and Restoring the Repository
To automate the extraction and reconstruction of the repository, we used **`git-dumper`**. This tool fetches all accessible Git objects and checks out the working tree locally:

```bash
git-dumper http://<TARGET_IP>:8080/.git ./room404_repo
```

### Explanation of Arguments Used

| Argument | Description |
| :--- | :--- |
| `http://<TARGET_IP>:8080/.git` | Specifies the target URL where the exposed Git repository is hosted. |
| `./room404_repo` | Defines the local destination directory where the reconstructed project files will be saved. |

### Retrieving the Flag
Once `git-dumper` finished reconstructing the repository, we navigated into the local directory and listed the recovered files:

```bash
cd ./room404_repo
ls -la
```

The working tree revealed three restored project files:
* `index.html`
* `app.js`
* `README.md`

Inspecting the contents of **`README.md`** revealed the flag for the machine:

```bash
cat README.md
```

* **Flag:** `THM{REDACTED_FLAG}` *(Hidden in accordance with TryHackMe write-up policies).*

---

## 4. Conclusions and Mitigation

1. **Restrict Access to Hidden Files and Directories:** Web servers (such as Nginx, Apache, or Python/Werkzeug reverse proxies) should be explicitly configured to deny HTTP requests to any path starting with a dot (e.g., `/.git`, `/.env`).
2. **Disable Directory Listing:** Turn off automatic directory indexing (*Index of*) in production environments so attackers cannot browse server folder structures.
3. **Clean Production Deployments:** Avoid deploying `.git` folders to production servers. Instead, use CI/CD pipelines or clean build artifacts (`git archive`) to push only the necessary runtime files.
