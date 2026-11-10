# Day 1: Budget alert + Linux test-out

**Date:** 9 Nov 2026 (tracker Day 1) | **Time:** ~2 hrs | **Status:** Completed

## What I built
- AWS Budget: monthly cost budget of $5.32 (about Rs 500)
  - Alert 1: actual cost above 75%
  - Alert 2: forecasted cost above 100%
  - Email verified, so alerts can be delivered
- Ubuntu 26.04 EC2 instance (t3.small, ap-south-1)
- Users and groups: `devuser`, group `devteam`, sudo access
- Shared folder `/opt/project` (group `devteam`, mode `2775` with setgid)
- systemd service `hello.service` (restarts automatically)
- Cron jobs running as `devuser`

## Commands I used
```bash
# Users and groups
sudo groupadd devteam
sudo useradd -m -s /bin/bash devuser
sudo usermod -aG devteam,sudo devuser
id devuser
getent group devteam
sudo gpasswd -d devuser sudo        # remove sudo

# Shared folder
sudo mkdir /opt/project
sudo chgrp devteam /opt/project
sudo chmod 2775 /opt/project        # 2 = setgid, new files inherit the group

# systemd
sudo systemctl daemon-reload
sudo systemctl enable --now hello
systemctl status hello
journalctl -u hello -n 10 --no-pager

# cron
sudo crontab -u devuser -l
* * * * * /opt/project/stamp.sh >> /opt/project/stamp.log 2>&1
```

## What I learned
- `usermod -aG` needs `-a` (append), or it replaces all the user's groups.
- The setgid bit (`s` in `drwxrwsr-x`) makes new files inherit the folder's group.
- Setgid only affects new files. Existing files keep their old owner and group.
- `Restart=always` makes systemd restart a crashed service. I killed the process and saw the PID change.
- Troubleshooting order for services: `systemctl status`, then `journalctl -u`, then fix.
- `>` overwrites a file, `>>` appends. `2>&1` sends errors to the same place as normal output.

## Troubleshooting drills
**1. Service won't start**
- Symptom: `status=203/EXEC`, `Permission denied`
- Cause: I removed the execute bit from `hello.sh`
- Fix: `chmod +x`, then `systemctl restart hello`

**2. Works by hand, fails in cron**
- Symptom: `mytool2: command not found` in the cron log
- Cause: `/opt/tools` is not in cron's PATH
- Fix: use the full path, `/opt/tools/mytool2`

## Rule I'm keeping
Use full paths in cron jobs and systemd services.

## Cleanup
Removed the cron jobs and stopped the EC2 instance.
