# LinuxAssignment :  Monitoring, Users, and Backups

## Environment
- OS: CentOS (root)
- Terminal: PuTTY (SSH)
- Timezone: Asia/Kolkata

## Task 1 – Monitoring
- Installed: `epel-release`, `htop`, `nmon`
  ```bash
  yum install -y epel-release     # enables EPEL repo (htop/nmon live here)
  yum install -y htop nmon

- Disk Monitoring: `df -h`, `du -h --max-depth=1`
  ```bash
  df -h                      # filesystem-level usage
  du -sh /home/*             # size of each top-level folder under /home
  du -h --max-depth=1 /var   # per-dir usage under /var

- Script: `/usr/local/bin/sys_monitor.sh`
  ```bash
    cat > /usr/local/bin/sys_monitor.sh << 'EOF'
    #!/bin/bash
    set -euo pipefail
    
    DATE=$(date +"%Y-%m-%d_%H-%M-%S")
    LOG_DIR="/var/log/sys_monitor"
    LOG_FILE="$LOG_DIR/monitor_$DATE.log"
    
    mkdir -p "$LOG_DIR"
    
    {
      echo "===== System Monitoring Report: $DATE ====="
      echo
      echo "--- CPU/Memory/Load (top snapshot) ---"
      top -b -n1 | head -15
      echo
      echo "--- Disk Usage (df -h) ---"
      df -h
      echo
      echo "--- Largest directories under / (depth 1) ---"
      du -h --max-depth=1 / 2>/dev/null | sort -hr | head -15
      echo
      echo "--- Top 5 processes by Memory ---"
      ps aux --sort=-%mem | head -6
      echo
      echo "--- Top 5 processes by CPU ---"
      ps aux --sort=-%cpu | head -6
    } > "$LOG_FILE"
    EOF
    
    chmod +x /usr/local/bin/sys_monitor.sh

- Logs: `/var/log/sys_monitor/monitor_YYYY-MM-DD_HH-MM-SS.log`
  ```bash
    /usr/local/bin/sys_monitor.sh
    ls -lh /var/log/sys_monitor/

- (Optional) Cron: `0 6 * * * /usr/local/bin/sys_monitor.sh`
  ```bash
    crontab -e    # root's crontab
    # Add this line, save & exit:
    0 6 * * * /usr/local/bin/sys_monitor.sh

## Task 2 – Users & Access
- Users: `Sarah` (home `/home/Sarah`), `Mike` (home `/home/Mike`)
  ```bash
    # Sarah
    useradd -m -d /home/Sarah Sarah
    passwd Sarah   # set a strong password
    
    # Mike
    useradd -m -d /home/Mike Mike
    passwd Mike    # set a strong password

- Workspaces: `/home/Sarah/workspace`, `/home/Mike/workspace` with `chmod 700`
    ```bash
      mkdir -p /home/Sarah/workspace
      mkdir -p /home/mike/workspace
      
      chown -R Sarah:Sarah /home/Sarah
      chown -R mike:mike   /home/mike
      
      chmod 700 /home/Sarah/workspace
      chmod 700 /home/mike/workspace
      
      # verify
      ls -ld /home/Sarah/workspace /home/mike/workspace

- Password policy:
  - `/etc/login.defs`: `PASS_MAX_DAYS=30`, `PASS_MIN_DAYS=7`, `PASS_WARN_AGE=7`
      ```bash
          vi /etc/login.defs
      
  ### Set these values
        PASS_MAX_DAYS   30   # must change every 30 days
        PASS_MIN_DAYS   7    # minimum 7 days before next change
        PASS_WARN_AGE   7    # warn 7 days before expiry

  - `chage` applied to both users
      ```bash
        chage -M 30 -m 7 -W 7 Sarah
        chage -M 30 -m 7 -W 7 Mike
        chage -l Sarah   # verify
        chage -l Mike
      
  - Complexity in `/etc/security/pwquality.conf`: `minlen=8`, `dcredit=-1`, `ucredit=-1`, `lcredit=-1`, `ocredit=-1`
    ```bash
      yum install -y libpwquality

      vi /etc/security/pwquality.conf

  Set
    minlen = 8
    dcredit = -1   # require at least 1 digit
    ucredit = -1   # require at least 1 uppercase
    lcredit = -1   # require at least 1 lowercase
    ocredit = -1   # require at least 1 special char

 
## Task 3 – Backups
- Backup dir: `/backups/`
  ```bash
    mkdir -p /backups
    chmod 755 /backups
    ls -ld /backups

- Scripts:
  - Apache: `/usr/local/bin/apache_backup.sh` → `/etc/httpd/` + `/var/www/html/`
    ```bash
    cat /usr/local/bin/apache_backup.sh << 'EOF'
    #!/bin/bash
    set -euo pipefail
    
    DATE=$(date +"%Y-%m-%d")
    BACKUP="/backups/apache_backup_${DATE}.tar.gz"
    VERIFY_LOG="/backups/apache_backup_${DATE}.log"
    
    # Create compressed archive of config + document root
    tar -czf "$BACKUP" /etc/httpd/ /var/www/html/
    
    # Verify the archive by listing its contents
    {
      echo "Apache backup verification for $DATE"
      echo "Archive: $BACKUP"
      tar -tzf "$BACKUP" | head -20
      echo "..."
      echo "Total files in archive: $(tar -tzf "$BACKUP" | wc -l)"
      echo "SHA256: $(sha256sum "$BACKUP" | awk '{print $1}')"
    } > "$VERIFY_LOG"
    
    echo "Backup complete: $BACKUP"
    echo "Verification log: $VERIFY_LOG"
    EOF
    
    chmod +x /usr/local/bin/apache_backup.sh

  - Nginx:  `/usr/local/bin/nginx_backup.sh` → `/etc/nginx/` + `/usr/share/nginx/html/`
      ```bash
      cat /usr/local/bin/nginx_backup.sh << 'EOF'
      #!/bin/bash
      set -euo pipefail
      
      DATE=$(date +"%Y-%m-%d")
      BACKUP="/backups/nginx_backup_${DATE}.tar.gz"
      VERIFY_LOG="/backups/nginx_backup_${DATE}.log"
      
      tar -czf "$BACKUP" /etc/nginx/ /usr/share/nginx/html/
      
      {
        echo "Nginx backup verification for $DATE"
        echo "Archive: $BACKUP"
        tar -tzf "$BACKUP" | head -20
        echo "..."
        echo "Total files in archive: $(tar -tzf "$BACKUP" | wc -l)"
        echo "SHA256: $(sha256sum "$BACKUP" | awk '{print $1}')"
      } > "$VERIFY_LOG"
      
      echo "Backup complete: $BACKUP"
      echo "Verification log: $VERIFY_LOG"
      EOF
      
      chmod +x /usr/local/bin/nginx_backup.sh

- Sudoers: `/etc/sudoers.d/web-backups` to allow only these scripts
    ```bash
        visudo -f /etc/sudoers.d/web-backups
### Paste:
        Sarah ALL=(root) NOPASSWD: /usr/local/bin/apache_backup.sh
        Mike  ALL=(root) NOPASSWD: /usr/local/bin/nginx_backup.sh

- Crons:
  - Sarah: `0 0 * * 2 sudo /usr/local/bin/apache_backup.sh`
    ```bash
      crontab -u Sarah -e
      # add this line, save & exit
      0 0 * * 2 sudo /usr/local/bin/apache_backup.sh

  - Mike:  `0 0 * * 2 sudo /usr/local/bin/nginx_backup.sh`
      ```bash
      crontab -u mike -e
      # add this line, save & exit
      0 0 * * 2 sudo /usr/local/bin/nginx_backup.sh

- Output:
  - Archives: `/backups/apache_backup_2025-09-10.tar.gz`, `/backups/nginx_backup_2025-09-10.tar.gz`
  - Logs: `/backups/apache_backup_2025-09-10.log`, `/backups/nginx_backup_2025-09-10.log` (contains file listing + SHA256)

## Verification (examples)
```bash
df -h
du -sh /home/*

/usr/local/bin/sys_monitor.sh
ls -lh /var/log/sys_monitor/

id Sarah && id Mike
ls -ld /home/Sarah/workspace /home/Mike/workspace
chage -l Sarah
chage -l Mike

/usr/local/bin/apache_backup.sh
/usr/local/bin/nginx_backup.sh
ls -lh /backups/
cat /backups/apache_backup_2025-09-10.log
cat /backups/nginx_backup_2025-09-10.log
