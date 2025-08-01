# Lesson 01: Linux Kernel Architecture

## Overview
The Linux kernel is the core of the operating system, managing hardware resources and providing essential services to applications. Understanding its architecture is crucial for system-level programming and optimization.

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

### Tracing System Calls:
```bash
# Trace system calls of a command
strace ls -la

# Trace specific system calls
strace -e open,read,write cat /etc/passwd

# Trace with timestamps
strace -t date

# Follow child processes
strace -f bash -c "echo hello | grep h"
```

## Kernel Modules

Kernel modules are pieces of code that can be loaded and unloaded into the kernel on demand.

### Working with Modules:
```bash
# List loaded modules
lsmod

# Get module information
modinfo ext4

# Load a module
sudo modprobe module_name

# Remove a module
sudo modprobe -r module_name

# View module parameters
cat /sys/module/*/parameters/*
```

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

### Kernel Parameters:
```bash
# View all kernel parameters
sysctl -a

# View specific parameter
sysctl kernel.hostname

# Change parameter temporarily
sudo sysctl kernel.hostname=newname

# Make permanent changes
echo "kernel.hostname=newname" | sudo tee -a /etc/sysctl.conf
```

## The /proc and /sys Filesystems

### /proc - Process and System Information
```bash
# CPU information
cat /proc/cpuinfo

# Memory information
cat /proc/meminfo

# Kernel version
cat /proc/version

# Process information
ls /proc/$$  # Current shell process

# Kernel command line
cat /proc/cmdline
```

### /sys - Sysfs for Device and Driver Information
```bash
# Block devices
ls /sys/block/

# Network devices
ls /sys/class/net/

# CPU information
ls /sys/devices/system/cpu/

# Module parameters
ls /sys/module/
```

## Practical Exercises

### Exercise 1: System Call Exploration
```bash
# Create a simple C program that uses system calls
cat > syscall_test.c << 'EOF'
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    int fd = open("/tmp/test.txt", O_CREAT | O_WRONLY, 0644);
    write(fd, "Hello from syscall\n", 19);
    close(fd);
    return 0;
}
EOF

# Compile and trace
gcc syscall_test.c -o syscall_test
strace -e open,write,close ./syscall_test
```

### Exercise 2: Kernel Module Development
1. Create the hello module from the example above
2. Compile and load it
3. Check kernel logs with `dmesg`
4. Unload the module

### Exercise 3: Kernel Parameter Tuning
```bash
# Explore network parameters
sysctl -a | grep net.ipv4.tcp

# Test impact of changing TCP congestion control
cat /proc/sys/net/ipv4/tcp_congestion_control
echo "bbr" | sudo tee /proc/sys/net/ipv4/tcp_congestion_control

# Monitor with:
ss -tin
```

### Exercise 4: Process Investigation
```bash
# Pick a running process
ps aux | grep sshd

# Explore its proc entry
sudo ls -la /proc/$(pidof sshd)
sudo cat /proc/$(pidof sshd)/maps
sudo cat /proc/$(pidof sshd)/status
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

```bash
# Kernel version and info
uname -a
cat /proc/version

# Module management
lsmod | grep module_name
modinfo module_name
sudo modprobe module_name
sudo rmmod module_name

# Kernel messages
dmesg | tail -20
journalctl -k

# System calls
strace -c command  # Count system calls
ltrace command     # Library calls

# Kernel parameters
sysctl -a | less
sysctl parameter_name
sysctl -w parameter=value
```

## Further Reading

- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)
- [Linux Device Drivers, 3rd Edition](https://lwn.net/Kernel/LDD3/)
- [Understanding the Linux Kernel](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/)
- [Linux Kernel Development by Robert Love](https://www.pearson.com/us/higher-education/program/Love-Linux-Kernel-Development-3rd-Edition/PGM202532.html)

## Next Lesson
[02. Systemd and Init Systems](02-systemd-init.md) - Learn about modern service management in Linux.