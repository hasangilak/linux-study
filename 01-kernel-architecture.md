# Lesson 01: Linux Kernel Architecture

## Overview
The Linux kernel is the core of the operating system, managing hardware resources and providing essential services to applications. Understanding its architecture is crucial for system-level programming and optimization.

## Learning Approach: The STAR Method
This lesson uses the STAR (Situation, Task, Action, Result) method to teach kernel concepts through real-world scenarios. Each topic will be presented as:
- **Situation**: A real production problem or requirement
- **Task**: What needs to be accomplished
- **Action**: The specific steps and commands to use
- **Result**: The measurable outcome and verification

## Learning Objectives
- Understand kernel space vs user space
- Learn about kernel subsystems
- Explore system calls and kernel interfaces
- Understand kernel modules and drivers

## Kernel Architecture Layers

### 1. Hardware Layer
The physical components: CPU, memory, devices, peripherals.

### 2. Kernel Space
Protected memory area where kernel code executes with full privileges.

```
+------------------+
|   Applications   |  } User Space
+------------------+
|  System Libraries|
+------------------+
===== System Call Interface =====
+------------------+
|  Process Mgmt    |
|  Memory Mgmt     |  } Kernel Space
|  File Systems    |
|  Device Drivers  |
|  Network Stack   |
+------------------+
|    Hardware      |
+------------------+
```

### 3. Key Kernel Subsystems

#### Process Management
- Process scheduling (CFS - Completely Fair Scheduler)
- Process creation and termination
- Inter-process communication (IPC)

#### Memory Management
- Virtual memory management
- Page allocation and swapping
- Memory mapping

#### File System
- Virtual File System (VFS) layer
- File system implementations (ext4, XFS, Btrfs)
- Block I/O layer

#### Device Drivers
- Character devices
- Block devices
- Network devices

#### Networking
- Protocol implementation (TCP/IP stack)
- Socket interface
- Netfilter framework

## System Calls

System calls are the interface between user space and kernel space.

### Common System Calls:
```c
// File operations
open(), read(), write(), close()

// Process management
fork(), exec(), wait(), exit()

// Memory management
mmap(), munmap(), brk()

// Network operations
socket(), bind(), listen(), accept()
```

### STAR Scenarios for System Call Tracing

#### STAR Scenario 1: Application Permission Mystery

**Situation**: Your Node.js application deployed to production suddenly fails with generic "permission denied" errors. The logs don't specify which file or resource is causing the issue.

**Task**: Identify the exact file or resource causing the permission error without modifying the application code.

**Action**:
```bash
# Run the application with strace to capture all system calls
strace -e open,access,stat -o /tmp/app_trace.log node app.js

# Analyze the trace for permission errors
grep -E "EACCES|EPERM" /tmp/app_trace.log

# Example output:
# open("/var/log/app/debug.log", O_WRONLY|O_APPEND) = -1 EACCES (Permission denied)
```

**Result**: Discovered the app couldn't write to `/var/log/app/debug.log`. Fixed by adjusting directory permissions. Application uptime restored within 5 minutes.

#### STAR Scenario 2: Configuration File Discovery

**Situation**: After inheriting a legacy nginx setup, you need to document all configuration files the service uses, including dynamically included configs.

**Task**: Create a complete inventory of configuration files nginx reads during startup and config validation.

**Action**:
```bash
# Trace nginx config test to see all file operations
sudo strace -e open,openat nginx -t 2>&1 | grep -E "\.conf|\.crt|\.key" | grep -v ENOENT | sort -u

# Capture to file for documentation
sudo strace -e open,openat -o nginx_files.trace nginx -t
grep "open" nginx_files.trace | grep -v ENOENT | cut -d'"' -f2 | sort -u > nginx_config_inventory.txt
```

**Result**: Documented 23 configuration files including 5 undocumented includes. Created automated config backup script based on the inventory.

#### STAR Scenario 3: Performance Bottleneck Investigation

**Situation**: A Python data processing script that normally completes in 30 seconds now takes over 5 minutes. No code changes were made.

**Task**: Identify which operations are causing the slowdown without profiling tools.

**Action**:
```bash
# Trace with timing information
strace -r -T -o slow_analysis.log python data_processor.py

# Analyze slowest system calls
sort -k2 -n -r slow_analysis.log | head -20

# Focus on specific slow operations
grep -E "read|write" slow_analysis.log | awk '$2 > 0.1 {print $0}'
```

**Result**: Found that reads from NFS mount were taking 2-3 seconds each. Migrated data to local SSD, reducing runtime to 25 seconds.

#### STAR Scenario 4: Database Connection Debugging

**Situation**: MySQL client intermittently fails to connect to the database server with timeout errors, but ping shows the server is reachable.

**Task**: Trace the exact network operations to understand where the connection process fails.

