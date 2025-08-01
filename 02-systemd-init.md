# Lesson 02: Systemd and Init Systems

## Overview
Systemd is the modern init system and service manager for Linux, replacing traditional SysV init. It manages the boot process, services, and system state with advanced features like parallel startup, on-demand activation, and dependency management.

## Learning Approach: The STAR Method
This lesson uses the STAR (Situation, Task, Action, Result) method to teach systemd concepts through real-world production scenarios. Each topic demonstrates practical problem-solving approaches used by systems engineers daily.

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

## STAR Scenarios for Service Management

### STAR Scenario 1: Web Server Emergency Recovery

**Situation**: E-commerce website went down during flash sale. Nginx service crashed and won't restart automatically. Customer support receiving 500+ calls/minute about site unavailability. Revenue loss: $10K/minute.

**Task**: Restore web service immediately and implement safeguards to prevent recurrence without disrupting the ongoing sale.

**Action**:
```bash
# Step 1: Quick assessment of service status
systemctl status nginx.service
# service failed with "Job for nginx.service failed"

# Step 2: Check immediate error details
journalctl -xeu nginx.service --no-pager -l
# Found: "nginx: [emerg] bind() to 0.0.0.0:443 failed (98: Address already in use)"

# Step 3: Identify port conflict
sudo ss -tlnp | grep :443
# Found rogue process binding to port 443

# Step 4: Kill conflicting process and restart nginx
sudo kill -9 $(sudo lsof -t -i:443)
sudo systemctl start nginx.service

# Step 5: Verify service is running and serving requests
systemctl is-active nginx.service
curl -I http://localhost
# HTTP/1.1 200 OK

# Step 6: Implement monitoring to prevent future conflicts
sudo systemctl edit nginx.service  # Create override
# Add: ExecStartPre=/bin/bash -c 'while ss -tlnp | grep :443 > /dev/null; do sleep 1; done'
```

**Result**: Website restored in 3 minutes. Prevented $30K revenue loss. Implemented pre-start port conflict detection. Zero similar incidents in following 6 months.

### STAR Scenario 2: Service Dependency Chain Failure

**Situation**: Database application stack failing to start after server reboot. Redis, PostgreSQL, and application services have complex interdependencies. Manual startup taking 45 minutes, missing SLA commitments.

**Task**: Analyze and resolve service startup order issues to achieve reliable automatic startup within 5 minutes.

**Action**:
```bash
# Step 1: Identify failed services and their dependencies
systemctl --failed
systemctl list-dependencies --reverse postgresql.service
systemctl list-dependencies --reverse redis.service

# Step 2: Analyze startup timing and conflicts
systemd-analyze critical-chain
systemd-analyze blame | head -20

# Step 3: Check service definitions for proper ordering
systemctl cat myapp.service
# Missing: After=postgresql.service redis.service

# Step 4: Create service override with correct dependencies
sudo systemctl edit myapp.service
```
```ini
[Unit]
After=postgresql.service redis.service
Wants=postgresql.service redis.service

[Service]
ExecStartPre=/usr/local/bin/wait-for-dependencies.sh
```

```bash
# Step 5: Create dependency check script
sudo tee /usr/local/bin/wait-for-dependencies.sh << 'EOF'
#!/bin/bash
# Wait for PostgreSQL
while ! pg_isready -h localhost -p 5432; do sleep 1; done
# Wait for Redis
while ! redis-cli ping > /dev/null 2>&1; do sleep 1; done
echo "All dependencies ready"
EOF
sudo chmod +x /usr/local/bin/wait-for-dependencies.sh

# Step 6: Test new configuration
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
systemctl status myapp.service
```

**Result**: Stack now starts automatically in 2 minutes 30 seconds. Reduced deployment failures by 90%. SLA compliance restored to 99.9%.

### STAR Scenario 3: Resource-Intensive Service Management

**Situation**: Machine learning training service consuming all system resources, causing other critical services to become unresponsive. Need to balance ML workload without stopping training job.

**Task**: Implement resource controls to limit ML service impact while maintaining training performance.

**Action**:
```bash
# Step 1: Identify resource consumption
systemctl status ml-training.service
systemd-cgtop  # Real-time cgroup resource usage

# Step 2: Check current resource limits
systemctl show ml-training.service | grep -E "Memory|CPU|IO"

# Step 3: Create resource-constrained override
sudo systemctl edit ml-training.service
```
```ini
[Service]
# Limit to 70% CPU and 12GB RAM
CPUQuota=70%
MemoryMax=12G
MemoryHigh=10G

# Reduce I/O priority
IOSchedulingClass=3
IOSchedulingPriority=7
IOWeight=100

# Limit number of processes/threads
TasksMax=1000
```

```bash
# Step 4: Apply changes and monitor impact
sudo systemctl daemon-reload
sudo systemctl restart ml-training.service

# Step 5: Verify resource usage and system responsiveness
systemd-cgtop
# Monitor for 30 minutes
systemctl status nginx.service postgresql.service
```

**Result**: ML training continues with 15% performance reduction. Other services restored to normal response times. System load average dropped from 24 to 8.

### STAR Scenario 4: Service Security Hardening After Breach

**Situation**: Security audit revealed application service running with excessive privileges after minor security incident. Service has root access and can write anywhere on filesystem.

**Task**: Implement comprehensive service hardening without breaking application functionality.

**Action**:
```bash
# Step 1: Create dedicated service user
sudo useradd -r -s /bin/false -d /var/lib/myapp myapp
sudo mkdir -p /var/lib/myapp /var/log/myapp
sudo chown myapp:myapp /var/lib/myapp /var/log/myapp

# Step 2: Implement security hardening
sudo systemctl edit --full myapp.service
```
```ini
[Unit]
Description=My Application Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
User=myapp
Group=myapp
WorkingDirectory=/var/lib/myapp

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectHome=true
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp /var/log/myapp

# Network restrictions
RestrictAddressFamilies=AF_INET AF_INET6
IPAddressDeny=any
IPAddressAllow=127.0.0.1/8 10.0.0.0/8

# Capability restrictions
CapabilityBoundingSet=
AmbientCapabilities=

# System call restrictions
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM

[Install]
WantedBy=multi-user.target
```

```bash
# Step 3: Test application functionality
sudo systemctl daemon-reload
sudo systemctl restart myapp.service
systemctl status myapp.service

# Step 4: Verify security restrictions
systemd-analyze security myapp.service
```

**Result**: Service now runs with minimal privileges. Security score improved from 9.6 to 2.1. Application functionality preserved. Hardening template applied to 15 other services.

### STAR Scenario 5: Blue-Green Deployment with Service Management

**Situation**: Need to deploy new application version without downtime. Current deployment process requires 10-minute service interruption affecting 50K active users.

**Task**: Implement zero-downtime blue-green deployment using systemd service management.

**Action**:
```bash
# Step 1: Create blue service (current production)
sudo systemctl status myapp-blue.service
sudo systemctl is-active myapp-blue.service
# active

# Step 2: Prepare green service configuration
sudo cp /etc/systemd/system/myapp-blue.service /etc/systemd/system/myapp-green.service
sudo systemctl edit myapp-green.service
```
```ini
[Service]
Environment=APP_VERSION=v2.1.0
Environment=APP_PORT=8081
Environment=APP_CONFIG=/etc/myapp/green.conf
ExecStart=/usr/local/bin/myapp --port=8081 --config=/etc/myapp/green.conf
```

```bash
# Step 3: Deploy and start green service
sudo systemctl daemon-reload
sudo systemctl start myapp-green.service

# Step 4: Health check green deployment
curl -f http://localhost:8081/health
# Automated tests
/usr/local/bin/smoke-tests.sh --port=8081

# Step 5: Switch load balancer to green service
sudo systemctl reload nginx  # Reload with green upstream
sleep 30  # Allow connection draining

# Step 6: Stop blue service and prepare for next deployment
sudo systemctl stop myapp-blue.service
sudo mv /etc/systemd/system/myapp-blue.service /etc/systemd/system/myapp-blue.service.backup
sudo cp /etc/systemd/system/myapp-green.service /etc/systemd/system/myapp-blue.service
```

**Result**: Achieved zero-downtime deployment. Deployment time reduced from 10 minutes to 90 seconds. User experience uninterrupted. Process automated for daily releases.

## STAR Scenarios for Custom Service Units

### STAR Scenario 1: Legacy Application Systemd Migration

**Situation**: Legacy financial trading application running via SysV init scripts experiencing reliability issues. Application randomly dies during high-volume trading periods, requiring manual restarts costing $50K per incident.

**Task**: Migrate to systemd with robust restart policies and monitoring without disrupting trading operations.

**Action**:
```bash
# Step 1: Analyze current SysV init script
cat /etc/init.d/trading-app
# Application starts as daemon, no proper exit handling

# Step 2: Create systemd service unit
sudo tee /etc/systemd/system/trading-app.service << 'EOF'
[Unit]
Description=Financial Trading Application
Documentation=file:///opt/trading/docs/README.md
After=network-online.target postgresql.service
Wants=network-online.target
Requires=postgresql.service

[Service]
Type=notify
ExecStart=/opt/trading/bin/trading-app --systemd-notify
ExecReload=/bin/kill -USR1 $MAINPID
PIDFile=/var/run/trading-app.pid
Restart=always
RestartSec=10s
StartLimitInterval=300s
StartLimitBurst=5

# Resource limits for stability
MemoryMax=8G
CPUQuota=400%

# Security
User=trading
Group=trading
WorkingDirectory=/opt/trading
PrivateTmp=true
NoNewPrivileges=true

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=trading-app

[Install]
WantedBy=multi-user.target
EOF

# Step 3: Create dedicated user
sudo useradd -r -d /opt/trading -s /bin/false trading
sudo chown -R trading:trading /opt/trading

# Step 4: Test during low-volume period
sudo systemctl daemon-reload
sudo systemctl enable trading-app.service
sudo systemctl start trading-app.service

# Step 5: Monitor for 24 hours before full migration
journalctl -u trading-app.service -f
```

**Result**: Zero unplanned restarts in 6 months. Automated restart reduced incident response time from 15 minutes to 30 seconds. Saved $200K annually in downtime costs.

### STAR Scenario 2: Microservice Container-Alternative Architecture

**Situation**: Startup needs to deploy 12 microservices without Docker/Kubernetes complexity. Services need resource isolation, automatic restarts, and service discovery on single server.

**Task**: Create systemd-based microservice architecture with service templates and dependency management.

**Action**:
```bash
# Step 1: Create service template
sudo tee /etc/systemd/system/microservice@.service << 'EOF'
[Unit]
Description=Microservice %i
Documentation=file:///opt/microservices/%i/README.md
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/opt/microservices/%i/bin/start.sh
ExecStop=/opt/microservices/%i/bin/stop.sh
Restart=on-failure
RestartSec=5s

# Instance-specific user
User=micro-%i
Group=micro-%i
WorkingDirectory=/opt/microservices/%i

# Resource isolation per service
CPUQuota=50%
MemoryMax=512M
TasksMax=100

# Security isolation
PrivateTmp=true
PrivateDevices=true
ProtectSystem=strict
ReadWritePaths=/opt/microservices/%i/data /opt/microservices/%i/logs

# Networking (unique port per service)
Environment=SERVICE_NAME=%i
Environment=SERVICE_PORT=808%i

[Install]
WantedBy=multi-user.target
EOF

# Step 2: Create service discovery target
sudo tee /etc/systemd/system/microservices.target << 'EOF'
[Unit]
Description=All Microservices
Documentation=file:///opt/microservices/README.md
Wants=microservice@api.service microservice@auth.service microservice@billing.service
After=microservice@api.service microservice@auth.service microservice@billing.service

[Install]
WantedBy=multi-user.target
EOF

# Step 3: Setup individual services
for service in api auth billing; do
    sudo useradd -r -d /opt/microservices/$service -s /bin/false micro-$service
    sudo mkdir -p /opt/microservices/$service/{bin,data,logs}
    sudo chown -R micro-$service:micro-$service /opt/microservices/$service
    
    # Enable individual services
    sudo systemctl enable microservice@$service.service
done

# Step 4: Start all services as a group
sudo systemctl daemon-reload
sudo systemctl enable microservices.target
sudo systemctl start microservices.target
```

