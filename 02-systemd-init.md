# Lesson 02: Systemd and Init Systems

## Overview
Systemd is the modern init system and service manager for Linux, replacing traditional SysV init. It manages the boot process, services, and system state with advanced features like parallel startup, on-demand activation, and dependency management.

## Learning Objectives
- Understand init systems evolution
- Master systemd architecture and concepts
- Manage services with systemctl
- Create custom systemd units
- Debug boot and service issues

## Evolution of Init Systems

### Traditional Init (SysV)
```
1. Kernel loads /sbin/init (PID 1)
2. Init reads /etc/inittab
3. Executes rc scripts sequentially
4. Runlevels (0-6) control system state
```

### Systemd Architecture
```
systemd (PID 1)
├── System and Service Manager
├── Journal Logging (journald)
├── Network Management (networkd)
├── DNS Resolution (resolved)
├── Login Management (logind)
└── Time Sync (timesyncd)
```

## Systemd Concepts

### Units
The basic building blocks of systemd:

- **Service units** (.service) - Daemons and processes
- **Socket units** (.socket) - IPC and network sockets
- **Target units** (.target) - Grouping and synchronization
- **Mount units** (.mount) - Filesystem mount points
- **Timer units** (.timer) - Scheduled execution
- **Path units** (.path) - File system path monitoring
- **Device units** (.device) - Device availability
- **Slice units** (.slice) - Resource management groups

### Unit File Locations
```bash
/etc/systemd/system/    # Administrator units (highest priority)
/run/systemd/system/    # Runtime units
/lib/systemd/system/    # Distribution units
/usr/lib/systemd/system/ # Package units

# View unit search path
systemctl -p UnitPath show
```

## Managing Services with systemctl

### Basic Service Operations
```bash
# Start/stop/restart services
sudo systemctl start nginx.service
sudo systemctl stop nginx.service
sudo systemctl restart nginx.service
sudo systemctl reload nginx.service  # Reload config without restart

# Enable/disable at boot
sudo systemctl enable nginx.service
sudo systemctl disable nginx.service

# Check status
systemctl status nginx.service
systemctl is-active nginx.service
systemctl is-enabled nginx.service

# List all services
systemctl list-units --type=service
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service
```

### Advanced Operations
```bash
# Show service dependencies
systemctl list-dependencies nginx.service

# Show reverse dependencies
systemctl list-dependencies --reverse nginx.service

# Mask/unmask services (prevent starting)
sudo systemctl mask bluetooth.service
sudo systemctl unmask bluetooth.service

# Edit service files
sudo systemctl edit nginx.service  # Creates override
sudo systemctl edit --full nginx.service  # Edit full unit

# Reload systemd configuration
sudo systemctl daemon-reload
```

## Creating Custom Service Units

### Basic Service Unit
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Service
Documentation=https://myapp.example.com/docs
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5s
User=myapp
Group=myapp
WorkingDirectory=/var/lib/myapp
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# Security hardening
PrivateTmp=true
NoNewPrivileges=true
ReadOnlyPaths=/
ReadWritePaths=/var/lib/myapp /var/log/myapp

[Install]
WantedBy=multi-user.target
```

### Service Types
```ini
# Type=simple (default)
ExecStart=/usr/bin/myapp  # Main process

# Type=forking
ExecStart=/usr/bin/myapp --daemon
PIDFile=/var/run/myapp.pid

# Type=oneshot
Type=oneshot
ExecStart=/usr/bin/setup-script
RemainAfterExit=yes

# Type=notify
Type=notify
ExecStart=/usr/bin/myapp  # Sends READY=1 to systemd

# Type=dbus
Type=dbus
BusName=com.example.MyApp
ExecStart=/usr/bin/myapp
```

## Timer Units (Cron Alternative)

### Creating a Timer
```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer
Documentation=man:backup(8)

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
```

### Associated Service
```ini
# /etc/systemd/system/backup.service
[Unit]
Description=System Backup
Documentation=man:backup(8)

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
StandardOutput=journal
StandardError=journal
```

### Timer Management
```bash
# Enable and start timer
sudo systemctl enable --now backup.timer

# List all timers
systemctl list-timers --all

# Check next execution
systemctl status backup.timer
```

## Socket Activation

### Socket Unit
```ini
# /etc/systemd/system/myapp.socket
[Unit]
Description=MyApp Socket
Documentation=man:myapp(8)

[Socket]
ListenStream=8080
Accept=false

