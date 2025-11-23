+++
date = '2025-11-21T17:20:29+03:00'
draft = false
title = 'Today I Learned: Setting up system updates with systemd timer'
+++

*How I automated Linux system updates with comprehensive logging and learned why systemd timers beat cron*


**The Wake-Up Call**

Picture this: You're managing a handful of Linux servers (or even just one VPS), and you know you should be keeping them updated. Security patches are released constantly, but manually SSH-ing into each server, running ***apt update && apt upgrade***, waiting for it to complete, and checking for issues is... tedious.

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



This works, but it has limitations:

* ❌ Limited logging (redirecting to files is clunky)
* ❌ No dependency management
* ❌ Harder to debug when things go wrong
* ❌ No integration with system logging
* ❌ Runs even if system was off at scheduled time


Enter systemd timers.

***Why Systemd Timers Are Better***

Systemd timers are the modern replacement for cron, and they offer significant advantages:

* Native logging with **journalctl**
* Dependency management (run after network is available)
* Persistent timers (catches up if system was off)
* Easy monitoring with **systemctl status**
* Better error handling and failure notifications
* Security hardening options built-in


My Solution: Three-Part System

I built a complete automated update system consisting of:

1. Bash script - Does the actual updating with comprehensive logging
1. Systemd service - Defines how the script runs
2. Systemd timer - Schedules when it runs

Let me walk you through each component.

**Part 1: The Update Script**

I created /usr/local/bin/system-update.sh with these key features:

**Feature 1: Structured Logging**

Every action is logged with timestamps:

code snippet

This creates a new log file for each run (e.g., ***update-2024-11-14_03-00-00.log***) while also maintaining a ***latest.log*** symlink for quick access.

Why this matters: When troubleshooting issues or conducting security audits, I need to know exactly what happened and when. These logs provide a complete audit trail.

Feature 2: Comprehensive Update Process

The script performs a complete update cycle:

code snippet

Each step is wrapped in error handling:

code snippet

**Lesson learned**: Non-interactive updates are crucial for automation. The ***-y*** flag auto-confirms prompts, and proper error handling ensures failures are logged and visible.

**Feature 3: Reboot Detection**

One critical aspect of system updates: some patches require a reboot. My script detects this:

code snippet

**Why this matters:** In production environments, you need to know when manual intervention is required. I don't auto-reboot because downtime needs to be planned, but I get clear notification that action is needed.

**Feature 4: Automatic Log Cleanup**

Logs shouldn't pile up forever:

code snippet

Keeps 30 days of history—enough for troubleshooting and audits, not so much that disk space becomes an issue.

**Feature 5: Disk Space Reporting**

The script includes current disk usage in each log:

code snippet

This helps spot trends—if disk usage is climbing, I know to investigate before it becomes critical.

**Part 2: The Systemd Service**
	
The service file (***/etc/systemd/system/system-update.service***) defines how the script runs:

code snippet

Key Points:
***After=network-online.target***: Don't start until network is available (can't update without internet!)

***Type=oneshot***: Run once and exit (not a continuously running service)

**Security hardening:** This is where I applied blue team thinking. The service runs with:

* Private temporary directory (PrivateTmp=yes)

* No privilege escalation (NoNewPrivileges=true)

* Read-only system files except what's explicitly needed

* Home directories hidden from the service

**Learning moment:** Understanding systemd security features is crucial for defensive security. Even automated maintenance tasks should follow the principle of least privilege.

**Part 3: The Systemd Timer**

The timer file (/etc/systemd/system/system-update.timer) schedules execution:

code snippet

Key Features:

*OnCalendar=daily:* Runs once per day (defaults to midnight). You can customize this:

* OnCalendar=*-*-* 03:00:00 - Daily at 3 AM

* OnCalendar=Sun *-*-* 02:00:00 - Sunday at 2 AM

* OnCalendar=Mon,Wed,Fri *-*-* 04:00:00 - Three times per week

Persistent=true: If the system was off at the scheduled time, run as soon as it boots. This is brilliant for laptops or VMs that aren't always on.
RandomizedDelaySec=15min: Adds random delay up to 15 minutes. If you're managing multiple servers, this prevents them all hammering your mirror servers simultaneously.
Installation: Putting It All Together
Here's the complete installation process:


