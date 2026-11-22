# Day 15: Ansible Setup

Set up Ansible on a control node and managed two worker VMs over SSH, with an inventory, ad-hoc commands, and a drill on `UNREACHABLE` errors.

## Setup

- **Control node:** existing Ubuntu EC2 VM (Ansible `2.20.1`)
- **Managed nodes:** 2 × Ubuntu `t3.micro` EC2 instances in the same VPC
- **Access:** key-based SSH from the control node. The workers' security group allows SSH only from the control node's security group.

## Passwordless SSH

1. Generated a key on the control node: `ssh-keygen -t ed25519`
2. Added the public key to `~/.ssh/authorized_keys` on each worker.
3. Checked it: `ssh ubuntu@<worker-ip> hostname` with no password prompt.

## Inventory

`inventory.ini`:

```ini
[workers]
worker1 ansible_host=<worker-1-private-ip>
worker2 ansible_host=<worker-2-private-ip>

[workers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

`ansible.cfg` points at it, so commands need no `-i`:

```ini
[defaults]
inventory = inventory.ini
```

The `ansible_python_interpreter` line removes the interpreter-discovery warning.

## Result

```bash
ansible all -m ping
```

Both workers returned `"ping": "pong"`. The Ansible ping module is not ICMP. It logs in over SSH and checks that Python works on the target, so it tests the whole chain: network, key, user and Python.

## Ad-hoc commands

```bash
ansible workers -a "uptime"
ansible workers -m apt -a "name=htop state=present" --become
ansible workers -m service -a "name=nginx state=started" --become
```

- `-a` runs a command on every host in the group. `--become` runs it as root.
- `htop` returned `changed: false` on both runs because it was already installed. A module checks the **state** first and acts only if needed. This is idempotency.
- `service ... nginx state=started` **failed**: `Could not find the requested service`. nginx was not installed. Ansible does not guess, so a service must exist before it can be started.

## Troubleshooting drill: `UNREACHABLE`

| Symptom | Cause | Fix |
|---|---|---|
| `ssh-copy-id`: `Permission denied (publickey)` | Password login was off and the key was not on the worker | Added the control node's public key to `authorized_keys` using the key pair's login |
| `UNREACHABLE ... wronguser@<ip>: Permission denied (publickey)` | Wrong `ansible_user` in the inventory | Restored `ansible_user=ubuntu` |
| `Host key verification failed` or a `yes/no` prompt | The worker's host key is not in `known_hosts` | Connect once with `ssh` and accept the key |
| `timed out` | Security group does not allow SSH from the control node, or the machine is stopped | Allow port 22 from the control node's security group, or start the machine |

**Reading the error:** `UNREACHABLE` means the login failed and no task ran. `FAILED!` means the login worked and the task failed.

## Lessons learned

- `authorized_keys` lines must be **appended** with `>>`. A single `>` overwrites the file and can lock you out of the instance.
- Never run `rm -rf` on a server without checking the full path first. Run `ls <path>` before deleting.
- Use dedicated SSH keys and keep private keys out of Git.

## Cleanup

Stop or terminate the two worker instances when finished, because they bill by the hour.
