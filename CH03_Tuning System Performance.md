# Chapter 3: Direct Ways to Improve System Performance

This chapter explains how to `enhance Linux system performance` using `tuned`, a powerful optimization daemon that applies `dynamic and static performance profiles`.\
We also explore how to manage tuned profiles, monitor active optimizations, and integrate tuning with tools like `Cockpit` for easy web-based management.

## 🔹 System Tuning with the tuned Daemon

System performance tuning in Linux focuses on adjusting kernel
parameters, I/O behavior, CPU scheduling, and power management to
achieve optimal performance, stability, and efficiency for different
workloads.

Red Hat--based systems provide a powerful framework for this: **tuned**.

### 🔧 What Is tuned?

`tuned` is a system daemon that automatically applies performance
optimizations based on predefined profiles.
-   A service in Linux that helps optimize system performance
-   Dynamically and statically adjusts system settings
-   Provides optimized profiles for different workload types
-   Selects the most suitable profile after startup
-   Continuously monitors system activity (in dynamic mode)

The **tuned daemon** manages tuning operations through: 
- Predefined profiles: optimized for specific workloads
- Plugins: that adjust kernel and system parameters
- Real-time monitoring: (in dynamic mode) to modify settings based on current usage



## ⚙️ Categories of System Tuning

### 1️⃣ Static Tuning
-   The system administrator selects a profile manually
-   The settings are applied once and stay fixed until the profile is changed
-   No real-time analysis
-   Best for stable workloads (e.g., production servers with predictable workloads)

### 2️⃣ Dynamic Tuning (Disabled by default)
-   Monitors system metrics such as CPU, memory, disk I/O
-   Automatically adjusts system parameters based on the current workload
-   Great for unpredictable workloads
-   Must be manually enabled


## 🔧 Installation & Enabling tuned
Tuned is usually available by default on Red Hat–based systems but may not be enabled.
``` bash
yum install tuned -y
systemctl enable --now tuned
```
Once active, tuned loads the best recommended profile for your hardware.

## 📂 Tuned Profiles

### 1️⃣ Power-Saving Profiles
Focus: reduce energy consumption\
Use cases: 
-   laptops
-   low-power servers
-   Environments prioritizing efficiency over performance

### 2️⃣ Performance Profiles
Focus: maximize system performance for specific workloads\
Use cases: 
-   Low-latency workloads: Trading systems, Telecom apps, Real-time processing
-   High-throughput workloads: Database servers, File servers, Storage-heavy applications
-   Virtualization performance: Virtual machines, Hypervisors (e.g., KVM hosts)



## 📂 Common tuned Profiles
| Profile                    | Purpose                     | Typical Use Case                                 |
| -------------------------- | --------------------------- | ------------------------------------------------ |
| **balanced** *(default)*   | General-purpose tuning      | Normal workloads                                 |
| **throughput-performance** | Maximizes system throughput | Databases, file servers, compute nodes           |
| **latency-performance**    | Minimizes latency           | Trading apps, telco workloads, real-time systems |
| **powersave**              | Reduces energy consumption  | Laptops, power-sensitive systems                 |
| **virtual-host**           | Optimize hypervisors        | KVM hosts                                        |
***---***

## 🛠️ Managing tuned with `tuned-adm`
```bash
tuned-adm list              # Show all available profiles and highlight the active one
tuned-adm active            # Display the current active profile
tuned-adm recommend         # Suggest the optimal profile for the system hardware
tuned-adm off               # Disable tuning and revert to default system settings
tuned-adm profile <name>    # Switch to a specific profile
```
Install additional profiles:
```bash
dnf search profiles
dnf install <profile>
tuned-adm list
```
------------------------------------------------------------------------

## ⚡ Advanced: Merging Profiles

`tuned` supports `profile layering`, allowing you to combine multiple profiles into a single, merged configuration.\
This lets your system inherit and merge settings from more than one profile.

#### **Key Points:**

-   Many built-in profiles (e.g., throughput-performance, latency-performance) already inherit from balanced.
-   You can activate multiple profiles together using tuned-adm, and tuned will merge them automatically.
-   Merge follows left-to-right order: later profiles override conflicting settings.