**Result**: Deployed 12 microservices with resource isolation. 40% lower memory usage than Docker. Service startup time: 2 seconds vs 30 seconds with containers.

### STAR Scenario 3: High-Availability Application with Health Checks

**Situation**: Critical payment processing service needs 99.99% uptime. Current service doesn't handle partial failures gracefully and takes 60 seconds to detect problems.

**Task**: Implement sophisticated health checking and automatic recovery with sub-10-second failure detection.

**Action**:
```bash
# Step 1: Create health check script
sudo tee /usr/local/bin/payment-health-check.sh << 'EOF'
#!/bin/bash
# Comprehensive health check for payment service

# Check process is responding
if ! curl -s --max-time 5 http://localhost:8080/health > /dev/null; then
    echo "HTTP health check failed"
    exit 1
fi

# Check database connectivity
if ! timeout 5 pg_isready -h localhost -p 5432 -U payments; then
    echo "Database connectivity failed"
    exit 1
fi

# Check memory usage (restart if over 90%)
MEM_USAGE=$(ps -o pid,pmem -p $MAINPID | awk 'NR==2 {print $2}' | cut -d. -f1)
if [[ $MEM_USAGE -gt 90 ]]; then
    echo "Memory usage too high: ${MEM_USAGE}%"
    exit 1
fi

echo "All health checks passed"
exit 0
EOF
sudo chmod +x /usr/local/bin/payment-health-check.sh

# Step 2: Create robust payment service
sudo tee /etc/systemd/system/payment-processor.service << 'EOF'
[Unit]
Description=Payment Processing Service
Documentation=https://payments.company.com/docs
After=network-online.target postgresql.service
Wants=network-online.target
Requires=postgresql.service

[Service]
Type=notify
ExecStart=/opt/payments/bin/payment-processor
ExecReload=/bin/kill -USR1 $MAINPID

# Health monitoring
ExecStartPost=/bin/sleep 10
ExecStartPost=/usr/local/bin/payment-health-check.sh
ExecReload=/usr/local/bin/payment-health-check.sh

# Restart policies
Restart=always
RestartSec=5s
StartLimitInterval=300s
StartLimitBurst=3

# Watchdog for runtime health
WatchdogSec=30s
NotifyAccess=main

# Resource limits
MemoryMax=2G
CPUQuota=200%

# Security
User=payments
Group=payments
WorkingDirectory=/opt/payments
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
EOF

# Step 3: Create companion health monitor service
sudo tee /etc/systemd/system/payment-health-monitor.service << 'EOF'
[Unit]
Description=Payment Health Monitor
After=payment-processor.service
Wants=payment-processor.service

[Service]
Type=simple
ExecStart=/usr/local/bin/continuous-health-monitor.sh
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

# Step 4: Create continuous monitoring script
sudo tee /usr/local/bin/continuous-health-monitor.sh << 'EOF'
#!/bin/bash
while true; do
    sleep 10
    if ! /usr/local/bin/payment-health-check.sh; then
        echo "Health check failed, restarting payment processor"
        systemctl restart payment-processor.service
        sleep 30  # Wait for restart
    fi
done
EOF
sudo chmod +x /usr/local/bin/continuous-health-monitor.sh

# Step 5: Enable and start services
sudo systemctl daemon-reload
sudo systemctl enable payment-processor.service payment-health-monitor.service
sudo systemctl start payment-processor.service payment-health-monitor.service
```

**Result**: Achieved 99.995% uptime. Failure detection time reduced from 60s to 10s. Automated recovery resolved 95% of issues without manual intervention.

## STAR Scenarios for Timers and Socket Activation

### STAR Scenario 1: Database Backup Automation Replacement

**Situation**: Company using unreliable cron jobs for database backups. Backups fail silently 15% of the time. No visibility into backup status. Critical data loss risk during system failures.

**Task**: Replace cron-based backup system with robust systemd timers providing monitoring, logging, and failure recovery.

