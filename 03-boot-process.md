# Lesson 03: The Boot Process

## Overview
Understanding the Linux boot process is essential for troubleshooting, optimization, and system recovery. This lesson covers the complete boot sequence from power-on to user login, including BIOS/UEFI, bootloaders, kernel initialization, and userspace startup.

## Learning Objectives
- Understand each stage of the boot process
- Master GRUB bootloader configuration
- Debug boot issues effectively
- Optimize boot performance
- Create custom initramfs images

## Boot Process Overview

```
Power On
    ↓
BIOS/UEFI Firmware
    ↓
Bootloader (GRUB)
    ↓
Kernel Loading
    ↓
Initramfs/Initrd
    ↓
Real Root Mount
    ↓
Init System (systemd)
    ↓
User Login
```

## Stage 1: BIOS/UEFI Firmware

### BIOS (Legacy)
```
1. Power-On Self Test (POST)
2. Hardware initialization
3. Boot device selection
4. Load MBR (first 512 bytes)
5. Execute bootloader code
```

### UEFI (Modern)
```
1. SEC (Security) phase
2. PEI (Pre-EFI Initialization)
3. DXE (Driver Execution Environment)
4. BDS (Boot Device Selection)
5. Load EFI bootloader from ESP
```

### Checking System Firmware
```bash
# Check if system uses UEFI
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"

# View UEFI variables
efibootmgr -v

# List EFI boot entries
sudo efibootmgr

# View current boot order
sudo efibootmgr | grep BootOrder

# UEFI firmware information
sudo dmidecode -t bios
```

### EFI System Partition (ESP)
```bash
# Locate ESP
lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT | grep -E "vfat|EFI"

# Mount and explore ESP
sudo mount /boot/efi
ls -la /boot/efi/EFI/

# Typical ESP structure:
/boot/efi/
├── EFI/
│   ├── ubuntu/
│   │   ├── grubx64.efi
│   │   ├── shimx64.efi
│   │   └── mmx64.efi
│   ├── BOOT/
│   │   └── BOOTX64.EFI
│   └── Microsoft/
│       └── Boot/
```

## Stage 2: Bootloader (GRUB2)

### GRUB Configuration
```bash
# Main configuration file
/boot/grub/grub.cfg  # Generated - don't edit directly

# Source configuration
/etc/default/grub    # User settings
/etc/grub.d/         # Configuration scripts

# Update GRUB configuration
sudo update-grub
# or
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Key GRUB Settings
```bash
# /etc/default/grub
GRUB_DEFAULT=0                    # Default menu entry
GRUB_TIMEOUT=10                   # Menu timeout
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"  # Kernel parameters
GRUB_CMDLINE_LINUX=""            # Always applied parameters
GRUB_GFXMODE=1920x1080          # Resolution
GRUB_DISABLE_RECOVERY="false"    # Recovery mode entries
```

### GRUB Command Line
```bash
# At boot, press 'c' for GRUB command line

# List devices
grub> ls
(hd0) (hd0,msdos1) (hd0,msdos2)

# Show partitions
grub> ls (hd0,1)/

# Set root and boot manually
grub> set root=(hd0,1)
grub> linux /vmlinuz root=/dev/sda2
grub> initrd /initrd.img
grub> boot

# Find files
grub> search -f /vmlinuz
```

### Creating Custom GRUB Entries
```bash
# Create custom menu entry
sudo tee /etc/grub.d/40_custom << 'EOF'
#!/bin/sh
exec tail -n +3 $0

menuentry 'Ubuntu (Custom Kernel)' {
    load_video
    insmod gzio
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root YOUR-ROOT-UUID
    linux /boot/vmlinuz-custom root=UUID=YOUR-ROOT-UUID ro quiet
    initrd /boot/initrd.img-custom
}
EOF

sudo chmod +x /etc/grub.d/40_custom
sudo update-grub
```

## Stage 3: Kernel Loading

### Kernel Boot Parameters
```bash
# View current kernel command line
cat /proc/cmdline