**Action**:
```bash
# Trace network-related system calls with timestamps
strace -tt -e trace=network mysql -h db.example.com -u app_user -p 2>&1 | tee mysql_trace.log

# Analyze connection attempts
grep -E "socket|connect|send|recv" mysql_trace.log

# Check for timeouts or connection resets
grep -E "ETIMEDOUT|ECONNREFUSED|ECONNRESET" mysql_trace.log
```

**Result**: Discovered connection attempts to port 3306 were timing out. Found firewall rule limiting connections per minute. Adjusted connection pooling settings, eliminating 95% of timeout errors.

#### STAR Scenario 5: Complex Script Debugging

**Situation**: A bash deployment script works perfectly when run manually but fails when executed via cron with no clear error messages.

**Task**: Understand the execution flow and identify why the script behaves differently in cron environment.

**Action**:
```bash
# Trace the script execution including all child processes
strace -f -e trace=process,file -o cron_debug.log bash deployment.sh

# Analyze process creation and file access patterns
grep -E "execve|fork|open" cron_debug.log

# Compare environment differences
# Manual run:
strace -f -e trace=process -o manual_run.log bash deployment.sh
# Then compare the two traces
diff <(grep execve manual_run.log) <(grep execve cron_debug.log)
```

**Result**: Found script was relying on PATH environment variable not set in cron. Added explicit paths to all commands, achieving 100% reliability in automated deployments.

## Kernel Modules

Kernel modules are pieces of code that can be loaded and unloaded into the kernel on demand.

### STAR Scenarios for Kernel Module Management

#### STAR Scenario 1: USB Storage Device Recognition Failure

**Situation**: A critical USB backup drive containing daily backups is not recognized when plugged into the production backup server. The drive works on other machines.

**Task**: Get the USB drive recognized and mounted without rebooting the server, as it's handling active backup jobs.

**Action**:
```bash
# Step 1: Verify device is detected at hardware level
lsusb
# Output shows: Bus 002 Device 005: ID 0781:5583 SanDisk Corp. Ultra Fit

# Step 2: Check if USB storage module is loaded
lsmod | grep -E "usb_storage|uas"
# No output - modules not loaded

# Step 3: Load required modules
sudo modprobe usb-storage
sudo modprobe uas  # For USB Attached SCSI

# Step 4: Verify device recognition
dmesg | tail -20
# Should show: "USB Mass Storage device detected"

# Step 5: Confirm block device creation
ls /dev/sd*
```

**Result**: USB drive recognized as `/dev/sdb`, mounted successfully. Implemented module auto-loading rule to prevent future issues. Zero downtime achieved.

#### STAR Scenario 2: 10G Network Card Performance Optimization

**Situation**: Newly installed 10G network cards are only achieving 3Gbps throughput on database replication traffic, causing replication lag.

**Task**: Optimize the network card driver parameters to achieve at least 8Gbps sustained throughput.

**Action**:
```bash
# Step 1: Identify network driver
ethtool -i eth2
# driver: ixgbe
# version: 5.1.0-k

# Step 2: Check current module parameters
modinfo ixgbe | grep "^parm"
# Shows available parameters including IntMode, InterruptThrottleRate

# Step 3: Test current performance
iperf3 -c 10.0.0.2 -t 30

# Step 4: Reload module with optimized parameters
sudo modprobe -r ixgbe
sudo modprobe ixgbe IntMode=2 InterruptThrottleRate=0,0,0,0

# Step 5: Make changes persistent
echo "options ixgbe IntMode=2 InterruptThrottleRate=0,0,0,0" | sudo tee /etc/modprobe.d/ixgbe.conf

# Step 6: Verify performance improvement
iperf3 -c 10.0.0.2 -t 30
```

**Result**: Achieved 9.4Gbps throughput, eliminating replication lag. Documented optimization for all database servers.

#### STAR Scenario 3: Filesystem Module Parameter Tuning

**Situation**: An ext4 filesystem on SSD storage shows high latency during database writes, affecting application response times.

**Task**: Tune ext4 module parameters for optimal SSD performance without reformatting.

**Action**:
```bash
# Step 1: Check current ext4 parameters
ls /sys/module/ext4/parameters/
cat /sys/module/ext4/parameters/*

# Step 2: Research optimal SSD parameters
modinfo ext4 | grep -B1 -A1 "parm:"

# Step 3: Apply runtime changes for testing
# Disable barriers for better SSD performance (if UPS protected)
mount -o remount,nobarrier /data

# Step 4: Test with database workload
# Run your database benchmark here

# Step 5: Make changes permanent in /etc/fstab
# /dev/nvme0n1p1 /data ext4 defaults,nobarrier,noatime 0 2
```

**Result**: Write latency reduced by 40%, database transaction throughput increased by 35%. Applied settings to all SSD-based database servers.