**Action**:
```bash
# Step 1: Create comprehensive backup script with error handling
sudo tee /usr/local/bin/database-backup.sh << 'EOF'
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/database-backup.log"

# Logging function
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Perform backup with compression and verification
log "Starting database backup"
if pg_dumpall -h localhost -U postgres | gzip > "$BACKUP_DIR/backup_$DATE.sql.gz"; then
    # Verify backup integrity
    if gzip -t "$BACKUP_DIR/backup_$DATE.sql.gz"; then
        log "Backup completed successfully: backup_$DATE.sql.gz"
        # Cleanup old backups (keep 30 days)
        find "$BACKUP_DIR" -name "backup_*.sql.gz" -mtime +30 -delete
        log "Old backups cleaned up"
    else
        log "ERROR: Backup verification failed"
        exit 1
    fi
else
    log "ERROR: Backup creation failed"
    exit 1
fi

# Send notification on success
curl -X POST "https://hooks.slack.com/webhook" \
    -d "{\"text\": \"Database backup completed successfully: $DATE\"}" || true
EOF

sudo chmod +x /usr/local/bin/database-backup.sh

# Step 2: Create backup service unit
sudo tee /etc/systemd/system/database-backup.service << 'EOF'
[Unit]
Description=PostgreSQL Database Backup
Documentation=file:///usr/local/bin/database-backup.sh
After=postgresql.service
Wants=postgresql.service

[Service]
Type=oneshot
User=postgres
Group=postgres
ExecStart=/usr/local/bin/database-backup.sh
StandardOutput=journal
StandardError=journal
SyslogIdentifier=database-backup

# Security hardening
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/backups/postgresql /var/log

# Resource limits
MemoryMax=1G
CPUQuota=50%
TimeoutStartSec=3600

# Failure handling
OnFailure=backup-failure-notify.service
EOF

# Step 3: Create timer with smart scheduling
sudo tee /etc/systemd/system/database-backup.timer << 'EOF'
[Unit]
Description=Database Backup Timer
Documentation=file:///usr/local/bin/database-backup.sh

[Timer]
# Run daily at 2 AM with randomization to avoid system load spikes
OnCalendar=*-*-* 02:00:00
RandomizedDelaySec=1800
Persistent=true

# Ensure backup runs even if system was down
WakeSystem=false

[Install]
WantedBy=timers.target
EOF

# Step 4: Create failure notification service
sudo tee /etc/systemd/system/backup-failure-notify.service << 'EOF'
[Unit]
Description=Backup Failure Notification

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'curl -X POST "https://hooks.slack.com/webhook" -d "{\"text\": \"⚠️ Database backup failed on $(hostname) at $(date)\"}"'
EOF

# Step 5: Enable and test the timer system
sudo systemctl daemon-reload
sudo systemctl enable --now database-backup.timer
sudo systemctl list-timers database-backup.timer

# Test backup manually
sudo systemctl start database-backup.service
```

**Result**: Backup reliability increased to 99.8%. Automated failure notifications reduced response time from hours to minutes. Persistent timers ensure backups run even after system downtime.

### STAR Scenario 2: Log Rotation and Cleanup Automation

**Situation**: Application servers running out of disk space due to excessive log files. Manual cleanup consuming 2 hours/week of engineer time. Need automated, intelligent log management across 50 servers.

**Task**: Implement automated log rotation, compression, and cleanup using systemd timers with different schedules for different log types.

**Action**:
```bash
# Step 1: Create intelligent log cleanup script
sudo tee /usr/local/bin/smart-log-cleanup.sh << 'EOF'
#!/bin/bash
set -euo pipefail

LOG_ROOT="/var/log"
COMPRESSION_AGE=7    # Compress logs older than 7 days
DELETION_AGE=90      # Delete logs older than 90 days
MIN_FREE_SPACE=20    # Minimum free space percentage

log_action() {
    logger -t log-cleanup "$1"
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Check available disk space
DISK_FREE=$(df /var/log | awk 'NR==2 {print $5}' | sed 's/%//')
DISK_USED=$((100 - DISK_FREE))

log_action "Starting log cleanup - Current disk usage: ${DISK_USED}%"

# Emergency cleanup if disk is almost full
if [[ $DISK_USED -gt 85 ]]; then
    log_action "EMERGENCY: Disk usage over 85%, aggressive cleanup mode"
    find "$LOG_ROOT" -name "*.log" -mtime +3 -size +100M -delete
    find "$LOG_ROOT" -name "*.log.*" -mtime +1 -delete
fi

# Regular compression of old logs
find "$LOG_ROOT" -name "*.log" -mtime +$COMPRESSION_AGE ! -name "*.gz" -exec gzip {} \;
log_action "Compressed logs older than $COMPRESSION_AGE days"

# Remove very old compressed logs
DELETED_COUNT=$(find "$LOG_ROOT" -name "*.log.gz" -mtime +$DELETION_AGE -delete -print | wc -l)
log_action "Deleted $DELETED_COUNT old compressed logs"

# Application-specific cleanup
# Clean up application temp logs
find /opt/*/logs -name "tmp_*.log" -mtime +1 -delete 2>/dev/null || true

# Clean up debug logs more aggressively
find "$LOG_ROOT" -name "*debug*.log" -mtime +14 -delete

FINAL_DISK_FREE=$(df /var/log | awk 'NR==2 {print $5}' | sed 's/%//')
FINAL_DISK_USED=$((100 - FINAL_DISK_FREE))

log_action "Cleanup completed - Final disk usage: ${FINAL_DISK_USED}%"
EOF

sudo chmod +x /usr/local/bin/smart-log-cleanup.sh

# Step 2: Create cleanup service
sudo tee /etc/systemd/system/log-cleanup.service << 'EOF'
[Unit]
Description=Smart Log Cleanup Service
Documentation=file:///usr/local/bin/smart-log-cleanup.sh

[Service]
Type=oneshot
ExecStart=/usr/local/bin/smart-log-cleanup.sh
StandardOutput=journal
StandardError=journal
SyslogIdentifier=log-cleanup

# Security
User=root
PrivateTmp=true
ProtectHome=true

# Resource limits
MemoryMax=256M
CPUQuota=25%
IOSchedulingClass=3
IOSchedulingPriority=7
EOF

# Step 3: Create multiple timers for different schedules
# Daily cleanup timer
sudo tee /etc/systemd/system/log-cleanup-daily.timer << 'EOF'
[Unit]
Description=Daily Log Cleanup
Documentation=file:///usr/local/bin/smart-log-cleanup.sh

[Timer]
OnCalendar=daily
RandomizedDelaySec=3600
Persistent=true

[Install]
WantedBy=timers.target
EOF

# Emergency cleanup timer (runs every 4 hours when disk is full)
sudo tee /etc/systemd/system/log-cleanup-emergency.timer << 'EOF'
[Unit]
Description=Emergency Log Cleanup
Documentation=file:///usr/local/bin/smart-log-cleanup.sh

[Timer]
OnCalendar=*-*-* 00,06,12,18:00:00
RandomizedDelaySec=600

[Install]
WantedBy=timers.target
EOF

# Step 4: Enable timers and monitor
sudo systemctl daemon-reload
sudo systemctl enable --now log-cleanup-daily.timer
sudo systemctl enable --now log-cleanup-emergency.timer

# Verify timer status
systemctl list-timers log-cleanup*
```

