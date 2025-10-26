# 🕒 Chapter 2 — Scheduling Future Tasks
---

This chapter explains how to schedule **one-time**, **recurring**, and **system-based** tasks in Linux using `at`, `cron`, `anacron`, and systemd timers.

---

## ⚙️ 1. Scheduling One-Time Tasks with `at` Command
The **`atd` service** (daemon) allows scheduling **one-time jobs** in Linux using the `at` command.

### 🧩 Service Management
```bash
systemctl status atd
```
> ✅ Usually enabled and running by default.

---

### 🧾 Basic Commands
| Command | Description |
|----------|-------------|
| `atq` or `at -l` | View pending jobs for the current user (job ID, execution time, queue, owner) |
| `watch atq` or `watch at -l` | Continuously monitor pending jobs |
| `atrm <jobID>` or `at -r <jobID>` | Remove a pending job |

---

### 🧠 Creating One-Time Tasks

#### 1️⃣ Using the `at` Prompt
```bash
at noon
at> echo "hello" >> /tmp/hello.txt     # Creates file if it doesn’t exist
at> <Ctrl+D>                           # Exit at prompt

# Check and manage jobs:
atq                                    # Get jobID
atrm <jobID>                           # Remove job
```

---

#### 2️⃣ Without Entering the Prompt (Using a Pipeline)
```bash
echo "date >> /tmp/todayDate.txt" | at now
```
> Runs immediately without opening the `at` prompt.

---

#### 3️⃣ Scheduling a Script
```bash
at now < /tmp/script.sh
```
> Make sure `/tmp/script.sh` exists before running the command.
> 
---

### 📂 Job Storage & Access Control

#### 🔍 Stored Jobs
- A scheduled job = one small script file that stores both your environment setup and the commands you told at to run later.
- All scheduled jobs are saved as scripts in:
```
/var/spool/at
```

| Command | Description |
|----------|-------------|
| `cd /var/spool/at` | Navigate to job directory |
| `atq` | List pending jobs |
| `at -c <jobID>` | View job content |

---

#### 🔐 Access Control
| File | Description |
|------|-------------|
| `/etc/at.allow` | If it exists → only listed users can use `at`. |
| `/etc/at.deny` | Used if `/etc/at.allow` doesn’t exist → listed users **cannot** use `at`. |
| Neither exists | Only **root** can use `at`. |
> 🧠 **Notes:**
> - If `/etc/at.allow` exists but is **empty**, no one (except root) can use `at`.  
> - If `/etc/at.deny` exists but is **empty**, all users (except explicitly restricted ones) can use `at`.

---

## ⏰ 2. Scheduling Repeated Tasks with `cron`
The **`crond` service** schedules **recurring jobs** using `crontab`.

### 🧩 Service Management
```bash
systemctl status crond
```

---

### 🧾 Basic Commands
| Command | Description |
|----------|-------------|
| `crontab -l` | List current user’s cron jobs |
| `crontab -e` | Edit cron jobs |
| `crontab -r` | Remove current user’s crontab |
| `crontab -u alice -l` | (Root only) View another user’s crontab |
| `crontab mycronfile` | Replace crontab with file contents |

**Backup Example**
```bash
crontab -l > mycronfile
crontab -r
crontab mycronfile
```

---

### 📘 `/etc/crontab` Syntax
```
# ┌───────────── minute (0–59)
# │ ┌─────────── hour (0–23)
# │ │ ┌───────── day of month (1–31)
# │ │ │ ┌─────── month (1–12) OR jan,feb,mar,apr,...
# │ │ │ │ ┌───── day of week (0–6) (0 or 7 = Sunday)
# │ │ │ │ │
# *  *  *  *  *  user-name  command
```

---

### 📂 Cron File Locations 

#### 🗂️ **/var/spool/cron/**
- **Purpose:** Stores each user’s personal crontab file created using `crontab -e`.
- **Format:**
  ```
  <minute> <hour> <day> <month> <day_of_week> <command>
  ```
- Each user (including root) has their own file here:
  ```
  /var/spool/cron/<username>
  ```
- Example:
  ```
  /var/spool/cron/root
  /var/spool/cron/alice
  ```
- Any allowed user can manage their own crontab without affecting others.
  
---

#### 🗂️ **/etc/cron.d/**
- **Purpose:**  
  A **system-wide directory** for scheduling jobs.  
  Only **root** or administrators can create files here.
- This directory allows software packages and sysadmins to install their own cron jobs **without editing user crontabs**.
  
- **File Format (inside `/etc/cron.d/`):**
  ```
  <minute> <hour> <day> <month> <day_of_week> <user> <command>
  ```
  ⚡ **Note:** Unlike user crontabs, you **must include the user field**.
  
- **Example file: `/etc/cron.d/example`**
  ```
  0 2 * * * root /usr/local/bin/backup.sh
  ```
  → Runs `backup.sh` daily at 2 AM as the root user.
  
