+++
date = '2025-12-15T09:40:18+03:00'
draft = false
title = 'Today I Learned CPU monitoring and troubleshooting'
+++
## Understanding CPU architecture

One of the metrics used when monitoring and troubleshooting a Linux system bottlenecks is CPU usage. 
Before doing the actual monitoring, you need to know what you're working with and that's understanding the CPU architecture for accurate interpretation of the metrics.

Run `lscpu` to get this information

```
Dojo-Analyst@Dojo:~
$ lscpu 
Architecture:                x86_64
  CPU op-mode(s):            32-bit, 64-bit
  Address sizes:             40 bits physical, 48 bits virtual
  Byte Order:                Little Endian
CPU(s):                      2 						# Total Logical CPU's
  On-line CPU(s) list:       0,1
Vendor ID:                   AuthenticAMD
  Model name:                AMD A4-1250 APU with Radeon(TM) HD Graphics
    CPU family:              22
    Model:                   0
    Thread(s) per core:      1
    Core(s) per socket:      2			# 2 physical cores
    Socket(s):               1				# 1 physical CPU chip
```

Correct interpretation of the above information is crucial to understanding and interpreting load averages.

## Load Average: The Big Picture
When you run commands like `uptime` or 	`top`, you will see load average numbers: which represent the average number of processes that are either running, runnable 	(waiting for CPU time) or in uninterruptible sleep (waiting for I/O).

```
$ uptime 
 10:00:16 up  1:08,  1 user,  load average: 0.70, 0.48, 0.37
```

The load average numbers, represent three numbers 1-minute, 5-minute and 15-minutes averages respectively.

## Load average metric interpretation
Here is how to interpret them;

```
Load < Cores				Underutilized 						Normal, Capacity available
Load == Cores 				Fully utilized							Monitor trends
Load = 1.5 x Cores    	Moderate queuing				Investigate within minutes
Load = 2 x Cores			Heavy queuing					Immediate investigation
Load > 3 x Cores 			Severe saturation 				Emergency response
```

**Rule of thumb**: Sustained load average above 70% of your core count deserves investigation. Above 100% means you have performance degradation.

## Real-Time Monitoring with `top`

Run `top` command and let's decode what you see;

```
top - 16:46:35 up  7:54,  1 user,  load average: 0.09, 0.09, 0.07
Tasks: 213 total,   1 running, 209 sleeping,   3 stopped,   0 zombie
%Cpu(s): 28.6 us, 28.6 sy,  0.0 ni, 28.6 id,  0.0 wa,  0.0 hi, 14.3 si,  0.0 st 
```

* **us(user)**: Time spent running user-space processes. High value indicate application workload.
* **sy(system)**: Time spent in kernel space. High values (>20%) suggest excessive system calls, context switching or kernel bottlenecks.
* **ni(nice)**: Time running low priority processes. Usually negligible.
* **id(idle**): Unoccupied CPU time. Lower is busier.
* **wa(I/O wait)**: CPU waiting for I/O operations. **Critical metric**! High wa (>10%) means disk or network is your bottleneck, not CPU.
* **hi(hardware interrupts)**: Time servicing hardware interrupts. High values may indicate network card or disk controller issues.
* **si(software interrupts)**: Time handling software interrupts. High with network heavy workloads.
* **st(steal time)**: Only relevant in VM's. Time stolen by hypervisor for others VM's.

**Interactive monitoring with `top` command**

* Press `1` to see per-core breakdown (crucial for spotting single-threaded bottlenecks)
* Press `c` to show full command lines
* Press `P` to sort by CPU usage
* Press `M` to sort by memory
* Press `k` to kill a process

## Detailed Per-Process Analysis

**Find CPU hogs:**

`ps aux --sort=-%cpu | head -10`

This will show the top 10 CPU consumers with their PID, user, and command.

`top -o %CPU`

The key columns to look out for in the above are:

* %CPU: Can exceed 100%
* Time+: Total CPU time consumed 
* S (state): S=sleeping, R=running, Z=Zombie, D=uninterruptible (I/O wait)
* RES: Actual physical RAM used

**Detailed Per-Core CPU utilization**

`mpstat -P ALL 1`

This shows CPU usage broken down by each logical CPU, updating every second. Look for:

* One core at 100% while others idle - means single-threaded bottleneck
* All cores evenly loaded - means well-parallelized workload
* Uneven distribution = possible CPU affinity issues or poor threading.

## How to troubleshoot CPU issues

When someone reports the "system is slow":

1. Check load averages with `uptime` - is it above core count?
2. Run `top` and press `1` - which cores are busy?
3. Check `%wa`, if high, it's I/O, not CPU issue
4. Identify the culprit with `ps aux --sort=-%cpu | head`
5. Investigate that process: What is it doing? Is it expected? Can it be optimized or scheduled differently?

**Performance Tuning Tips**

* **CPU affinity:** Pin critical processes to specific cores with `taskset` to improve cache locality.
* **Nice values:** Lower priority of background tasks with nice or renice.
* **Process limits:** Check if processes hit CPU time limits with `ulimit -a`

The key is knowing when high CPU is actually the problem versus a symptom of something else usually I/O wait.