**Result**: Reduced disk space usage by 60% on average. Eliminated manual cleanup work saving 8 hours/month. Prevented 12 disk-full incidents in first quarter.

### STAR Scenario 3: High-Performance Socket Activation for Microservices

**Situation**: Microservices architecture experiencing cold start delays of 3-5 seconds when services aren't actively used. Memory consumption high with 20+ idle services running continuously.

**Task**: Implement socket activation to reduce memory footprint and eliminate cold start delays while maintaining sub-100ms response times.

**Action**:
```bash
# Step 1: Create socket-activated web service
sudo tee /etc/systemd/system/api-service.socket << 'EOF'
[Unit]
Description=API Service Socket
Documentation=https://api.company.com/docs

[Socket]
ListenStream=8080
ListenStream=8443
Accept=false
MaxConnections=1000
MaxConnectionsPerSource=100

# Socket optimization
NoDelay=true
KeepAlive=true
KeepAliveIntervalSec=60
KeepAliveProbes=3
KeepAliveTimeSec=7200

# Security
SocketUser=api-service
SocketGroup=api-service
SocketMode=0660

[Install]
WantedBy=sockets.target
EOF

# Step 2: Create the service that gets activated
sudo tee /etc/systemd/system/api-service.service << 'EOF'
[Unit]
Description=API Service
Documentation=https://api.company.com/docs
Requires=api-service.socket
After=api-service.socket
Wants=network-online.target postgresql.service
After=network-online.target postgresql.service

[Service]
Type=notify
ExecStart=/opt/api/bin/api-server --socket-activation
ExecReload=/bin/kill -USR1 $MAINPID
KillMode=mixed
KillSignal=SIGTERM
TimeoutStopSec=30

# Auto-shutdown when idle to save memory
RuntimeMaxSec=3600

# Resource limits
MemoryMax=512M
CPUQuota=100%

# Security
User=api-service
Group=api-service
WorkingDirectory=/opt/api
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/opt/api/data /var/log/api

# Socket activation
StandardInput=socket
StandardOutput=journal
StandardError=journal

[Install]
Also=api-service.socket
EOF

# Step 3: Create socket-activated database connection pool
sudo tee /etc/systemd/system/db-pool.socket << 'EOF'
[Unit]
Description=Database Connection Pool Socket
Documentation=man:pgbouncer(1)

[Socket]
ListenStream=5432
Accept=false
SocketUser=postgres
SocketGroup=postgres

[Install]
WantedBy=sockets.target
EOF

sudo tee /etc/systemd/system/db-pool.service << 'EOF'
[Unit]
Description=Database Connection Pool
Documentation=man:pgbouncer(1)
Requires=db-pool.socket
After=db-pool.socket postgresql.service

[Service]
Type=forking
User=postgres
Group=postgres
PIDFile=/var/run/pgbouncer/pgbouncer.pid
ExecStart=/usr/bin/pgbouncer -d /etc/pgbouncer/pgbouncer.ini
ExecReload=/bin/kill -HUP $MAINPID
StandardInput=socket

# Auto-shutdown when idle
RuntimeMaxSec=1800

[Install]
Also=db-pool.socket
EOF

# Step 4: Configure nginx for socket proxying
sudo tee /etc/nginx/sites-available/socket-proxy << 'EOF'
upstream api_backend {
    server unix:/run/api-service.sock;
}

server {
    listen 80;
    server_name api.company.com;
    
    location / {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 1s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
EOF

# Step 5: Enable socket activation
sudo systemctl daemon-reload
sudo systemctl enable api-service.socket db-pool.socket
sudo systemctl start api-service.socket db-pool.socket

# Test socket activation
systemctl status api-service.socket
ss -tlnp | grep :8080

# Trigger service activation
curl http://localhost:8080/health
systemctl status api-service.service
```

**Result**: Reduced idle memory consumption by 70%. Eliminated cold start delays - services now respond in 50ms. Automatic shutdown saves $200/month in server costs.

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

## STAR Learning Exercises

These exercises follow the STAR method to reinforce learning through hands-on scenarios that mirror real production challenges.

### Exercise 1: Emergency Service Recovery

**Situation**: You've inherited a legacy application that keeps crashing in production. The current systemd service has no restart policy, no resource limits, and runs as root. Every crash requires manual intervention, happening 2-3 times per week.

**Task**: Create a robust, secure service configuration that automatically recovers from failures and implements security best practices.

**Your Action Plan**:
1. Create a secure, non-privileged user for the service
2. Implement comprehensive restart policies
3. Add resource limits to prevent system impact
4. Enable proper logging and monitoring
5. Test failure scenarios

**Expected Result**: Service should automatically recover from crashes, run securely, and provide actionable logs for debugging.

```bash
# Starter setup - build upon this
sudo useradd -r -s /bin/false -d /opt/myapp myapp

# Create your robust service here...
# Your service unit should include:
# - Security hardening
# - Resource limits  
# - Restart policies
# - Proper logging

# Test your configuration:
# - Kill the process and verify auto-restart
# - Verify it runs as non-root
# - Check resource usage limits
```

### Exercise 2: Multi-Environment Deployment System