#### STAR Scenario 4: KVM Virtualization Emergency Setup

**Situation**: Physical server failure requires immediate migration of VMs to a spare server that doesn't have virtualization configured.

**Task**: Enable KVM virtualization support and verify hardware acceleration is working.

**Action**:
```bash
# Step 1: Check CPU virtualization support
grep -E "vmx|svm" /proc/cpuinfo
# vmx = Intel VT-x, svm = AMD-V

# Step 2: Load KVM modules based on CPU type
CPU_TYPE=$(grep -m1 "vendor_id" /proc/cpuinfo | awk '{print $3}')
if [ "$CPU_TYPE" = "GenuineIntel" ]; then
    sudo modprobe kvm_intel
else
    sudo modprobe kvm_amd
fi

# Step 3: Verify modules loaded correctly
lsmod | grep kvm
# kvm_intel  315392  0
# kvm        847872  1 kvm_intel

# Step 4: Check acceleration is available
ls -la /dev/kvm
# crw-rw---- 1 root kvm 10, 232 Jan 10 14:22 /dev/kvm

# Step 5: Ensure modules load at boot
echo "kvm_intel" | sudo tee -a /etc/modules-load.d/kvm.conf
```

**Result**: KVM enabled successfully, migrated 12 VMs within 2 hours. Created automation script for future emergency preparations.

#### STAR Scenario 5: Bluetooth Audio Device Connectivity

**Situation**: Executive's Bluetooth headset won't connect to the conference room Ubuntu system, potentially disrupting an important client presentation.

**Task**: Enable Bluetooth support and ensure audio devices can connect reliably.

**Action**:
```bash
# Step 1: Check Bluetooth hardware presence
lsusb | grep -i bluetooth
# Bus 001 Device 003: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle

# Step 2: Verify Bluetooth modules
lsmod | grep bluetooth
# No output - modules not loaded

# Step 3: Load required Bluetooth modules
sudo modprobe bluetooth
sudo modprobe btusb
sudo modprobe rfcomm  # For serial devices
sudo modprobe bnep    # For network devices

# Step 4: Start Bluetooth service
sudo systemctl start bluetooth
sudo systemctl enable bluetooth

# Step 5: Verify Bluetooth is operational
sudo bluetoothctl power on
sudo bluetoothctl scan on
```

**Result**: Bluetooth headset connected successfully. Created startup script to ensure Bluetooth is always available. Presentation proceeded without technical issues.

### Creating a Simple Module:
```c
// hello.c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int __init hello_init(void) {
    printk(KERN_INFO "Hello, Kernel!\n");
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye, Kernel!\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("A simple kernel module");
```

### Makefile for Module:
```makefile
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

## Kernel Configuration

### Viewing Current Configuration:
```bash
# Current running kernel config
zcat /proc/config.gz

# Or from boot directory
cat /boot/config-$(uname -r)

# Search for specific options
grep CONFIG_CGROUPS /boot/config-$(uname -r)
```

### STAR Scenarios for Kernel Parameters

#### STAR Scenario 1: Application File Descriptor Crisis

**Situation**: Production Java application serving 10,000 concurrent users starts throwing "Too many open files" exceptions during peak hours, causing service degradation.

**Task**: Increase system file descriptor limits to handle the load without restarting the application or server.

**Action**:
```bash
# Step 1: Check current system-wide limit
sysctl fs.file-max
# fs.file-max = 65536

# Step 2: Monitor current usage
cat /proc/sys/fs/file-nr
# 8192	0	65536  (used, free, max)

# Step 3: Calculate required limit (10k users * 10 fd/user + 20% buffer)
# Required: 120,000

# Step 4: Apply new limit immediately
sudo sysctl fs.file-max=200000

# Step 5: Make change permanent
echo "fs.file-max=200000" | sudo tee -a /etc/sysctl.d/99-app-limits.conf

# Step 6: Also update per-process limits
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# Step 7: Verify changes
sysctl fs.file-max
ulimit -n
```

**Result**: Application stabilized immediately. Zero errors in following 72 hours. Implemented monitoring alert at 80% utilization.

#### STAR Scenario 2: E-commerce Site Black Friday Optimization

**Situation**: E-commerce platform expecting 10x normal traffic on Black Friday. Previous year saw connection timeouts and lost sales worth $500K.

**Task**: Optimize TCP/IP stack to handle massive concurrent connections without hardware upgrades.

**Action**:
```bash
# Step 1: Baseline current settings
sysctl net.ipv4.tcp_fin_timeout net.core.somaxconn net.ipv4.tcp_max_syn_backlog

# Step 2: Apply aggressive TCP optimizations
# Reduce TIME_WAIT duration
sudo sysctl net.ipv4.tcp_fin_timeout=15

