# Day 3: Bastion Host + Security Groups vs NACLs

**Date:** 11 Nov 2026 | **Region:** Mumbai (ap-south-1) | **Status:** Completed

## Goal

Reach a private EC2 instance (no public IP) by jumping through a bastion host, then compare Security Groups and NACLs by breaking SSH on purpose.

## Architecture

```
                         Internet
                             |
                     [ Internet Gateway ]
                             |
 +------------------- day3-vpc (10.0.0.0/16) -------------------+
 |                                                              |
 |   Public subnet                  Private subnet              |
 |   +---------------------+        +---------------------+     |
 |   |  day3-bastion       |  SSH   |  day3-private       |     |
 |   |  public IP: yes     | -----> |  public IP: none    |     |
 |   |  10.0.10.91         |        |  10.0.139.243       |     |
 |   +---------------------+        +---------------------+     |
 |   route: 0.0.0.0/0 -> IGW        route: local only           |
 +--------------------------------------------------------------+

 My laptop --SSH (my IP only)--> bastion --SSH--> private server
```

## What I built

| Component | Setting |
|---|---|
| VPC | `day3-vpc`, `10.0.0.0/16`, DNS resolution and DNS hostnames enabled |
| Subnets | 1 public, 1 private (same Availability Zone) |
| Internet Gateway | Attached to the VPC |
| Route tables | Public: `0.0.0.0/0` to IGW. Private: `local` only |
| NAT Gateway | None (not needed for SSH, and it costs money) |
| Bastion | Ubuntu 26.04 LTS, t3.micro, public subnet, public IP |
| Private server | Ubuntu 26.04 LTS, t3.micro, private subnet, no public IP |
| `day3-bastion-sg` | Inbound SSH (22) from **my IP** only |
| `day3-private-sg` | Inbound SSH (22) from **`day3-bastion-sg`** only |
| Key pair | `day3-key` (RSA, .pem), stored on my laptop only |

The VPC was created with the console's **VPC and more** wizard, since I built each piece by hand on Day 2.

## How I connected

MobaXterm with an SSH gateway (jump host):

1. Remote host: the private server's private IP.
2. Network settings > SSH gateway: the bastion's public IP, user `ubuntu`.
3. The same `.pem` key selected on both the session and the gateway.

The key stays on my laptop. It is never copied to the bastion, because a compromised bastion would then expose everything behind it.

## Test results

| Test | Command | Result |
|---|---|---|
| Bastion to internet | `curl -I https://google.com` | `HTTP/2 301` (works, via Internet Gateway) |
| Bastion to private server | `nc -vz 10.0.139.243 22` | `succeeded` |
| Private server to internet | `curl -I --max-time 5 https://google.com` | `Connection timed out` (expected: no NAT gateway) |

## Security Group vs NACL

| | Security Group | NACL |
|---|---|---|
| Protects | One instance | A whole subnet |
| Rules | Allow only | Allow and deny |
| State | Stateful: reply traffic is allowed automatically | Stateless: reply traffic needs its own rule |
| Rule order | All rules evaluated together | Lowest number first, first match wins |
| Default | Blocks all inbound | Default NACL allows everything |

**Why the private group's rule uses the bastion's security group and not my IP:** the private server only ever sees connections coming from the bastion, not from my laptop. A rule that references the bastion's security group also keeps working if the bastion's IP changes.

## NACL experiment

1. Added inbound rule **90**: Deny, SSH (22), source = public subnet CIDR.
2. SSH from the bastion to the private server stopped working, even though the security group was still correct.
3. Deleted rule 90 and SSH worked again.

Rule 90 took effect because NACLs are read from the lowest number up and the first match wins. It was checked before the default allow rule (100). A deny numbered above 100 would never have applied.

## Troubleshooting: SSH fails

| Symptom | Likely cause | What to check |
|---|---|---|
| Connection timed out | Firewall or routing | Security group, NACL, whether my public IP changed, correct IP used (public for bastion, private for the private server) |
| Permission denied (publickey) | Key or user | Correct `.pem` file, username `ubuntu` |
| Unprotected private key file | File permissions | Restrict the key file (`chmod 400`, or `icacls` on Windows) |

## Key takeaways

- A bastion is the only machine with a public address, so there is only one door to protect and monitor.
- Private subnet means no route to an Internet Gateway. The name alone does not make a subnet private.
- A security group with no matching rule silently drops traffic, which shows up as a timeout and not as "refused".
- Stateful vs stateless is the main difference between security groups and NACLs.
- Without a NAT gateway, a private server cannot reach the internet. That is by design.

## Cleanup

- Terminated `day3-bastion` and `day3-private`.
- No NAT Gateway or Elastic IP was created, so nothing is billed by the hour.
- VPC, subnets, route tables, Internet Gateway and security groups are free and were kept.

## Not done

- Security group experiment (delete the SSH rule, observe the timeout, restore it). The NACL experiment covered the timeout case.
