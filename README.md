# Linux Mastery: From Developer to System Expert

A comprehensive Linux curriculum designed for experienced developers transitioning to Linux system mastery.

## Learning Path Overview

### Section 1: System Architecture & Boot Process
- [01. Linux Kernel Architecture](01-kernel-architecture.md) - Understanding the heart of Linux
- [02. Systemd and Init Systems](02-systemd-init.md) - Modern service management
- [03. The Boot Process](03-boot-process.md) - From BIOS/UEFI to login prompt
- [04. Device Management](04-device-management.md) - udev, sysfs, and hardware abstraction
- [05. Kernel Modules](05-kernel-modules.md) - Dynamic kernel extensions

### Section 2: Advanced File Systems & Storage
- [06. Filesystem Internals](06-filesystem-internals.md) - ext4, XFS, and VFS layer
- [07. LVM and RAID](07-lvm-raid.md) - Flexible storage management
- [08. Modern Filesystems: Btrfs & ZFS](08-btrfs-zfs.md) - Next-generation storage
- [09. Disk I/O Optimization](09-disk-optimization.md) - Performance tuning and monitoring
- [10. Quotas and ACLs](10-quotas-acls.md) - Advanced permissions and limits
- [11. FUSE Filesystems](11-fuse-filesystems.md) - Userspace filesystem development

### Section 3: Process Management & Performance
- [12. Process Scheduling](12-process-scheduling.md) - CFS, real-time, and priority
- [13. Memory Management](13-memory-management.md) - Virtual memory, swap, and OOM
- [14. Cgroups and Namespaces](14-cgroups-namespaces.md) - Resource control and isolation
- [15. Performance Tuning](15-performance-tuning.md) - Kernel parameters and optimization
- [16. Debugging and Tracing](16-debugging-tracing.md) - perf, ftrace, and eBPF

### Section 4: Networking Deep Dive
- [17. Linux Network Stack](17-network-stack.md) - From packets to sockets
- [18. Iptables and Nftables](18-iptables-nftables.md) - Advanced firewall configuration
- [19. Network Namespaces](19-network-namespaces.md) - Network isolation and virtualization
- [20. VPN and Tunneling](20-vpn-tunneling.md) - OpenVPN, WireGuard, and IPSec
- [21. Traffic Shaping](21-traffic-shaping.md) - QoS and bandwidth management
- [22. Network Troubleshooting](22-network-troubleshooting.md) - Advanced diagnostics

### Section 5: Security & Hardening
- [23. SELinux and AppArmor](23-selinux-apparmor.md) - Mandatory access control
- [24. PAM Authentication](24-pam-authentication.md) - Flexible authentication framework
- [25. Linux Audit Framework](25-audit-framework.md) - System auditing and compliance
- [26. System Hardening](26-hardening-practices.md) - Best practices and tools
- [27. Cryptography Tools](27-cryptography-tools.md) - LUKS, GPG, and certificates

### Section 6: Advanced Topics & Automation
- [28. Container Internals](28-containers-internals.md) - How containers really work
- [29. Automation with Ansible](29-automation-ansible.md) - Infrastructure as code
- [30. Building Custom Kernels](30-custom-kernels.md) - Kernel compilation and patches
- [31. Embedded Linux](31-embedded-linux.md) - Minimal systems and IoT
- [32. Troubleshooting Methodology](32-troubleshooting-methodology.md) - Systematic problem solving

## Study Approach

### For Each Lesson:
1. **Read** the theory and concepts
2. **Practice** with the provided examples
3. **Complete** the hands-on exercises
4. **Apply** knowledge in real scenarios
5. **Document** your learnings

### Recommended Schedule:
- **Week 1-2**: System Architecture (Lessons 1-5)
- **Week 3-4**: File Systems & Storage (Lessons 6-11)
- **Week 5-6**: Process Management (Lessons 12-16)
- **Week 7-8**: Networking (Lessons 17-22)
- **Week 9-10**: Security (Lessons 23-27)
- **Week 11-12**: Advanced Topics (Lessons 28-32)

## Lab Environment Setup

Create a dedicated lab environment:
```bash
# Install essential tools
sudo apt update
sudo apt install -y build-essential git vim tmux htop iotop sysstat strace ltrace

# Create lab directory
mkdir -p ~/linux-lab/{scripts,configs,logs}
cd ~/linux-lab
```

## Additional Resources

### Books:
- "Linux Kernel Development" by Robert Love
- "Understanding the Linux Kernel" by Bovet & Cesati
- "Linux System Programming" by Robert Love

### Online:
- [Kernel Documentation](https://www.kernel.org/doc/html/latest/)
- [Linux From Scratch](http://www.linuxfromscratch.org/)
- [Arch Wiki](https://wiki.archlinux.org/)

## Progress Tracking

Use this checklist to track your progress:

- [ ] Section 1: System Architecture
- [ ] Section 2: File Systems
- [ ] Section 3: Process Management
- [ ] Section 4: Networking
- [ ] Section 5: Security
- [ ] Section 6: Advanced Topics

---

*Remember: The goal is not just to learn commands, but to understand the underlying systems and principles that make Linux powerful.*