# Increase connection queues
sudo sysctl net.core.somaxconn=4096
sudo sysctl net.ipv4.tcp_max_syn_backlog=8192

# Enable TCP Fast Open
sudo sysctl net.ipv4.tcp_fastopen=3

# Optimize for many short connections
sudo sysctl net.ipv4.tcp_tw_reuse=1
sudo sysctl net.ipv4.tcp_tw_recycle=0  # Deprecated, ensure disabled

# Increase local port range
sudo sysctl net.ipv4.ip_local_port_range="1024 65535"

# Step 3: Save all optimizations
cat << EOF | sudo tee /etc/sysctl.d/99-blackfriday.conf
net.ipv4.tcp_fin_timeout=15
net.core.somaxconn=4096
net.ipv4.tcp_max_syn_backlog=8192
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_tw_reuse=1
net.ipv4.ip_local_port_range=1024 65535
EOF

# Step 4: Load test to verify
# Run your load testing tool
```

**Result**: Handled 850K concurrent connections (previous max: 80K). Zero timeouts during Black Friday. Revenue increased 40% YoY.

#### STAR Scenario 3: PostgreSQL Shared Memory Optimization

**Situation**: PostgreSQL database migration to new server fails with "could not create shared memory segment" errors. Database requires 32GB shared memory for optimal performance.

**Task**: Configure kernel to support PostgreSQL's large shared memory requirements.

**Action**:
```bash
# Step 1: Calculate required shared memory
# PostgreSQL needs: 32GB shared_buffers + overhead
# Total required: ~35GB

# Step 2: Check current limits
sysctl kernel.shmmax kernel.shmall kernel.shmmni
# kernel.shmmax = 68719476736  (64GB)
# kernel.shmall = 4294967296   (16GB in pages)
# kernel.shmmni = 4096

# Step 3: Calculate and set new values
# shmmax: max single segment size (35GB)
sudo sysctl kernel.shmmax=37580963840

# shmall: total pages (35GB / 4KB page size)
sudo sysctl kernel.shmall=9175040

# Step 4: Apply huge pages for better performance
sudo sysctl vm.nr_hugepages=16384  # 32GB in 2MB pages

# Step 5: Make permanent
cat << EOF | sudo tee /etc/sysctl.d/99-postgresql.conf
kernel.shmmax=37580963840
kernel.shmall=9175040
vm.nr_hugepages=16384
vm.hugetlb_shm_group=$(id -g postgres)
EOF

# Step 6: Reload and verify
sudo sysctl -p /etc/sysctl.d/99-postgresql.conf
```

**Result**: PostgreSQL started successfully with 32GB shared buffers. Query performance improved 60%. Huge pages reduced memory overhead by 15%.

#### STAR Scenario 4: DDoS Attack Mitigation

**Situation**: Public-facing web server experiencing SYN flood attack, legitimate users cannot connect. Attack traffic: 100K SYN packets/second.

**Task**: Harden kernel network stack to mitigate attack while maintaining service for legitimate users.

**Action**:
```bash
# Step 1: Enable SYN cookies immediately
sudo sysctl net.ipv4.tcp_syncookies=1

# Step 2: Reduce SYN-ACK retries
sudo sysctl net.ipv4.tcp_synack_retries=2

# Step 3: Enable reverse path filtering
sudo sysctl net.ipv4.conf.all.rp_filter=1
sudo sysctl net.ipv4.conf.default.rp_filter=1

# Step 4: Limit ICMP responses
sudo sysctl net.ipv4.icmp_echo_ignore_broadcasts=1
sudo sysctl net.ipv4.icmp_ignore_bogus_error_responses=1

# Step 5: Rate limit new connections
sudo iptables -A INPUT -p tcp --syn -m limit --limit 50/s --limit-burst 100 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

# Step 6: Apply comprehensive hardening
cat << EOF | sudo tee /etc/sysctl.d/99-ddos-protection.conf
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_synack_retries=2
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv4.icmp_ignore_bogus_error_responses=1
net.ipv4.tcp_max_syn_backlog=4096
net.ipv4.tcp_fin_timeout=15
EOF

# Step 7: Monitor attack mitigation
watch -n 1 'netstat -an | grep SYN_RECV | wc -l'
```

**Result**: Attack mitigated within 5 minutes. Legitimate traffic restored. Implemented permanent DDoS protection playbook.

#### STAR Scenario 5: High-Speed Network Optimization

**Situation**: New 40Gbps network link between data centers only achieving 15Gbps throughput for backup transfers, delaying critical backup windows.

**Task**: Optimize kernel network buffers and parameters for maximum throughput on high-bandwidth, high-latency link.

**Action**:
```bash
# Step 1: Calculate optimal buffer sizes
# Bandwidth-Delay Product: 40Gbps * 50ms = 250MB