- **Default `0hourly` file:**
  - File: `/etc/cron.d/0hourly`
  - Purpose: Runs every hour at minute 1.
  - It connects to `/etc/cron.hourly/` through a script named `0anacron`.
  - This integration allows **Anacron** to manage daily/weekly/monthly jobs —  
    even if the system was off at the scheduled time.
    
---

#### 🗂️ **/etc/anacrontab**
- **Purpose:**  
  Configuration file for **Anacron**, which runs periodic jobs on systems that are **not always on** (like laptops or desktops).
  
- Unlike cron (which assumes the system is always running), **Anacron ensures missed jobs run after reboot**.

- **Format:**
  ```
  <period>  <delay>  <job-identifier>  <command>
  ```
  | Field | Meaning |
  |--------|----------|
  | `period` | How often the job should run (in days) — e.g. 1 → daily, 7 → weekly, 30 → monthly |
  | `delay` | How many minutes to wait after system startup before running |
  | `job-identifier` | A unique name used for logging |
  | `command` | The command or script to execute |
  
- **Example:**
  ```
  # period(days) delay(min) job-id        command
  1              5          cron.daily    nice run-parts /etc/cron.daily
  7              25         cron.weekly   nice run-parts /etc/cron.weekly
  @monthly       45         cron.monthly  nice run-parts /etc/cron.monthly
  ```
> 🧠 **Note:**  
> `nice run-parts` runs all executable scripts inside the specified directory in order.  
> This ensures all daily, weekly, and monthly maintenance tasks execute automatically after boot.

---

#### 🗂️ **/etc/cron.hourly/**
- Contains scripts that run **every hour**.  
- Example use cases:
  - Log rotation updates  
  - Cache cleanup tasks

---

#### 🗂️ **/etc/cron.daily/**
- Contains scripts that run **once per day**.  
- Example use cases:
  - Updating the `mlocate` database  
  - Removing temporary files

---

#### 🗂️ **/etc/cron.weekly/**
- Contains scripts that run **once per week**.  
- Example use cases:
  - System maintenance  
  - Cleaning unused files  
  - Generating weekly reports

---

#### 🗂️ **/etc/cron.monthly/**
- Contains scripts that run **once per month**.  
- Example use cases:
  - Monthly log rotation  
  - Monthly summaries or report generation

---

#### 🔐 **/etc/cron.allow** & **/etc/cron.deny**
- Control which users can use the `cron` service.
- Behavior (same as `at.allow` and `at.deny`):
  - If `/etc/cron.allow` exists → only listed users can use `cron`.
  - If `/etc/cron.deny` exists → listed users **cannot** use `cron`.
  - If neither file exists → only **root** can use `cron`.

---

✅ **Summary Table**

| Path | Description | Who Can Edit | Example Use |
|------|--------------|--------------|--------------|
| `/var/spool/cron/` | User-specific cron jobs | Each user | Personal scheduled tasks |
| `/etc/cron.d/` | System-wide cron files | Root/Admin | Package or global jobs |
| `/etc/anacrontab` | Runs missed jobs after reboot | Root | Ensures periodic jobs run after downtime |
| `/etc/cron.hourly/` | Hourly tasks | Root | Temporary cleanup, log rotation |
| `/etc/cron.daily/` | Daily tasks | Root | File cleanup, updates |
| `/etc/cron.weekly/` | Weekly tasks | Root | Maintenance, reports |
| `/etc/cron.monthly/` | Monthly tasks | Root | Monthly logs or summaries |
| `/etc/cron.allow` `/etc/cron.deny` | Access control | Root | Define who can use cron |


## 🧭 3. Systemd Timer Units
Modern Linux systems often use **systemd timers** instead of cron or anacron.
A `.timer` unit triggers a corresponding `.service` unit:
- **.service** → defines *what* to do  
- **.timer** → defines *when* to do it  

Check active timers:
```bash
systemctl list-units --type=timer
```

---

### 🧩 Examples
| Timer | Description |
|--------|-------------|
| `logrotate.timer` | Daily log rotation |
| `systemd-tmpfiles-clean.timer` | Daily cleanup of temporary files |

---

### ⚙️ Managing Temporary Files with `systemd-tmpfiles`

Systemd uses **tmpfiles** and timer units to manage temporary files and directories automatically.  
These components ensure system directories like `/tmp`, `/var/tmp`, and `/run` remain clean and correctly initialized.


| Command | Description |
|----------|-------------|
| `systemd-tmpfiles --create` | Apply tmpfiles rules manually |
| `systemd-tmpfiles --clean` | Clean up expired files |

---

#### 🗂️ Configuration File Locations
| Path | Purpose | Priority |
|------|----------|-----------|
| `/usr/lib/tmpfiles.d/` | **Package defaults** — provided by the distribution or installed packages. | Lowest |
| `/etc/tmpfiles.d/` | **Administrator overrides** — custom rules that replace defaults. | Highest |
| `/run/tmpfiles.d/` | Runtime configurations that disappear after reboot. | Temporary |

---