# Common parameters:
root=UUID=xxx     # Root filesystem
ro               # Mount root read-only initially
quiet            # Reduce boot messages
splash           # Show boot splash
nomodeset        # Disable kernel mode setting
init=/bin/bash   # Emergency shell
single           # Single user mode
```

### Early Boot Messages
```bash
# View kernel ring buffer
dmesg | less

# Filter by timestamp
dmesg -T

# Watch live kernel messages
dmesg -w

# Early boot messages (if preserved)
journalctl -b -k | head -100
```

## Stage 4: Initramfs/Initrd

### Understanding Initramfs
Initial RAM filesystem containing drivers and tools needed to mount the real root filesystem.

```bash
# List initramfs contents
lsinitramfs /boot/initrd.img-$(uname -r)

# Extract initramfs
mkdir /tmp/initramfs
cd /tmp/initramfs
zcat /boot/initrd.img-$(uname -r) | cpio -id

# Explore structure
ls -la
tree -d -L 2
```

### Creating Custom Initramfs
```bash
# Update existing initramfs
sudo update-initramfs -u

# Create new initramfs for current kernel
sudo update-initramfs -c -k $(uname -r)

# Update all kernels
sudo update-initramfs -u -k all

# Include extra modules
echo "module_name" | sudo tee -a /etc/initramfs-tools/modules
sudo update-initramfs -u
```

### Initramfs Hooks and Scripts
```bash
# Hook script example
sudo tee /etc/initramfs-tools/hooks/custom << 'EOF'
#!/bin/sh
PREREQ=""
prereqs() {
    echo "$PREREQ"
}
case $1 in
prereqs)
    prereqs
    exit 0
    ;;
esac

. /usr/share/initramfs-tools/hook-functions

# Copy binary
copy_exec /usr/bin/myprogram /bin

# Copy library
copy_lib /usr/lib/x86_64-linux-gnu/libexample.so

# Add module
manual_add_modules ext4
EOF

sudo chmod +x /etc/initramfs-tools/hooks/custom
```

### Boot Script
```bash
# Custom boot script
sudo tee /etc/initramfs-tools/scripts/init-premount/custom << 'EOF'
#!/bin/sh
PREREQ=""
prereqs() {
    echo "$PREREQ"
}
case $1 in
prereqs)
    prereqs
    exit 0
    ;;
esac

# Custom initialization code
echo "Running custom boot script..."
# Add your code here
EOF

sudo chmod +x /etc/initramfs-tools/scripts/init-premount/custom
```

## Stage 5: Root Filesystem Mount

### Mount Process
```bash
# View mount order
findmnt --tree

# Check filesystem
sudo fsck -n /dev/sda1

# Mount options in /etc/fstab
cat /etc/fstab

# Emergency mount
mount -o remount,rw /
```

### Switching Root
```bash
# The pivot_root process (simplified)
# 1. Mount new root
mount /dev/sda2 /newroot

# 2. Move special filesystems
mount --move /dev /newroot/dev
mount --move /proc /newroot/proc
mount --move /sys /newroot/sys

# 3. Switch root
exec switch_root /newroot /sbin/init
```

## Stage 6: Init System Startup

### Systemd Boot Sequence
```bash
# View boot timeline
systemd-analyze plot > boot.svg
firefox boot.svg

# Critical chain
systemd-analyze critical-chain

# List boot targets
systemctl list-units --type=target

# Boot target dependencies
systemctl list-dependencies default.target
```

### Early Userspace Services
```bash
# Essential early services
systemctl status systemd-journald
systemctl status systemd-udevd
systemctl status systemd-tmpfiles-setup

# Network initialization
systemctl status systemd-networkd
systemctl status NetworkManager
```

## Boot Debugging Techniques

### Kernel Debugging Parameters
```bash
# Remove quiet and splash
linux /vmlinuz root=/dev/sda2 ro

# Enable verbose messages
linux /vmlinuz root=/dev/sda2 ro debug

