+++
date = '2025-11-21T17:20:29+03:00'
draft = false
title = 'Today I Learned: Setting up system updates with systemd timer'
+++

*How I automated Linux system updates with comprehensive logging and learned why systemd timers beat cron*


**The Wake-Up Call**

Picture this: You're managing a handful of Linux servers (or even just one VPS), and you know you should be keeping them updated. Security patches are released constantly, but manually SSH-ing into each server, running `apt update && apt upgrade`, waiting for it to complete, and checking for issues is... tedious.

So updates get postponed. "I'll do it tomorrow," you tell yourself. Then a week passes. Then a month. Then you read about a critical vulnerability that was patched three weeks ago, and panic sets in.
There had to be a better way.

**Why Automate System Updates?**

Before diving into the solution, let's talk about why this matters—especially for system Admins like me:

* **Security First:** Unpatched systems are low-hanging fruit for attackers. In blue team operations, keeping systems updated is defense 101.

* **Consistency:** Automated updates happen on schedule, every time, without fail. No human forgetfulness involved.

* **Audit Trail:** With proper logging, you have a complete record of what was updated and when—crucial for incident response and compliance.

* **Time Savings:** Set it up once, benefit forever. Your time is better spent learning new skills or handling more complex issues.

* **Learning Opportunity:** Building this taught me about systemd, bash scripting, logging practices, and service management—all fundamental sysadmin skills.
**
The Traditional Approach: Cron**

Most guides suggest using cron for scheduling tasks:

```
# Run updates daily at 3 AM
0 3 * * * /usr/bin/apt update && /usr/bin/apt upgrade -y
```

This works, but it has limitations:

* ❌ Limited logging (redirecting to files is clunky)
* ❌ No dependency management
* ❌ Harder to debug when things go wrong
* ❌ No integration with system logging
* ❌ Runs even if system was off at scheduled time


Enter systemd timers.

***Why Systemd Timers Are Better***

Systemd timers are the modern replacement for cron, and they offer significant advantages:

* Native logging with `journalctl`
* Dependency management (run after network is available)
* Persistent timers (catches up if system was off)
* Easy monitoring with `systemctl status`
* Better error handling and failure notifications
* Security hardening options built-in


My Solution: Three-Part System

I built a complete automated update system consisting of:

1. `Bash script` - Does the actual updating with comprehensive logging
1. `Systemd service` - Defines how the script runs
2. `Systemd timer` - Schedules when it runs

Let me walk you through each component.

**Part 1: The Update Script**

I created `/usr/local/bin/system-update.sh` with these key features:

**Feature 1: Structured Logging**

Every action is logged with timestamps:

```
LOG_DIR="/var/log/system-updates"
LOG_FILE="${LOG_DIR}/update-$(date +%Y-%m-%d_%H-%M-%S).log"

log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}
```

This creates a new log file for each run (e.g., `update-2024-11-14_03-00-00.log`) while also maintaining a `latest.log` symlink for quick access.

Why this matters: When troubleshooting issues or conducting security audits, I need to know exactly what happened and when. These logs provide a complete audit trail.

Feature 2: Comprehensive Update Process

The script performs a complete update cycle:

```
# 1. Update package lists
apt-get update

# 2. Upgrade packages (non-interactive)
apt-get upgrade -y

# 3. Full upgrade (handles dependencies)
apt-get full-upgrade -y

# 4. Remove unnecessary packages
apt-get autoremove -y

# 5. Clean up cache
apt-get autoclean
```

Each step is wrapped in error handling:

```
run_command() {
    log_message "Running: $1"
    eval "$1" >> "$LOG_FILE" 2>&1
    local exit_code=$?
    if [ $exit_code -eq 0 ]; then
        log_message "✓ Command completed successfully"
    else
        log_message "✗ Command failed with exit code: $exit_code"
    fi
    return $exit_code
}
```

**Lesson learned**: Non-interactive updates are crucial for automation. The `-y` flag auto-confirms prompts, and proper error handling ensures failures are logged and visible.

**Feature 3: Reboot Detection**

One critical aspect of system updates: some patches require a reboot. My script detects this:

```
if [ -f /var/run/reboot-required ]; then
    log_message "⚠ REBOOT REQUIRED!"
    log_message "Packages requiring reboot:"
    cat /var/run/reboot-required.pkgs >> "$LOG_FILE" 2>&1
else
    log_message "✓ No reboot required"
fi
```

**Why this matters:** In production environments, you need to know when manual intervention is required. I don't auto-reboot because downtime needs to be planned, but I get clear notification that action is needed.

**Feature 4: Automatic Log Cleanup**

Logs shouldn't pile up forever:

```
RETENTION_DAYS=30

# Clean up old logs
find "$LOG_DIR" -name "update-*.log" -type f -mtime +${RETENTION_DAYS} -delete
```

Keeps 30 days of history—enough for troubleshooting and audits, not so much that disk space becomes an issue.

**Feature 5: Disk Space Reporting**

The script includes current disk usage in each log:

```
log_message "Current disk usage:"
df -h / | tee -a "$LOG_FILE"
```

This helps spot trends—if disk usage is climbing, I know to investigate before it becomes critical.

**Part 2: The Systemd Service**
	
The service file (`/etc/systemd/system/system-update.service`) defines how the script runs:

```
[Unit]
Description=Automated System Update and Upgrade
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/system-update.sh
StandardOutput=journal
StandardError=journal

# Security hardening
PrivateTmp=yes
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/log/system-updates /var/lib/apt /var/cache/apt /var/lib/dpkg

[Install]
WantedBy=multi-user.target
```