#### **How merging works:**
```bash
    tuned-adm profile balanced throughput-performance
```
What happens:
-   tuned creates one layered profile **internally**.
-   Settings from balanced are applied first.
-   Settings from throughput-performance are applied next, overriding any conflicts.
-   The system ends up with one unified profile optimized for throughput while retaining general settings from balanced.

✅ Example scenario:
-   balanced sets CPU governor to ondemand.
-   throughput-performance sets CPU governor to performance.
-   After merging, the active governor is performance (from the last profile in the command).
-   This allows flexible, customized tuning without manually editing profile files.


## 📌 Important Tuned Files

### `/etc/tuned/active_profile`
A simple text file that contains only the name of the currently active tuned profile.

### `/etc/tuned/tuned-main.conf`

-   Main **tuned** configuration file.
-   Controls global tuned settings such as dynamic tuning
    **<dynamic_tuning=0>** by default.

### `/usr/lib/tuned/`
Contains **all predefined profiles** and each contains **tuned.conf**: 
``` ini
/usr/lib/tuned/balanced/ 
/usr/lib/tuned/throughput-performance/ 
/usr/lib/tuned/latency-performance/ 
/usr/lib/tuned/virtual-guest/ 
/usr/lib/tuned/virtual-host/ 
/usr/lib/tuned/desktop/  
...
```
#### 🔹 tuned.conf — Profile Configuration 
This file defines all tuning parameters applied by that profile, such as:
-   CPU governor settings
-   Disk I/O scheduler
-   Kernel sysctl parameters
-   Virtual memory tuning
-   Network optimizations
Example:
``` ini
# cat /usr/lib/tuned/desktop/tuned.conf
[main]
summary=Optimize system for desktop usage   # Displayed in tuned-adm list
include=balanced                            # Desktop profile inherits all settings from the balanced profile     
# [tuning-parameters...]
```

------------------------------------------------------------------------

## 📌 Creating a Custom Tuned Profile

Custom profiles should always be placed under: `/etc/tuned/` \
This directory overrides anything in `/usr/lib/tuned/` and is preserved across updates.

### 1️⃣ Create profile directory

``` bash
cd /etc/tuned
mkdir web_server
cd web_server
```

### 2️⃣ Create tuned.conf
    vim tuned.conf
Add:
``` ini
[main]
summary=Optimize system for web_server
include=balanced
# [script]
# script=script_name.sh       # Optional: attach a script
```

### 3️⃣ Optional Script
``` bash
touch script_name.sh
chmod +x script_name.sh
```
Script content:
``` bash
#!/bin/bash
echo "Optimize system for web_server" >> /var/log/script_name.log
# Tuned will run this script when the profile is activated.
```

### 4️⃣ Check profile
``` bash
# Verify profile
tuned-adm list

# Activate
tuned-adm profile web_server

# Check logs
cat /var/log/script_name.log
```

### Turn tuned OFF if needed

    tuned-adm off
    systemctl disable --now tuned

## 📌 Managing tuned Using Cockpit
Cockpit is a web-based administrative interface used to manage Linux servers and virtual machines. \
It also provides a simple way to manage tuned performance profiles.

### 1️⃣ Start Cockpit
    systemctl status cockpit
    systemctl start cockpit

### 2️⃣ Get IP
    ifconfig

### 3️⃣ Access Cockpit UI

Open in browser: https://SERVER-IP:9090 \
Login: `root / password`

### 4️⃣ Manage Performance Profiles

In **Overview → Configuration**: 
-   If the tuned service is off, it will show: `Performance Profile: none`
-   Select any performance profile (e.g., balanced, throughput-performance, latency-performance, virtual-host, etc.)
-   This gives you a graphical alternative to:
```bash
    tuned-adm profile <profile-name>
```


## ✅ Summary
-   `tuned` optimizes system performance using predefined profiles
-   Supports `static` and `dynamic` tuning
-   Built-in profiles: **balanced**, **throughput-performance**, **latency-performance**, **powersave**, **virtual-host**
-   `tuned-adm` manages profiles and supports merging multiple profiles
-   Custom profiles can include scripts and reside under `/etc/tuned/`
-   `Cockpit` allows web-based tuning management
-   Profile **merging** applies settings dynamically without creating new files