# Break at initramfs
linux /vmlinuz root=/dev/sda2 ro break

# Enable systemd debugging
linux /vmlinuz root=/dev/sda2 ro systemd.log_level=debug

# Emergency shell
linux /vmlinuz root=/dev/sda2 ro init=/bin/bash
```

### Recovery Techniques
```bash
# Boot to recovery mode
# Select "Advanced options" → "Recovery mode" in GRUB

# Manual recovery boot
# 1. Boot with init=/bin/bash
# 2. Remount root read-write
mount -o remount,rw /

# 3. Fix issues
# Reset root password
passwd root

# Fix broken packages
dpkg --configure -a
apt-get update
apt-get -f install

# 4. Sync and reboot
sync
reboot -f
```

### Rescue Initramfs
```bash
# Break to initramfs shell
# Add 'break' to kernel parameters

# In initramfs shell:
(initramfs) help
(initramfs) ls /dev/
(initramfs) cat /proc/cmdline
(initramfs) modprobe module_name
(initramfs) mount -t ext4 /dev/sda2 /root
(initramfs) exit  # Continue boot
```

## Boot Performance Optimization

### Measuring Boot Time
```bash
# Overall boot time
systemd-analyze

# Per-service breakdown
systemd-analyze blame

# Critical path
systemd-analyze critical-chain

# Detailed timing
journalctl -b -o short-monotonic
```

### Optimization Techniques
```bash
# 1. Disable unnecessary services
systemctl disable bluetooth
systemctl disable cups

# 2. Use systemd-readahead (if available)
sudo systemctl enable systemd-readahead-collect
sudo systemctl enable systemd-readahead-replay

# 3. Optimize filesystem checks
# In /etc/fstab, set fsck order (0 = no check)
UUID=xxx / ext4 errors=remount-ro 0 1

# 4. Configure GRUB timeout
sudo sed -i 's/GRUB_TIMEOUT=10/GRUB_TIMEOUT=1/' /etc/default/grub
sudo update-grub

# 5. Remove unnecessary kernel modules
echo "blacklist module_name" | sudo tee /etc/modprobe.d/blacklist-custom.conf
```

### Creating Minimal Boot
```bash
# Create custom target
sudo tee /etc/systemd/system/minimal.target << EOF
[Unit]
Description=Minimal Boot Target
Requires=basic.target
Conflicts=rescue.service rescue.target
After=basic.target rescue.service rescue.target
AllowIsolate=yes

[Install]
Alias=default.target
EOF

# Set as default
sudo systemctl set-default minimal.target
```

## Practical Exercises

### Exercise 1: Boot Process Investigation
```bash
# 1. Record boot time
systemd-analyze

# 2. Create boot chart
systemd-analyze plot > ~/boot-chart.svg

# 3. Identify slowest services
systemd-analyze blame | head -20

# 4. Check critical chain
systemd-analyze critical-chain

# 5. Review boot messages
journalctl -b | grep -E "(Started|Starting|Reached)"
```

### Exercise 2: Custom GRUB Entry
```bash
# 1. Find your root partition UUID
lsblk -f
blkid

# 2. Create debug boot entry
sudo tee -a /etc/grub.d/40_custom << EOF

menuentry 'Ubuntu (Debug Mode)' {
    insmod gzio
    insmod part_gpt
    insmod ext2
    search --no-floppy --fs-uuid --set=root $(findmnt -no UUID /)
    linux /boot/vmlinuz-$(uname -r) root=UUID=$(findmnt -no UUID /) ro debug systemd.log_level=debug
    initrd /boot/initrd.img-$(uname -r)
}
EOF

# 3. Update GRUB
sudo update-grub

# 4. Reboot and test
```

### Exercise 3: Initramfs Customization
```bash
# 1. Create custom hook
sudo tee /etc/initramfs-tools/hooks/banner << 'EOF'
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case $1 in
prereqs) prereqs; exit 0 ;;
esac

. /usr/share/initramfs-tools/hook-functions