# Step 2: Check current buffer limits
sysctl net.core.rmem_max net.core.wmem_max

# Step 3: Set appropriate buffer sizes
sudo sysctl net.core.rmem_max=536870912  # 512MB
sudo sysctl net.core.wmem_max=536870912  # 512MB

# TCP specific buffers (min, default, max)
sudo sysctl net.ipv4.tcp_rmem="4096 87380 536870912"
sudo sysctl net.ipv4.tcp_wmem="4096 65536 536870912"

# Step 4: Enable TCP tuning features
sudo sysctl net.ipv4.tcp_window_scaling=1
sudo sysctl net.ipv4.tcp_timestamps=1
sudo sysctl net.ipv4.tcp_sack=1

# Step 5: Adjust congestion control for datacenter use
sudo sysctl net.ipv4.tcp_congestion_control=bbr

# Step 6: Make permanent
cat << EOF | sudo tee /etc/sysctl.d/99-highspeed-network.conf
net.core.rmem_max=536870912
net.core.wmem_max=536870912
net.ipv4.tcp_rmem=4096 87380 536870912
net.ipv4.tcp_wmem=4096 65536 536870912
net.ipv4.tcp_congestion_control=bbr
net.core.netdev_max_backlog=5000
EOF

# Step 7: Test throughput
iperf3 -c remote-datacenter -t 60
```

**Result**: Achieved 38.5Gbps sustained throughput. Backup window reduced from 8 hours to 2 hours. ROI on network upgrade realized.

## The /proc and /sys Filesystems

### STAR Scenarios for /proc Filesystem

#### STAR Scenario 1: Production Application CPU Spike Investigation

**Situation**: Node.js application suddenly consuming 400% CPU (4 cores maxed out) in production. Users reporting slow response times. No recent deployments.

**Task**: Identify root cause of CPU spike and gather evidence for developers without stopping the application.

**Action**:
```bash
# Step 1: Identify the problematic process
top -b -n 1 | head -20
# Found: node process PID 15432 using 398% CPU

# Step 2: Get process details
cat /proc/15432/status | grep -E "Name|State|Threads|Cpus_allowed"
# Name:	node
# State:	R (running)
# Threads:	24

# Step 3: Check which threads are consuming CPU
ps -L -p 15432 -o pid,tid,psr,pcpu,comm
# Shows 4 threads at 99% CPU each

# Step 4: Examine open files to understand workload
ls -la /proc/15432/fd/ | wc -l
# 1847 open file descriptors

# Step 5: Check process memory maps for anomalies
cat /proc/15432/maps | grep -E "heap|stack" | tail -5

# Step 6: Capture stack traces for analysis
cat /proc/15432/stack
sudo gdb -p 15432 -batch -ex "thread apply all bt" -ex "quit" > stacktrace.txt
```

**Result**: Discovered infinite loop in logging module triggered by malformed user input. Deployed hotfix within 30 minutes. CPU usage normalized immediately.

#### STAR Scenario 2: Memory Leak Detection and Quantification

**Situation**: Python data pipeline gradually consuming all server memory over 48 hours, requiring daily restarts. Need to prove memory leak to development team.

**Task**: Document memory growth pattern and identify specific memory regions growing abnormally.

**Action**:
```bash
# Step 1: Create monitoring script
cat > monitor_memory.sh << 'EOF'
#!/bin/bash
PID=$1
LOGFILE="memory_trace_${PID}.log"

echo "Timestamp,VmSize,VmRSS,VmData,VmStk,Heap" > $LOGFILE

while true; do
    TIMESTAMP=$(date +%s)
    VMSIZE=$(grep VmSize /proc/$PID/status | awk '{print $2}')
    VMRSS=$(grep VmRSS /proc/$PID/status | awk '{print $2}')
    VMDATA=$(grep VmData /proc/$PID/status | awk '{print $2}')
    VMSTK=$(grep VmStk /proc/$PID/status | awk '{print $2}')
    HEAP=$(grep heap /proc/$PID/maps | awk '{split($1,a,"-"); print strtonum("0x"a[2]) - strtonum("0x"a[1])}' | awk '{sum+=$1} END {print sum/1024}')
    
    echo "$TIMESTAMP,$VMSIZE,$VMRSS,$VMDATA,$VMSTK,$HEAP" >> $LOGFILE
    sleep 60
done
EOF

chmod +x monitor_memory.sh
./monitor_memory.sh 8745 &

# Step 2: After 4 hours, analyze growth
tail -20 memory_trace_8745.log