**Situation**: Your team needs to deploy the same application to development, staging, and production environments on the same server. Each environment needs different configurations, ports, and resource limits.

**Task**: Design a systemd template-based deployment system that allows easy management of multiple application instances.

**Your Action Plan**:
1. Create a systemd template service
2. Design environment-specific configuration
3. Implement service discovery mechanism
4. Add environment-specific resource controls
5. Create management scripts for easy deployment

**Expected Result**: Ability to run `systemctl start myapp@dev.service` and have development environment configured automatically.

```bash
# Design template service: myapp@.service
# Environment configs should be in: /etc/myapp/%i.conf
# Ports should auto-increment: dev=8080, staging=8081, prod=8082

# Your template design here...
# Test with: systemctl start myapp@dev.service myapp@staging.service
```

### Exercise 3: Intelligent Backup System with Failure Recovery

**Situation**: Current backup system using cron fails silently. You need a backup solution that retries failures, sends notifications, and has different schedules for different data types.

**Task**: Build a comprehensive backup system using systemd timers with error handling, notifications, and multiple backup strategies.

**Your Action Plan**:
1. Create backup scripts with comprehensive error handling
2. Design multiple timer schedules (hourly, daily, weekly)
3. Implement failure notification system
4. Add backup verification and cleanup
5. Create monitoring and reporting

**Expected Result**: Reliable backup system with automated failure handling and clear visibility into backup status.

```bash
# Create backup service that:
# - Handles database, files, and configuration backups
# - Retries failures with exponential backoff
# - Sends success/failure notifications
# - Verifies backup integrity
# - Cleans up old backups automatically

# Your backup system implementation...
```

### Exercise 4: High-Availability Service Architecture

**Situation**: Critical payment processing service needs 99.9% uptime. Current setup has no failover, health checks, or load balancing. You need to design a system that continues working even during maintenance.

**Task**: Implement a blue-green deployment system using systemd with health checks, automatic failover, and zero-downtime updates.

**Your Action Plan**:
1. Design blue-green service architecture
2. Implement comprehensive health checking
3. Create automatic failover logic
4. Add load balancer integration
5. Test deployment and rollback procedures

**Expected Result**: System that can deploy updates without any service interruption and automatically handles service failures.

```bash
# Design services: payment-blue.service, payment-green.service
# Create health check monitoring
# Implement nginx upstream switching
# Add automated deployment pipeline

# Your high-availability setup...
```

### Exercise 5: Complete System Troubleshooting

**Situation**: Production server has multiple systemd-related issues: services won't start, boot takes 10+ minutes, disk is full from logs, and there are memory leaks causing crashes.

**Task**: Systematically diagnose and fix all issues while documenting your troubleshooting process.

**Your Action Plan**:
1. Analyze boot performance and identify bottlenecks
2. Investigate service startup failures
3. Implement log management and cleanup
4. Set up memory monitoring and leak detection
5. Document all fixes and create prevention measures

**Expected Result**: Fully functional system with preventive monitoring to avoid future issues.

```bash
# Use all troubleshooting tools:
# - systemd-analyze for boot issues
# - journalctl for service problems  
# - systemd-cgtop for resource monitoring
# - coredumpctl for crash analysis

# Your comprehensive troubleshooting approach...
```

### Self-Assessment Checklist

After completing each exercise, verify:
- [ ] Service starts automatically on boot
- [ ] Proper error handling and recovery implemented
- [ ] Security best practices followed (non-root user, limited permissions)
- [ ] Resource limits configured appropriately
- [ ] Comprehensive logging with actionable information
- [ ] Monitoring and alerting in place
- [ ] Documentation created for operational procedures

### Bonus Challenge: Production Readiness Assessment

Create a comprehensive systemd service health check that evaluates:
- Service configuration security score
- Resource utilization efficiency
- Restart policy effectiveness
- Log management compliance
- Dependency chain optimization

Document findings and recommendations for production deployment.

### Creating Your Own STAR Scenarios

Template for documenting your real-world systemd experiences:

```markdown
**Situation**: [Describe the production issue, urgency, and business impact]

**Task**: [Define specific objectives and success criteria]

**Action**:
```bash
# Step 1: [Investigation and diagnosis]
# Step 2: [Solution implementation]  
# Step 3: [Testing and verification]
# Step 4: [Making changes persistent]
# Step 5: [Monitoring and prevention]
```

**Result**: [Quantify improvements and lessons learned]
```

## STAR Scenarios for Troubleshooting and Boot Analysis

### STAR Scenario 1: Production Service Startup Failure Investigation

**Situation**: Critical payment processing service fails to start after routine security update. Service worked fine before update. Payment processing down affecting $50K/hour revenue. No error visible in basic systemctl status.

**Task**: Diagnose root cause and restore service within 15 minutes while preserving security updates.

**Action**:
```bash
# Step 1: Get detailed service status and recent logs
systemctl status payment-processor.service -l
journalctl -xeu payment-processor.service --no-pager -l

# Step 2: Check if unit file has syntax issues
systemd-analyze verify payment-processor.service
# Found: Invalid value for ExecStart

# Step 3: Compare current unit with backup
sudo diff /etc/systemd/system/payment-processor.service /etc/systemd/system/payment-processor.service.backup
# Shows: ExecStart path changed from /opt/payments/bin/app to /opt/payment/bin/app (typo)

# Step 4: Check dependency chain for issues
systemctl list-dependencies payment-processor.service --failed
# postgresql.service failed to start

# Step 5: Investigate PostgreSQL failure
journalctl -u postgresql.service --since "1 hour ago"
# Found: Permission denied on /var/lib/postgresql/data

# Step 6: Fix permissions and path issues
sudo chown -R postgres:postgres /var/lib/postgresql/data
sudo chmod 700 /var/lib/postgresql/data

# Fix the typo in service file
sudo sed -i 's|/opt/payment/bin/app|/opt/payments/bin/app|' /etc/systemd/system/payment-processor.service

# Step 7: Test fix
sudo systemctl daemon-reload
sudo systemctl start postgresql.service
sudo systemctl start payment-processor.service

# Step 8: Verify service is processing payments
curl -f http://localhost:8080/health
systemctl is-active payment-processor.service
```

