# Day 16: Ansible Playbooks

Wrote an idempotent playbook that installs nginx on two worker VMs, deploys a templated page and nginx config, and restarts nginx through a handler only when the config changes.

## Files

| File | Purpose |
|---|---|
| `nginx.yml` | The playbook |
| `templates/index.html.j2` | Web page template (shows the host name and port) |
| `templates/demo.conf.j2` | nginx site config template (listen port from a variable) |
| `inventory.ini`, `ansible.cfg` | Inventory with the 2 workers (from Day 15) |

## What the playbook does

1. Installs nginx (`apt`, `state: present`)
2. Creates the web root `/var/www/demo`
3. Deploys the page from a Jinja2 template
4. Deploys the nginx config from a template and **notifies a handler**
5. Makes sure nginx is started and enabled at boot

The handler `Restart nginx` runs once at the end of the play, and only if the config task changed something.

Variables (`app_port`, `site_message`) are set in the playbook and can be overridden with `-e`.

## Run it

```bash
yamllint nginx.yml
ansible-playbook nginx.yml --syntax-check
ansible-playbook nginx.yml --check --diff     # dry run
ansible-playbook nginx.yml
ansible workers -m uri -a "url=http://localhost:8080 return_content=yes"
```

## Results

| Run | Recap | Notes |
|---|---|---|
| First run | `ok=7 changed=5 failed=0` | Install, web root, page, config, handler restart |
| Second run | `ok=6 changed=0 failed=0` | No handler. **Idempotent** |
| `-e site_message="Updated by Ansible"` | `changed=1` | Only the page task changed. The handler did not run, because the nginx config was unchanged |

The check on each worker returned HTTP `200` and a page showing its own host name, so one template produced different content per host.

## Concepts

- **Playbook:** YAML list of tasks for a group of hosts. It can be kept in Git, unlike ad-hoc commands.
- **Idempotency:** a task checks state first and acts only if needed, so a re-run changes nothing.
- **Template (`.j2`):** a file with `{{ variables }}` filled in per host.
- **Handler:** a task that runs only when notified by a task that reported `changed`, and only once per play. This avoids needless restarts.
- **`become: true`:** run tasks as root (sudo).
- **`--check --diff`:** a dry run that shows what would change.

## Troubleshooting drills

| Symptom | Cause | Fix |
|---|---|---|
| `Could not open lock file /var/lib/dpkg/lock-frontend - Permission denied ... are you root?` | `become` was disabled, so apt ran as an ordinary user. It only fails when a change is needed. With nginx already installed the same task passed | Set `become: true` |
| `YAML parsing failed: Mapping values are not allowed in this context` at line 14, column 21 | One line (`state: present`) was indented two spaces too few. The parser reports the **next** line, where the structure stops making sense | Fix the indent. Catch it early with `ansible-playbook --syntax-check` and `yamllint` |

**Reading a failure:** Ansible prints the file, line and column of the failing task (`Origin:`). `FAILED!` means the login worked and the task failed. `UNREACHABLE` (Day 15) means the login failed.

## Cleanup

Stop or terminate the two worker instances when finished, because they bill by the hour.