# Step 3: Deep dive into memory maps
cat /proc/8745/smaps | grep -A 10 heap
```

**Result**: Documented 2GB/hour memory growth in heap. Identified leaked database connection objects. Fix deployed, memory stable for 7 days continuous operation.

#### STAR Scenario 3: Capacity Planning Data Collection

**Situation**: CFO requesting hardware budget for next year. Need accurate data on current system utilization across 50 production servers.

**Task**: Create automated system inventory and utilization report for capacity planning.

**Action**:
```bash
# Step 1: Create comprehensive system profiler
cat > system_profile.sh << 'EOF'
#!/bin/bash

echo "=== System Profile Report ==="
echo "Generated: $(date)"
echo

echo "=== CPU Information ==="
grep -E "processor|model name|cpu MHz|cache size" /proc/cpuinfo | sort -u

echo -e "\n=== Memory Information ==="
grep -E "MemTotal|MemFree|MemAvailable|SwapTotal" /proc/meminfo

echo -e "\n=== Current Load ==="
cat /proc/loadavg

echo -e "\n=== Process Limits ==="
cat /proc/sys/kernel/pid_max
cat /proc/sys/kernel/threads-max

echo -e "\n=== File System Limits ==="
cat /proc/sys/fs/file-nr

echo -e "\n=== Network Settings ==="
cat /proc/sys/net/core/somaxconn
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
EOF

# Step 2: Run across all servers
for server in $(cat servers.txt); do
    ssh $server 'bash -s' < system_profile.sh > "profile_${server}.txt"
done

# Step 3: Generate summary report
# Aggregate data for executive summary
```

**Result**: Identified 30% overcapacity in compute, 85% memory utilization. Recommended targeted memory upgrades saving $200K vs full server refresh.

### STAR Scenarios for /sys Filesystem

#### STAR Scenario 1: SSD Performance Optimization

**Situation**: Database server migration to NVMe SSDs showing only 20% performance improvement instead of expected 10x. Random I/O performance particularly poor.

**Task**: Optimize kernel I/O subsystem for NVMe SSD characteristics to achieve expected performance gains.

**Action**:
```bash
# Step 1: Check current I/O scheduler
cat /sys/block/nvme0n1/queue/scheduler
# [mq-deadline] kyber none

# Step 2: Switch to optimal scheduler for NVMe
echo none | sudo tee /sys/block/nvme0n1/queue/scheduler

# Step 3: Optimize queue parameters
# Increase queue depth for NVMe
echo 1024 | sudo tee /sys/block/nvme0n1/queue/nr_requests

# Disable rotational flag
echo 0 | sudo tee /sys/block/nvme0n1/queue/rotational

# Optimal read-ahead for database workload
echo 0 | sudo tee /sys/block/nvme0n1/queue/read_ahead_kb

# Step 4: Set optimal CPU affinity for NVMe interrupts
cat /proc/interrupts | grep nvme
echo 2 | sudo tee /proc/irq/24/smp_affinity_list

# Step 5: Make changes persistent
cat << EOF | sudo tee /etc/udev/rules.d/99-nvme-optimize.rules
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/nr_requests}="1024"
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/read_ahead_kb}="0"
EOF
```

**Result**: Random I/O performance increased 8.5x, sequential reads hit 3.5GB/s. Database query times reduced by 70%.

#### STAR Scenario 2: Network Packet Loss Diagnosis

**Situation**: Video streaming service experiencing 0.1% packet loss during peak hours, causing buffering for thousands of users. Network team claims infrastructure is fine.

**Task**: Prove whether packet loss is occurring at the server level and identify the bottleneck.

**Action**:
```bash
# Step 1: Check interface statistics
for i in {1..10}; do
    echo "=== $(date) ==="
    cat /sys/class/net/eth0/statistics/rx_dropped
    cat /sys/class/net/eth0/statistics/rx_errors
    cat /sys/class/net/eth0/statistics/rx_fifo_errors
    sleep 10
done

# Step 2: Monitor ring buffer status
ethtool -S eth0 | grep -E "rx_no_buffer|rx_missed"

# Step 3: Check current ring buffer size
ethtool -g eth0

# Step 4: Increase ring buffer size
sudo ethtool -G eth0 rx 4096 tx 4096

# Step 5: Monitor CPU interrupt handling
watch -n 1 'cat /proc/interrupts | grep eth0'

# Step 6: Optimize interrupt coalescing
sudo ethtool -C eth0 adaptive-rx on adaptive-tx on
```

**Result**: Identified ring buffer overflows during traffic spikes. Increased buffers and enabled adaptive interrupt coalescing. Packet loss eliminated, customer complaints dropped 100%.

#### STAR Scenario 3: Server Power Consumption Optimization

**Situation**: Data center power costs increased 40%. Management demands 20% reduction in power consumption without performance impact.

**Task**: Implement CPU power management to reduce consumption during low-load periods while maintaining performance SLAs.

**Action**:
```bash
# Step 1: Check current CPU governor
for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    cat $cpu/cpufreq/scaling_governor
done