# Create banner file
cat > ${DESTDIR}/etc/banner << BANNER
*********************************
*  Custom Initramfs Boot Banner *
*  System: $(hostname)         *
*  Date: $(date)               *
*********************************
BANNER
EOF

sudo chmod +x /etc/initramfs-tools/hooks/banner

# 2. Create boot script
sudo tee /etc/initramfs-tools/scripts/init-top/show_banner << 'EOF'
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case $1 in
prereqs) prereqs; exit 0 ;;
esac

[ -f /etc/banner ] && cat /etc/banner
EOF

sudo chmod +x /etc/initramfs-tools/scripts/init-top/show_banner

# 3. Update initramfs
sudo update-initramfs -u

# 4. Reboot and observe
```

### Exercise 4: Boot Recovery Practice
```bash
# 1. Create backup
sudo cp /etc/fstab /etc/fstab.backup

# 2. Intentionally break fstab (DO THIS IN A VM!)
# sudo sed -i 's/UUID=/UUID=broken/' /etc/fstab

# 3. Reboot and fix:
# - Boot to recovery mode
# - Select "Root shell"
# - mount -o remount,rw /
# - cp /etc/fstab.backup /etc/fstab
# - reboot

# 4. Alternative: Boot with init=/bin/bash
# - Edit GRUB entry
# - Add init=/bin/bash to kernel line
# - Boot and fix
```

## Advanced Topics

### Secure Boot
```bash
# Check Secure Boot status
mokutil --sb-state

# List enrolled keys
mokutil --list-enrolled

# Sign custom kernel
sbsign --key MOK.key --cert MOK.crt \
    --output vmlinuz-signed vmlinuz
```

### Network Boot (PXE)
```bash
# DHCP configuration for PXE
# /etc/dhcp/dhcpd.conf
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    next-server 192.168.1.10;
    filename "pxelinux.0";
}

# TFTP server setup
sudo apt install tftpd-hpa
sudo systemctl enable tftpd-hpa
```

### Custom Init System
```c
// minimal_init.c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    printf("Minimal init starting...\n");
    
    // Mount essential filesystems
    system("mount -t proc proc /proc");
    system("mount -t sysfs sys /sys");
    system("mount -t devtmpfs dev /dev");
    
    // Start shell
    if (fork() == 0) {
        execl("/bin/bash", "bash", NULL);
    }
    
    // Reap zombies
    while (1) {
        wait(NULL);
    }
}
```

## Troubleshooting Common Boot Issues

### GRUB Issues
```bash
# Reinstall GRUB (from live USB)
sudo mount /dev/sda2 /mnt
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt
grub-install /dev/sda
update-grub
```

### Kernel Panic
```
Common causes:
1. Missing initramfs
2. Wrong root= parameter
3. Missing drivers
4. Corrupt kernel

Recovery:
1. Boot older kernel
2. Rebuild initramfs
3. Check disk for errors
```

### Emergency Boot
```bash
# Single user mode
systemctl rescue

# Emergency mode (minimal)
systemctl emergency

# From GRUB:
# Add 'single' or 'emergency' to kernel line
```

## Best Practices

1. **Always keep a working kernel**
   - Don't remove old kernels immediately
   - Test new kernels before removing old ones

2. **Document boot configuration**
   - Keep notes on custom parameters
   - Document hardware-specific settings

3. **Regular backups**
   - Backup /boot partition
   - Keep bootable USB ready

4. **Monitor boot health**
   - Check systemd-analyze regularly
   - Review failed services

## Further Reading

- [UEFI Specification](https://uefi.org/specifications)
- [GRUB Manual](https://www.gnu.org/software/grub/manual/)
- [Linux Boot Process](https://www.ibm.com/developerworks/library/l-linuxboot/)
- [systemd Boot Process](https://freedesktop.org/wiki/Software/systemd/Bootup/)

## Next Lesson
[04. Device Management](04-device-management.md) - Understanding udev, sysfs, and hardware abstraction in Linux.