Key Points:
`After=network-online.target`: Don't start until network is available (can't update without internet!)

`Type=oneshot`: Run once and exit (not a continuously running service)

**Security hardening:** This is where I applied blue team thinking. The service runs with:

* Private temporary directory (`PrivateTmp=yes`)

* No privilege escalation (`NoNewPrivileges=true`)

* Read-only system files except what's explicitly needed

* Home directories hidden from the service

**Learning moment:** Understanding systemd security features is crucial for defensive security. Even automated maintenance tasks should follow the principle of least privilege.

**Part 3: The Systemd Timer**

The timer file (`/etc/systemd/system/system-update.timer`) schedules execution:

```
[Unit]
Description=Run system updates automatically
Requires=system-update.service

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=15min

[Install]
WantedBy=timers.target
```

Key Features:

`OnCalendar=daily:` Runs once per day (defaults to midnight). You can customize this:

* `OnCalendar=*-*-* 03:00:00` - Daily at 3 AM

* `OnCalendar=Sun *-*-* 02:00:00` - Sunday at 2 AM

* `OnCalendar=Mon,Wed,Fri *-*-* 04:00:00` - Three times per week

`Persistent=true:` If the system was off at the scheduled time, run as soon as it boots. This is brilliant for laptops or VMs that aren't always on.

`RandomizedDelaySec=15min:` Adds random delay up to 15 minutes. If you're managing multiple servers, this prevents them all hammering your mirror servers simultaneously.

**Installation: Putting It All Together**

Here's the complete installation process:
```
# 1. Copy the script
sudo cp system-update.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/system-update.sh

# 2. Copy service files
sudo cp system-update.service /etc/systemd/system/
sudo cp system-update.timer /etc/systemd/system/

# 3. Reload systemd
sudo systemctl daemon-reload

# 4. Enable and start the timer
sudo systemctl enable system-update.timer
sudo systemctl start system-update.timer

# 5. Verify it's working
systemctl status system-update.timer
systemctl list-timers system-update.timer
```



**Pro tip:** Test the service manually before relying on the timer:

```
sudo systemctl start system-update.service
sudo systemctl status system-update.service
sudo cat /var/log/system-updates/latest.log
```

**Lessons Learned**
1. **First Test**: I initially had an `Exec format error` in the `.service` file that always had it fail to run the bash script. I resolved it by adding `/bin/bash` to the path where the script is located in the `system_update,service` file.
2. **Check logs regularly**: Even automated systems need monitoring. I added a weekly calendar reminder to review the logs.
3. **Plan for reboots:** The script detects but doesn't auto-reboot. I created a separate process to review reboot notifications and schedule maintenance windows.
4. **Customize the schedule:** I started with daily updates at midnight but moved to 3 AM after realizing I sometimes have late-night SSH sessions that would get interrupted.
5. **Log rotation is crucial:** Without the 30-day cleanup, logs would eventually fill the disk. I tested this by temporarily setting retention to 7 days.

**Troubleshooting Common Issues**

**Problem: Timer shows as "dead"**
```
# Check timer status
systemctl status system-update.timer

# If dead, restart it
sudo systemctl restart system-update.timer
```

**Problem: Service failing**
```
# View detailed errors
sudo journalctl -xe -u system-update.service

# Common causes:
# - Script not executable: chmod +x /usr/local/bin/system-update.sh
# - Wrong path in service file: Check ExecStart path
# - Permission issues: Verify ReadWritePaths in service file
```

**Problem: Updates not running at scheduled time**
```
# Verify timer is enabled
systemctl is-enabled system-update.timer

# Check when it will run next
systemctl list-timers system-update.timer

# View timer configuration
systemctl cat system-update.timer
```

**Problem: Logs not being created**
```
# Create log directory manually
sudo mkdir -p /var/log/system-updates

# Check script has write access
sudo ls -ld /var/log/system-updates

# Test script manually
sudo /usr/local/bin/system-update.sh
```

**The Bigger Picture**

Building this automated update system taught me far more than just "how to schedule updates." I learned:

**Systemd architecture: **Understanding services, timers, targets, and dependencies.

**Security hardening: **Applying defense-in-depth to even maintenance tasks.

**Operational practices:** Logging, monitoring, and audit trails.

**Bash scripting:** Error handling, functions, and best practices

**System administration philosophy:** Automation, documentation, and reliability.

These skills directly support my goal of moving from system administration to cybersecurity engineering. Understanding how to secure and maintain infrastructure is fundamental to defending it.

**Resources & Code**

The complete code for this project is available in my GitHub repository:
[Linux SysAdmin Toolkit - System Updates](https://github.com/kayzfiq/linux-sysadmin-toolkit/tree/main/automation/system-updates)

The repository includes:

* Complete, commented source code
* Detailed installation instructions
* Troubleshooting guide
* Example logs

**Conclusion**

Automating system updates might seem like a simple task, but doing it properly—with logging, error handling, security hardening, and monitoring—requires thoughtful design. This project gave me hands-on experience with essential system administration skills while creating a tool I'll use throughout my career.

**Your Turn**

Have you automated your system updates? What challenges did you face? What improvements would you suggest? Drop your thoughts in the comments!

**Found this helpful?** Star the [GitHub repository](https://github.com/kayzfiq/linux-sysadmin-toolkit/tree/main/automation/system-updates) and share with others learning Linux administration.