# Step 2: Analyze load patterns
# Collect 24-hour CPU utilization data
sar -u 1 86400 > cpu_utilization.log

# Step 3: Implement intelligent power management
# Create load-based governor switching
cat > /usr/local/bin/adaptive_cpu_power.sh << 'EOF'
#!/bin/bash
LOAD=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | cut -d. -f1)
CPUS=$(nproc)
LOAD_PER_CPU=$((LOAD / CPUS))

if [ $LOAD_PER_CPU -gt 80 ]; then
    # High load - maximum performance
    for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
        echo performance > $cpu/cpufreq/scaling_governor
    done
else
    # Normal load - balanced power
    for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
        echo powersave > $cpu/cpufreq/scaling_governor
    done
fi
EOF

# Step 4: Schedule regular governor updates
echo "* * * * * root /usr/local/bin/adaptive_cpu_power.sh" >> /etc/crontab

# Step 5: Monitor power consumption
turbostat --interval 60 > power_consumption.log &
```

**Result**: Achieved 25% power reduction during off-peak hours. Annual savings: $45,000 per rack. Zero impact on application SLAs.

#### STAR Scenario 4: USB Peripheral Reliability

**Situation**: Point-of-sale system USB barcode scanners disconnecting randomly throughout the day, disrupting checkout operations at 200 stores.

**Task**: Stabilize USB connectivity to achieve 99.9% uptime for peripheral devices.

**Action**:
```bash
# Step 1: Identify USB devices and their power status
for device in /sys/bus/usb/devices/*/power/control; do
    echo "Device: $device"
    cat $device
done

# Step 2: Disable USB autosuspend globally
echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend

# Step 3: Make specific device rules
# Find barcode scanner vendor/product ID
lsusb | grep -i "barcode\|scanner"
# Bus 001 Device 003: ID 05f9:2206 PSC Scanning, Inc.

# Step 4: Create udev rule for scanners
cat << EOF | sudo tee /etc/udev/rules.d/99-barcode-scanner.rules
# Disable power management for barcode scanners
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="05f9", ATTR{idProduct}=="2206", TEST=="power/control", ATTR{power/control}="on"
EOF

# Step 5: Apply rules and verify
sudo udevadm control --reload-rules
sudo udevadm trigger

# Verify settings applied
cat /sys/bus/usb/devices/*/power/control | grep -v auto
```

**Result**: USB disconnections eliminated. Checkout system uptime improved to 99.95%. Saved 500 employee hours/month in manual scanner resets.

## STAR Learning Exercises

These exercises follow the STAR method to reinforce learning through practical scenarios.

### Exercise 1: Performance Crisis Response

**Situation**: Your web application is experiencing intermittent slowdowns. Users report 10-second page loads that normally take 1 second. You need to diagnose without stopping the service.

**Task**: Use kernel tools to identify the bottleneck and provide actionable data to the development team.

**Your Action Plan**:
1. Use strace to sample system calls from the web server process
2. Check /proc filesystem for memory and CPU patterns
3. Analyze /sys filesystem for I/O bottlenecks
4. Document findings with specific metrics

**Expected Result**: Identify whether the issue is CPU, memory, I/O, or network related, with specific evidence.

```bash
# Starter commands for your investigation:
# 1. Find web server PID
ps aux | grep -E "nginx|apache|node"

# 2. Sample system calls (replace PID)
sudo strace -c -p PID -f -T 30

# 3. Check process status
cat /proc/PID/status

# Document your findings here...
```

### Exercise 2: Hardware Compatibility Challenge

**Situation**: You've received 10 new USB 3.0 external drives for backup purposes, but they're not being recognized on your backup servers running Ubuntu 24 LTS.

**Task**: Get the drives working without rebooting the production backup servers.

**Your Action Plan**:
1. Check if USB 3.0 modules are loaded
2. Investigate kernel messages for errors
3. Load appropriate modules if missing
4. Verify drives are recognized
5. Create persistent configuration

**Expected Result**: All drives recognized and available for backup operations.

```bash
# Investigation steps:
# Check current USB modules
lsmod | grep -E "usb|xhci"

# Monitor kernel messages while plugging in drive
dmesg -w

# Your solution here...
```

### Exercise 3: Database Server Optimization

**Situation**: Your PostgreSQL server can only handle 100 concurrent connections before throwing "out of memory" errors, but you need to support 500 connections for a new microservices architecture.

**Task**: Tune kernel parameters to support 500 concurrent database connections with 64GB RAM available.

**Your Action Plan**:
1. Calculate memory requirements (500 connections × 10MB each = 5GB)
2. Check current kernel limits
3. Adjust shared memory settings
4. Tune file descriptor limits
5. Test with connection pooler

**Expected Result**: Database handles 500 concurrent connections successfully.

```bash
# Current state analysis:
sysctl kernel.shmmax kernel.shmall
sysctl fs.file-max
ulimit -n