**Result**: Service restored in 8 minutes. Root cause: Security update changed file permissions + typo in service file. Implemented automated unit file validation in CI/CD pipeline.

### STAR Scenario 2: System Boot Hang Diagnosis

**Situation**: Production database server hangs during boot after power outage. Server sits at "A start job is running for..." message for 10+ minutes. Database clients timing out, affecting 15 dependent applications.

**Task**: Identify boot bottleneck and restore normal boot process without data loss.

**Action**:
```bash
# Step 1: Reboot with debug information (from rescue console)
# Edit kernel parameters at boot: systemd.log_level=debug systemd.log_target=kmsg

# Step 2: Analyze boot performance after successful boot
systemd-analyze
# Shows: Startup finished in 2min 30s (kernel) + 8min 45s (userspace)

systemd-analyze blame | head -20
# Shows: postgresql.service took 6min 30s

# Step 3: Analyze critical chain
systemd-analyze critical-chain postgresql.service
# Shows: postgresql depends on slow-mounting.service

# Step 4: Investigate slow mount
systemctl status slow-mounting.service -l
journalctl -u slow-mounting.service
# Found: NFS mount timing out

# Step 5: Check mount dependencies
systemctl list-dependencies postgresql.service
cat /etc/fstab | grep nfs
# Found: PostgreSQL data directory on NFS mount with _netdev missing

# Step 6: Fix mount options to prevent boot hang
sudo cp /etc/fstab /etc/fstab.backup
sudo sed -i 's|nfs4.*defaults|nfs4 defaults,_netdev,soft,timeo=30,retrans=2|' /etc/fstab

# Step 7: Create override to make PostgreSQL less dependent on problematic mount
sudo systemctl edit postgresql.service
```
```ini
[Unit]
# Don't fail if NFS isn't available, use local storage
After=local-fs.target
Wants=local-fs.target
```

```bash
# Step 8: Test boot with recovery options
sudo systemctl daemon-reload
sudo systemctl enable postgresql.service

# Create emergency fallback
sudo tee /etc/systemd/system/postgresql-local-fallback.service << 'EOF'
[Unit]
Description=PostgreSQL Local Fallback
After=postgresql.service
Requires=postgresql.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/check-postgresql-health.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
EOF
```

**Result**: Boot time reduced from 11+ minutes to 45 seconds. NFS mount failures no longer block PostgreSQL startup. Implemented local storage fallback for critical services.

### STAR Scenario 3: Service Memory Leak and Crash Analysis

**Situation**: Web application service randomly crashes every 2-3 days in production. No obvious pattern. Service restarts automatically but causes 30-second service interruption affecting user experience.

**Task**: Identify crash root cause using systemd's debugging capabilities and implement preventive measures.

**Action**:
```bash
# Step 1: Enable comprehensive crash debugging
sudo systemctl edit webapp.service
```
```ini
[Service]
# Enable core dumps
LimitCORE=infinity
Environment=SYSTEMD_DUMP_CORE=1

# Restart with delay to avoid restart loops
Restart=on-failure
RestartSec=30s
StartLimitInterval=300s
StartLimitBurst=3
```

```bash
# Step 2: Configure systemd-coredump
sudo tee /etc/systemd/coredump.conf << 'EOF'
[Coredump]
Storage=external
Compress=yes
ProcessSizeMax=8G
ExternalSizeMax=8G
MaxUse=16G
KeepFree=1G
EOF

sudo systemctl restart systemd-coredump

# Step 3: Add memory monitoring to service
sudo tee /usr/local/bin/memory-monitor.sh << 'EOF'
#!/bin/bash
SERVICE_NAME="webapp.service"
MEMORY_THRESHOLD=80  # Restart if memory usage > 80%

while true; do
    if systemctl is-active "$SERVICE_NAME" >/dev/null; then
        PID=$(systemctl show -p MainPID "$SERVICE_NAME" | cut -d= -f2)
        if [ "$PID" != "0" ]; then
            MEMORY_PERCENT=$(ps -o pid,pmem --no-headers -p "$PID" | awk '{print $2}' | cut -d. -f1)
            if [ "$MEMORY_PERCENT" -gt "$MEMORY_THRESHOLD" ]; then
                logger -t memory-monitor "WARNING: $SERVICE_NAME memory usage ${MEMORY_PERCENT}% exceeds threshold"
                # Trigger graceful restart
                systemctl reload-or-restart "$SERVICE_NAME"
            fi
        fi
    fi
    sleep 60
done
EOF

sudo chmod +x /usr/local/bin/memory-monitor.sh

# Step 4: Create memory monitoring service
sudo tee /etc/systemd/system/webapp-memory-monitor.service << 'EOF'
[Unit]
Description=Webapp Memory Monitor
After=webapp.service
Wants=webapp.service

[Service]
Type=simple
ExecStart=/usr/local/bin/memory-monitor.sh
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

# Step 5: Wait for next crash and analyze
# After crash occurs:
coredumpctl list webapp
coredumpctl info webapp | head -30
coredumpctl gdb webapp << 'EOF'
bt
info registers
print $rsp
quit
EOF

# Step 6: Analyze service logs around crash time
journalctl -u webapp.service --since "24 hours ago" | grep -C 10 -i "crash\|segfault\|killed"

# Step 7: Enable additional debugging if needed
sudo systemctl edit webapp.service
```
```ini
[Service]
Environment=MALLOC_CHECK_=2
Environment=MALLOC_PERTURB_=42
# Enable AddressSanitizer if available
Environment=ASAN_OPTIONS=abort_on_error=1:disable_coredump=0
```

