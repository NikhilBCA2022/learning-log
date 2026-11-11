# Day 5: Install Jenkins on an Ubuntu VM

**Date:** 11 Nov 2026 | **Region:** Mumbai (ap-south-1) | **Status:** Completed

## Goal

Install Jenkins on an Ubuntu EC2 instance, open it in the browser, finish the setup wizard, and practise troubleshooting an unreachable Jenkins UI.

## What Jenkins is, in simple words

Think of a factory assembly line for software. Every time code changes, someone must build it, test it, and package it. **Jenkins is a robot worker that does those steps automatically.** I describe the steps once (a pipeline), and Jenkins runs them on every change. This is the "CI" in CI/CD (Continuous Integration).

## Environment

| Item | Value |
|---|---|
| Instance | `84DayChallenge`, t3.small (2 vCPU, 1.9 GiB RAM), 14 GB disk |
| OS | Ubuntu 26.04 LTS |
| Java | OpenJDK 21 |
| Jenkins | 2.580.2 (LTS), installed from the `debian-stable` apt repository |
| Port | 8080, allowed in the security group from **my IP only** |

## Install steps

```bash
# 1. Check resources
free -h
df -h /
sudo apt update

# 2. Java first (Jenkins is a Java program)
sudo apt install -y fontconfig openjdk-21-jre
java -version

# 3. Add the Jenkins repository (2026 key)
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update

# 4. Install and start Jenkins
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
systemctl status jenkins --no-pager

# 5. Check from inside the server
sudo ss -tlnp | grep 8080
curl -I http://localhost:8080          # 403 = running, login required
```

Notes:
- Install Java **before** Jenkins, or the service can fail to start.
- Jenkins moved to a **2026 repository key**. Older tutorials that use the 2023 key no longer work.
- The `Ign: ... InRelease` line during `apt update` is harmless.

## Browser setup

1. Security group: added inbound **Custom TCP 8080, source My IP** (not `0.0.0.0/0`, because Jenkins can run code on the server).
2. Opened `http://<public-ip>:8080` and got **Unlock Jenkins**.
3. Chose **Install suggested plugins** (Git, Pipeline, Credentials and other starter plugins). Installing everything would slow a 2 GB server and add security updates to track.
4. Created the first admin user.

## Problem 1: Built-In Node went offline (disk space)

The log showed:

```
Making Built-In Node offline temporarily due to the lack of disk space
```

**Cause:** Jenkins wants about 1 GB free in its temp folder. On this Ubuntu, `/tmp` is a `tmpfs` (it lives in RAM), sized at half of RAM:

```
tmpfs   953M   4.8M   948M   /tmp
```

The main disk had 11 GB free, so the disk was never the problem. `/tmp` was just under the limit.

**Fix:** lower the **Free Temp Space** threshold under Manage Jenkins > Nodes > Configure. The log then showed the node coming back online.

## Problem 2: Locked out of Jenkins

**Symptom:** "Invalid username or password."

**What I found:**
- `/var/lib/jenkins/secrets/initialAdminPassword` no longer existed. **Jenkins deletes it after the setup wizard**, so the unlock password cannot be recovered.
- Jenkins has no "forgot password" email by default.

**Recovery (own test server only):**

```bash
sudo systemctl stop jenkins
sudo cp /var/lib/jenkins/config.xml /var/lib/jenkins/config.xml.bak
sudo sed -i 's#<useSecurity>true</useSecurity>#<useSecurity>false</useSecurity>#' /var/lib/jenkins/config.xml
sudo systemctl start jenkins
```

1. With the login off, opened Manage Jenkins > Security.
2. Set the security realm to Jenkins' own user database, authorization to "Logged-in users can do anything", anonymous read access off, and temporarily allowed sign-up.
3. Created a new account and logged in.
4. **Turned sign-up off again** and confirmed `useSecurity` was `true`:

```bash
sudo grep useSecurity /var/lib/jenkins/config.xml    # must print true
sudo rm /var/lib/jenkins/config.xml.bak
```

**Warning:** while security is off, anyone who can reach port 8080 has full control. My security group limited this to my IP, and I finished quickly. Do not do this on a real server without a better plan.

**Lesson:** save the admin username and password in a password manager during setup.

## Troubleshooting drill: Jenkins UI unreachable on port 8080

| Symptom | `curl localhost:8080` on the server | Likely cause | First check |
|---|---|---|---|
| Fails instantly (refused) | Fails | Service stopped | `systemctl status jenkins` |
| Hangs, then times out | Works (403) | Firewall or security group, or my IP changed | Port 8080 rule in the security group |
| Page loads, asks for a password | Works (403) | Normal | Log in |

**Service-stopped test (done):**

```bash
sudo systemctl stop jenkins
curl -I --max-time 5 http://localhost:8080
# curl: (7) Failed to connect ... Could not connect to server  (connection refused)
sudo systemctl start jenkins
curl -I --max-time 5 http://localhost:8080
# HTTP/1.1 403 Forbidden   (Jenkins is back, login required)
```

Where to look for the cause: `systemctl status jenkins`, then `journalctl -u jenkins -n 30 --no-pager`.

## Key takeaways

- Jenkins is a Java service managed by systemd, the same as the `hello.service` from Day 1.
- "Connection refused" means nothing is listening (a service problem). A timeout usually means a firewall.
- A `403` from `curl localhost:8080` is a good sign: Jenkins is running and wants a login.
- Check free space on `/tmp` as well as `/`, because `/tmp` can be a small RAM disk.
- Open Jenkins' port to my IP only, and keep sign-up closed.

## Cleanup

- Stopped `84DayChallenge` (stopped, not terminated).

## Not done

- Drill 2 (remove the port 8080 rule and watch the browser time out). I already knew this symptom from the Day 3 NACL experiment.