# Your optimization steps...
```

### Exercise 4: Network Performance Investigation

**Situation**: After migrating to 10Gbps network cards, file transfers between servers only achieve 1Gbps, exactly the same as the old 1Gbps cards.

**Task**: Identify and resolve the network performance bottleneck.

**Your Action Plan**:
1. Check network driver and parameters
2. Verify TCP tuning parameters
3. Test with different congestion control algorithms
4. Optimize interrupt handling
5. Document before/after performance

**Expected Result**: Achieve at least 8Gbps sustained transfer rate.

```bash
# Diagnostic commands:
ethtool eth0
sysctl net.ipv4.tcp_congestion_control
cat /proc/interrupts | grep eth

# Performance testing:
iperf3 -s  # On one server
iperf3 -c SERVER_IP -t 30  # On another

# Your solution...
```

### Exercise 5: Emergency Module Recovery

**Situation**: A junior admin accidentally removed the ext4 module on a running system. The root filesystem is ext4, and you cannot reboot as it would fail to mount root.

**Task**: Restore the ext4 module without rebooting the system.

**Your Action Plan**:
1. Verify current situation (system still running because module was in use)
2. Find the module file on the system
3. Load the module back into kernel
4. Ensure it persists across reboot
5. Test on a non-critical mount point

**Expected Result**: ext4 module restored and system reboot-safe.

```bash
# CAREFUL: This is a critical operation
# Check module status
lsmod | grep ext4

# Find module location
find /lib/modules -name "ext4.ko*"

# Your recovery steps...
```

### Self-Assessment Checklist

After completing each exercise, verify:
- [ ] Identified the root cause with specific evidence
- [ ] Implemented solution without service disruption
- [ ] Documented measurable improvements
- [ ] Created persistent configuration where needed
- [ ] Can explain the solution to team members

### Creating Your Own STAR Scenarios

Template for documenting your real-world experiences:

```markdown
**Situation**: [Describe the problem, impact, and constraints]

**Task**: [Define specific objectives and success criteria]

**Action**:
- Step 1: [Investigation phase]
- Step 2: [Root cause analysis]
- Step 3: [Solution implementation]
- Step 4: [Verification]
- Step 5: [Documentation]

**Result**: [Quantify improvements, lessons learned]
```

## Advanced Topics to Explore

1. **Kernel Debugging**
   - Using `ftrace` for function tracing
   - `perf` for performance analysis
   - `eBPF` for advanced tracing

2. **Real-time Kernel**
   - PREEMPT_RT patches
   - Real-time scheduling policies

3. **Security Modules**
   - SELinux integration
   - AppArmor
   - Custom LSM development

## Commands Reference

### Quick Scenario-Based Command Guide

**Kernel Information Scenarios:**
```bash
# Before installing software, check kernel compatibility:
uname -r  # Get kernel release version
cat /proc/version  # Full version with compiler info

# Verify system architecture for downloads:
uname -m  # Shows x86_64, aarch64, etc.
```

**Module Management Scenarios:**
```bash
# WiFi not working? Check wireless modules:
lsmod | grep -E "iwl|ath|rtl"  # Common wireless drivers

# Need module details before loading:
modinfo nvidia  # Check before installing graphics driver
sudo modprobe -n --first-time module_name  # Dry run

# Safely remove unused modules:
sudo rmmod module_name
```

**Kernel Messages Scenarios:**
```bash
# Just plugged in USB device:
dmesg | tail -20  # See recent kernel messages

# System acting strange after update:
journalctl -k --since "1 hour ago"  # Kernel logs from last hour

# Hardware errors suspected:
dmesg | grep -i error  # Search for error messages
```

**System Call Debugging Scenarios:**
```bash
# Profile application performance:
strace -c python script.py  # Count syscalls and time

# Debug library loading issues:
ltrace -e "*@*" ./app  # Trace library calls
```

**Kernel Parameter Scenarios:**
```bash
# Quick system tuning check:
sysctl -a | grep -E "vm.swappiness|vm.dirty"  # Memory tuning

# Apply production server optimizations:
sysctl -p /etc/sysctl.d/99-custom.conf  # Load custom settings

# Emergency parameter change:
sysctl -w kernel.panic=10  # Reboot 10s after kernel panic
```

## Further Reading

- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)
- [Linux Device Drivers, 3rd Edition](https://lwn.net/Kernel/LDD3/)
- [Understanding the Linux Kernel](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/)
- [Linux Kernel Development by Robert Love](https://www.pearson.com/us/higher-education/program/Love-Linux-Kernel-Development-3rd-Edition/PGM202532.html)

## Next Lesson
[02. Systemd and Init Systems](02-systemd-init.md) - Learn about modern service management in Linux.