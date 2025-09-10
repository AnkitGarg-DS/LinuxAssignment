# LinuxAssignment :  Monitoring, Users, and Backups

## Environment
- OS: CentOS (root)
- Terminal: PuTTY (SSH)
- Timezone: Asia/Kolkata

## Task 1 – Monitoring
- Installed: `epel-release`, `htop`, `nmon`
- Disk Monitoring: `df -h`, `du -h --max-depth=1`
- Script: `/usr/local/bin/sys_monitor.sh`
- Logs: `/var/log/sys_monitor/monitor_YYYY-MM-DD_HH-MM-SS.log`
- (Optional) Cron: `0 6 * * * /usr/local/bin/sys_monitor.sh`

## Task 2 – Users & Access
- Users: `Sarah` (home `/home/Sarah`), `Mike` (home `/home/Mike`)
- Workspaces: `/home/Sarah/workspace`, `/home/Mike/workspace` with `chmod 700`
- Password policy:
  - `/etc/login.defs`: `PASS_MAX_DAYS=30`, `PASS_MIN_DAYS=7`, `PASS_WARN_AGE=7`
  - `chage` applied to both users
  - Complexity in `/etc/security/pwquality.conf`: `minlen=8`, `dcredit=-1`, `ucredit=-1`, `lcredit=-1`, `ocredit=-1`
 
## Task 3 – Backups
- Backup dir: `/backups/`
- Scripts:
  - Apache: `/usr/local/bin/apache_backup.sh` → `/etc/httpd/` + `/var/www/html/`
  - Nginx:  `/usr/local/bin/nginx_backup.sh` → `/etc/nginx/` + `/usr/share/nginx/html/`
- Sudoers: `/etc/sudoers.d/web-backups` to allow only these scripts
- Crons:
  - Sarah: `0 0 * * 2 sudo /usr/local/bin/apache_backup.sh`
  - Mike:  `0 0 * * 2 sudo /usr/local/bin/nginx_backup.sh`
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
