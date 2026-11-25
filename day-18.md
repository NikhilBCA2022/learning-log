# Day 18 – Bash Scripting & Linux Automation

## Overview

Day 18 focused on learning Bash scripting and automating common Linux system administration tasks on an Ubuntu EC2 instance.

The goal was to understand how to write Bash scripts, monitor server resources, automate backups, manage users, and schedule recurring tasks using Cron Jobs.

## Technologies Used

- Linux (Ubuntu)
- Bash Shell Scripting
- AWS EC2
- Cron
- AWK
- TAR
- GZIP

## Project Structure

```text
day18-bash-lab/
│
├── scripts/
│   ├── system_info.sh
│   ├── backup.sh
│   ├── disk_alert.sh
│   ├── log_rotation.sh
│   ├── create_user.sh
│   └── health_check.sh
│
├── backups/
├── application.log
└── healthcheck.log
```

## Tasks Completed

### 1. System Information Script

Created a Bash script to display basic server information.

Information displayed:
- Current date and time (IST)
- Hostname
- System uptime
- Disk usage

Commands learned:
- `date`
- `hostname`
- `uptime`
- `df -h`

### 2. Automated Backup Script

Created a script to automate directory backups using TAR.

Features:
- Creates backup directories.
- Generates timestamped backup filenames.
- Compresses files using gzip.
- Checks whether backup creation succeeded.

Commands learned:
- `tar -czf`
- `tar -tzf`
- `tar -xzf`
- `mkdir -p`
- Command substitution

### 3. Disk Usage Alert Script

Created a script to monitor root filesystem disk usage.

Features:
- Retrieves disk usage percentage.
- Removes the percentage symbol using `tr`.
- Compares disk usage against a threshold.
- Displays a warning when usage reaches or exceeds 80%.

Commands learned:
- `df`
- `awk`
- `tr`
- `if-else`
- Numeric comparisons (`-ge`, `-gt`, `-lt`)

### 4. Log Rotation Script

Created a script to manage large log files.

Features:
- Checks whether a log file exists.
- Retrieves file size in bytes.
- Checks whether the file exceeds 10 MB.
- Compresses the log file using GZIP.
- Creates a new empty log file.

Commands learned:
- `stat`
- `gzip`
- `gunzip`
- `touch`
- `-f` file condition

### 5. Linux User Creation Script

Created a script to automate Linux user creation.

Features:
- Accepts a username as a command-line argument.
- Checks whether the username was provided.
- Checks whether the user already exists.
- Creates a new user if necessary.
- Displays appropriate success or error messages.

Commands learned:
- `$1`
- `$0`
- `$?`
- `id`
- `useradd`
- `sudo`
- `-z`
- Output redirection (`&>/dev/null`)

### 6. Server Health Check Script

Created a script to monitor the overall health of the EC2 server.

Information displayed:
- Hostname
- Current date and time
- Server uptime
- Root filesystem usage
- Total memory
- Used memory
- Available memory
- CPU load average

Commands learned:
- `free -h`
- `df -h`
- `uptime`
- `awk`
- Command substitution
- Bash variables

### 7. Cron Job Automation

Learned how to schedule scripts automatically using Linux Cron.

Configured a cron job to execute the server health-check script every minute.

Cron expression:

```cron
*/1 * * * * /bin/bash /home/ubuntu/day18-bash-lab/scripts/health_check.sh >> /home/ubuntu/day18-bash-lab/healthcheck.log 2>&1
```

Explanation:

| Component | Purpose |
|---|---|
| `*/1` | Every minute |
| First `*` | Every hour |
| Second `*` | Every day of the month |
| Third `*` | Every month |
| Fourth `*` | Every day of the week |
| `/bin/bash` | Executes the script |
| `>>` | Appends output to the log file |
| `2>&1` | Redirects errors to the same log file |

Commands learned:

```bash
crontab -e
crontab -l
```

Verified automatic execution using:

```bash
tail -n 20 ~/day18-bash-lab/healthcheck.log
```

## Important Bash Concepts Learned

- Variables and command substitution
- Conditional statements (`if-else`)
- Numeric comparisons
- Bash functions
- Command-line arguments
- File existence checks
- Exit status (`$?`)
- Input, output and error redirection
- Linux file descriptors (0, 1 and 2)
- AWK text processing
- TR character manipulation
- TAR archive management
- GZIP compression
- Cron scheduling

## Key Learnings

By completing this lab, I learned how to:

1. Automate repetitive Linux administration tasks.
2. Write Bash scripts for server monitoring.
3. Create and manage compressed backups.
4. Monitor disk and memory usage.
5. Automate Linux user creation.
6. Compress and rotate large log files.
7. Schedule scripts using Cron.
8. Redirect script output and errors into log files.

## Final Outcome

Successfully completed a practical Linux Bash automation lab on an AWS EC2 Ubuntu server.

This exercise provided hands-on experience with Linux administration, Bash scripting, server monitoring, backup automation, and scheduled task execution.

**Status: COMPLETED**