**Result**: Identified memory leak in third-party library causing crashes. Implemented proactive memory monitoring preventing crashes. Service uptime improved from 99.2% to 99.8%.

### STAR Scenario 4: Systemd Journal Space Management Crisis

**Situation**: Server disk full due to systemd journal consuming 15GB of space. Critical services failing to write logs. System becoming unstable. Need immediate space recovery without losing important diagnostic information.

**Task**: Free up disk space immediately while preserving recent logs and implementing long-term journal management.

**Action**:
```bash
# Step 1: Assess journal disk usage immediately
journalctl --disk-usage
# Shows: Archived and active journals take up 15.2G on disk

# Step 2: Check what's consuming the most space
du -sh /var/log/journal/*
ls -lah /var/log/journal/*/

# Step 3: Emergency cleanup - remove old logs but keep recent ones
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=2G

# Step 4: Verify space recovered
df -h /var/log
journalctl --disk-usage

# Step 5: Configure permanent journal size limits
sudo tee /etc/systemd/journald.conf << 'EOF'
[Journal]
# Storage location
Storage=persistent

# Size limits
SystemMaxUse=1G
SystemKeepFree=2G
SystemMaxFileSize=100M
SystemMaxFiles=10

# Retention
MaxRetentionSec=30day
MaxFileSec=1week

# Forward to syslog for long-term storage
ForwardToSyslog=yes
ForwardToWall=no

# Compress logs
Compress=yes

# Rate limiting
RateLimitInterval=10s
RateLimitBurst=200
EOF

# Step 6: Restart journald to apply configuration
sudo systemctl restart systemd-journald

# Step 7: Set up log rotation for important services
sudo tee /etc/logrotate.d/critical-services << 'EOF'
/var/log/webapp/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0644 webapp webapp
    postrotate
        systemctl reload webapp.service
    endscript
}

/var/log/database/*.log {
    daily
    rotate 60
    compress
    delaycompress
    missingok
    notifempty
    create 0640 postgres postgres
}
EOF

# Step 8: Create disk space monitoring
sudo tee /usr/local/bin/disk-space-monitor.sh << 'EOF'
#!/bin/bash
THRESHOLD=85
DISK_USAGE=$(df /var/log | awk 'NR==2 {print $5}' | sed 's/%//')

if [ "$DISK_USAGE" -gt "$THRESHOLD" ]; then
    logger -p crit "Disk usage at ${DISK_USAGE}% - running emergency cleanup"
    journalctl --vacuum-size=500M
    find /var/log -name "*.log.*" -mtime +7 -delete
fi
EOF

sudo chmod +x /usr/local/bin/disk-space-monitor.sh

# Add to cron for regular monitoring
echo "*/30 * * * * root /usr/local/bin/disk-space-monitor.sh" | sudo tee -a /etc/crontab
```

**Result**: Recovered 13GB disk space immediately. Implemented automated journal management preventing future incidents. System stability restored with proper log retention policies.

## Commands Reference

### Scenario-Based Command Guide

**Emergency Service Recovery:**
```bash
# Quick service status with full details
systemctl status service-name.service -l --no-pager

# Get last 50 lines of service logs with timestamps
journalctl -u service-name.service -n 50 --no-pager

# Restart failed service and follow logs
sudo systemctl restart service-name.service && journalctl -u service-name.service -f
```

**Service Dependency Investigation:**
```bash
# Find what prevents service from starting
systemctl list-dependencies service-name.service --failed

# See what services depend on this one
systemctl list-dependencies --reverse service-name.service

# Check which services are blocking boot
systemd-analyze critical-chain
```

**Resource Usage Monitoring:**
```bash
# Real-time resource usage by services
systemd-cgtop

# Check specific service resource limits and usage
systemctl show service-name.service | grep -E "Memory|CPU|Tasks"

# Monitor service restarts and failures
journalctl -u service-name.service | grep -E "Started|Stopped|Failed"
```

**Timer and Job Management:**
```bash
# List all timers with next execution time
systemctl list-timers --all

# Check why timer didn't run
journalctl -u timer-name.timer --since "yesterday"

# Force timer to run now for testing
sudo systemctl start timer-service-name.service
```

**Boot Performance Analysis:**
```bash
# Overall boot time breakdown
systemd-analyze

# Services taking longest to start
systemd-analyze blame | head -20

# Critical path analysis
systemd-analyze critical-chain

# Generate boot timeline graph
systemd-analyze plot > boot-analysis.svg
```

**Log Management:**
```bash
# Check journal disk usage by service
journalctl --disk-usage

# Clean up old logs (keep last 7 days)
sudo journalctl --vacuum-time=7d

# Follow logs from multiple services
journalctl -u service1.service -u service2.service -f

# Search logs for errors across all services
journalctl -p err --since "1 hour ago"
```

**Security Analysis:**
```bash
# Analyze service security configuration
systemd-analyze security service-name.service

# Check which services run as root
systemctl list-units --type=service --state=running --no-pager | xargs -I {} systemctl show {} -p User | grep "User=root" -B1

# Find services with network access
systemctl list-units --type=service --state=running --no-pager | xargs -I {} systemctl show {} -p RestrictAddressFamilies | grep -v "RestrictAddressFamilies=$"
```

**Configuration Management:**
```bash
# Edit service with override (preserves original)
sudo systemctl edit service-name.service

# Edit full service file
sudo systemctl edit --full service-name.service

# Reload configuration after changes
sudo systemctl daemon-reload

# Verify service configuration syntax
systemd-analyze verify service-name.service
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