### 🔄 Related Units
1. **systemd-tmpfiles-setup.service**  
   -  cat /usr/lib/systemd/system/systemd-tmpfiles-setup.service` → Check configuration
   - Runs once at boot (`--create --remove --boot`) to prepare `/tmp`, `/var/tmp`, etc.

🧠 **So what happens at boot?**
1. The system starts mounting filesystems.  
2. `systemd-tmpfiles-setup.service` runs **once**.  
3. It ensures all volatile directories (`/run`, `/tmp`, etc.) exist, are properly permissioned, and cleaned.  
4. After the initial setup, **periodic cleanup** is handled by:
   - `systemd-tmpfiles-clean.timer` → schedules cleanup.  
   - `systemd-tmpfiles-clean.service` → performs cleanup.

2. **systemd-tmpfiles-clean.timer**
   - `systemctl cat systemd-tmpfiles-clean.timer` → Check configuration
   - Schedules **regular cleanup** of temporary files by triggering `systemd-tmpfiles-clean.service`.
   - Default Configuration
    `OnBootSec=15min` → start 15 min after boot  
    `OnUnitActiveSec=1d` → run daily

🧩 **Explanation:**
- The first cleanup occurs **15 minutes after boot** (to avoid slowing down startup).  
- Then it repeats **once every 24 hours**.  
```
Boot system → 15 minutes later → cleanup runs once
Then → repeats every 24 hours
```

3. **systemd-tmpfiles-clean.service**  
   - `cat /usr/lib/systemd/system/systemd-tmpfiles-clean.service` → Check configuration
   - Performs the **actual cleanup** of temporary files.
   - Default Configuration
    `Type=oneshot` → runs a single cleanup task and exits.    
    `ExecStart=systemd-tmpfiles --clean` runs the tmpfiles tool with the `--clean` flag to remove expired files according to system rules.

---

#### 🗂️ Example Tmpfiles Config
File: `/usr/lib/tmpfiles.d/tmp.conf`
That’s the one which defines the cleanup policy for /tmp and /var/tmp.
```
# Clear tmp directories separately
q /tmp     1777 root root 10d
q /var/tmp 1777 root root 30d
```
- `q` → clean directory  
- `1777` → permissions (sticky world-writable)  
- `10d / 30d` → delete files older than these ages

---

### 🧩 Customizing Timer Settings
To modify timer intervals:
```bash
cp /usr/lib/systemd/system/systemd-tmpfiles-clean.timer /etc/systemd/system/
vim /etc/systemd/system/systemd-tmpfiles-clean.timer
systemctl daemon-reload
```
Systemd prefers the copy in `/etc/systemd/system/`.
Check Updates:
```bash
systemctl cat systemd-tmpfiles-clean.timer
```

---

#### ⚙️ Overriding Default Rules
```bash
cp /usr/lib/tmpfiles.d/tmp.conf /etc/tmpfiles.d/tmp.conf
vim /etc/tmpfiles.d/tmp.conf     #  d /tmp1 1777 root root 10d   
```

#### 🧠 What Happens After Adding the Rule
1. On the next boot, systemd-tmpfiles-setup.service will:
- Create /tmp1 automatically.
- Apply permissions and ownership defined in your rule.

2. Then, systemd-tmpfiles-clean.timer will:
- Schedule periodic cleanup.
- Trigger systemd-tmpfiles-clean.service to remove expired files.

If you don’t want to reboot, you can create the directory manually using:
```bash
systemd-tmpfiles --create
```

#### 🧪 Testing Your Configuration
```bash
ls -ld /tmp1            # Ensure /tmp1 exists
touch /tmp1/test        # Create a test file
sleep 30s               # Wait 30 seconds
ls /tmp1                # File still exists
```

🧠 Why is it still there?
> - 30s means “delete files not accessed for 30 seconds”.
> - But cleanup only happens when systemd-tmpfiles --clean runs.
> - By default, the timer runs once per day (OnUnitActiveSec=1d).
> - To test cleanup immediately, run:
```bash
systemd-tmpfiles --clean /etc/tmpfiles.d/tmp.conf
```
---

## 📊 Sysstat & Systemd Timers

**Sysstat** tools collect performance statistics using a systemd timer.

```bash
yum install sysstat
systemctl status sysstat
systemctl list-units --type=timer
```

Default timer: `sysstat-collect.timer` → runs every **10 minutes**.
sysstat-collect.timer This timer triggers:  sysstat-collect.service

```bash 
systemctl cat sysstat-collect.timer
```
Timer unit file is located at: /usr/lib/systemd/system/sysstat-collect.timer

Change interval:
```bash
cp /usr/lib/systemd/system/sysstat-collect.timer /etc/systemd/system/
vim /etc/systemd/system/sysstat-collect.timer   # Change *:00/10 → *:00/01
systemctl daemon-reload
```

View collected data:
```bash
sar -r
```

---

## ✅ Summary

| Tool | Purpose | Typical Use |
|-------|----------|-------------|
| `at` | One-time job scheduling | Run once at a specific time |
| `cron` | Repeated job scheduling | Hourly, daily, weekly jobs |
| `anacron` | Run missed cron jobs after reboot | For non-24/7 systems |
| `systemd timers` | Modern system-level scheduling | Replaces cron/anacron with better integration |