[Install]
WantedBy=sockets.target
```

### Service Triggered by Socket
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=MyApp Service
Documentation=man:myapp(8)
Requires=myapp.socket

[Service]
Type=simple
ExecStart=/usr/bin/myapp
StandardInput=socket
StandardOutput=journal
StandardError=journal

[Install]
Also=myapp.socket
```

## Targets (Runlevel Replacement)

### Common Targets
```bash
# View current target
systemctl get-default

# Set default target
sudo systemctl set-default multi-user.target

# Switch to target
sudo systemctl isolate multi-user.target

# Common targets:
# poweroff.target    - Halt
# rescue.target      - Single user
# multi-user.target  - Multi-user, no GUI
# graphical.target   - Multi-user with GUI
# reboot.target      - Reboot
# emergency.target   - Emergency shell
```

### Custom Target
```ini
# /etc/systemd/system/myapp.target
[Unit]
Description=My Application Stack
Documentation=https://myapp.example.com/
Requires=multi-user.target
After=multi-user.target
AllowIsolate=yes

[Install]
WantedBy=multi-user.target
```

## Boot Process Analysis

### Boot Performance
```bash
# Analyze boot time
systemd-analyze

# Detailed breakdown
systemd-analyze blame

# Critical chain
systemd-analyze critical-chain

# Plot boot chart
systemd-analyze plot > boot.svg

# Verify unit files
systemd-analyze verify
```

### Boot Debugging
```bash
# Enable debug shell on tty9
sudo systemctl enable debug-shell.service

# Boot with debug options
# Add to kernel command line:
systemd.log_level=debug
systemd.log_target=console

# Check failed services
systemctl --failed

# List jobs
systemctl list-jobs
```

## Journald - System Logging

### Basic Journal Usage
```bash
# View all logs
journalctl

# Follow logs (like tail -f)
journalctl -f

# Show kernel messages
journalctl -k

# Show logs from current boot
journalctl -b

# Show logs from previous boot
journalctl -b -1

# Filter by time
journalctl --since "2024-01-01" --until "2024-01-02"
journalctl --since "1 hour ago"
```

### Advanced Filtering
```bash
# By unit
journalctl -u nginx.service

# By priority
journalctl -p err  # error and above
journalctl -p warning..err  # warning to error

# By executable
journalctl /usr/bin/bash

# By PID
journalctl _PID=1234

# Output formats
journalctl -o json-pretty
journalctl -o cat  # No metadata
```

### Journal Management
```bash
# Disk usage
journalctl --disk-usage

# Vacuum logs
sudo journalctl --vacuum-time=2weeks
sudo journalctl --vacuum-size=500M

# Verify journal
journalctl --verify

# Journal configuration
sudo nano /etc/systemd/journald.conf
```

## Resource Control with Systemd

### Resource Limits
```ini
[Service]
# CPU
CPUQuota=50%
CPUWeight=100

# Memory
MemoryMax=2G
MemoryHigh=1G

# IO
IOWeight=100
IOReadBandwidthMax=/dev/sda 10M

# Tasks
TasksMax=100
```

### Using systemd-run
```bash
# Run with resource limits
systemd-run --uid=1000 --gid=1000 --setenv=HOME=/home/user \
    --property=MemoryMax=500M \
    --property=CPUQuota=25% \
    mycommand

# Create transient service
systemd-run --uid=1000 --property=Type=forking \
    --property=RemainAfterExit=yes \
    --unit=myapp.service \
    /usr/bin/myapp --daemon
```

## Practical Exercises

### Exercise 1: Create a Web Application Service
```bash
# Create a Python web app
cat > /tmp/webapp.py << 'EOF'
#!/usr/bin/env python3
from http.server import HTTPServer, BaseHTTPRequestHandler
import signal
import sys

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from systemd service!\n")

def signal_handler(sig, frame):
    print('Shutting down gracefully...')
    sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)

httpd = HTTPServer(('', 8888), Handler)
print('Starting server on port 8888...')
httpd.serve_forever()
EOF

chmod +x /tmp/webapp.py

# Create service unit
sudo tee /etc/systemd/system/webapp.service << EOF
[Unit]
Description=Python Web Application
After=network.target

[Service]
Type=simple
ExecStart=/tmp/webapp.py
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Start and test
sudo systemctl daemon-reload
sudo systemctl start webapp.service
curl localhost:8888
sudo systemctl status webapp.service
```

### Exercise 2: Create a Timer for Cleanup
```bash
# Create cleanup script
sudo tee /usr/local/bin/cleanup.sh << 'EOF'
#!/bin/bash
echo "Cleaning temporary files at $(date)"
find /tmp -type f -name "*.tmp" -mtime +7 -delete
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;
EOF

sudo chmod +x /usr/local/bin/cleanup.sh

# Create timer unit
sudo tee /etc/systemd/system/cleanup.timer << EOF
[Unit]
Description=Weekly Cleanup Timer

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Create service unit
sudo tee /etc/systemd/system/cleanup.service << EOF
[Unit]
Description=System Cleanup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cleanup.sh
EOF

# Enable timer
sudo systemctl daemon-reload
sudo systemctl enable --now cleanup.timer
systemctl list-timers
```

### Exercise 3: Socket Activation
```bash
# Create echo server
cat > /tmp/echo-server.py << 'EOF'
#!/usr/bin/env python3
import sys
import socket

# Get socket from systemd
sock = socket.fromfd(3, socket.AF_INET, socket.SOCK_STREAM)

while True:
    conn, addr = sock.accept()
    data = conn.recv(1024)
    conn.send(b"Echo: " + data)
    conn.close()
EOF

chmod +x /tmp/echo-server.py

# Create socket unit
sudo tee /etc/systemd/system/echo.socket << EOF
[Unit]
Description=Echo Socket

[Socket]
ListenStream=9999
Accept=false

[Install]
WantedBy=sockets.target
EOF

# Create service unit
sudo tee /etc/systemd/system/echo.service << EOF
[Unit]
Description=Echo Service
Requires=echo.socket

[Service]
Type=simple
ExecStart=/tmp/echo-server.py
StandardInput=socket

[Install]
Also=echo.socket
EOF

# Start socket
sudo systemctl daemon-reload
sudo systemctl start echo.socket

# Test
echo "Hello" | nc localhost 9999
```

### Exercise 4: Debug Boot Issues
```bash
# Analyze boot time
systemd-analyze
systemd-analyze blame | head -20
systemd-analyze critical-chain

# Create problematic service
sudo tee /etc/systemd/system/slow.service << EOF
[Unit]
Description=Slow Starting Service
Before=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sleep 30
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# Enable and reboot to see impact
# sudo systemctl enable slow.service
# After reboot, analyze:
# systemd-analyze blame | grep slow
```

## Troubleshooting Guide

### Common Issues and Solutions

1. **Service won't start**
```bash
# Check status and logs
systemctl status myservice
journalctl -xeu myservice

# Verify unit file
systemd-analyze verify myservice.service

# Check dependencies
systemctl list-dependencies myservice
```

2. **Service starts but crashes**
```bash
# Enable core dumps
echo "kernel.core_pattern=|/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h %e" | \
    sudo tee /etc/sysctl.d/50-coredump.conf
sudo sysctl -p /etc/sysctl.d/50-coredump.conf

# List core dumps
coredumpctl list

# Analyze core dump
coredumpctl info
coredumpctl gdb
```

3. **Boot hangs**
```bash
# Boot with debug
# Add to kernel parameters:
systemd.log_level=debug systemd.log_target=kmsg

# Emergency mode
# Add to kernel parameters:
systemd.unit=emergency.target

# Rescue mode
systemd.unit=rescue.target
```

## Best Practices

1. **Service Design**
   - Use Type=notify when possible
   - Implement proper signal handling
   - Don't use Type=forking unless necessary
   - Use systemd's built-in features instead of shell scripts

2. **Security Hardening**
   - Run services as non-root users
   - Use PrivateTmp=true
   - Implement sandboxing with namespace options
   - Limit file system access with ReadOnlyPaths

3. **Logging**
   - Use StandardOutput=journal
   - Structure logs with systemd.journal fields
   - Set appropriate SyslogIdentifier

4. **Dependencies**
   - Use After= for ordering
   - Use Requires= sparingly
   - Prefer Wants= for loose coupling
   - Consider BindsTo= for strict binding

## Further Reading

- [systemd Documentation](https://www.freedesktop.org/wiki/Software/systemd/)
- [systemd by Example](https://systemd-by-example.com/)
- [The systemd for Administrators Series](https://0pointer.de/blog/projects/systemd-for-admins-1.html)
- [systemd Cookbook](https://systemd.io/)

## Next Lesson
[03. The Boot Process](03-boot-process.md) - Understanding Linux boot from BIOS/UEFI to login prompt.