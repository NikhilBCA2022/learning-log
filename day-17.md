# Day 17 – Ansible Roles and Galaxy

## Project Overview

This project demonstrates Ansible Roles and Ansible Galaxy by automating Nginx installation and configuration across multiple remote worker nodes.

The project uses a reusable Nginx role to install Nginx, create a web directory, deploy a custom HTML page, configure the Nginx server, and restart the service only when configuration changes.

## Technologies Used

- Ansible
- Ansible Galaxy
- Ubuntu Linux
- Nginx
- YAML
- Jinja2 Templates
- Git

## Project Structure

```text
ansible-lab/
│
├── ansible.cfg
├── inventory.ini
├── site.yml
├── requirements.yml
├── README.md
├── .gitignore
│
└── roles/
    ├── nginx/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── handlers/
    │   │   └── main.yml
    │   ├── tasks/
    │   │   └── main.yml
    │   └── templates/
    │       ├── demo.conf.j2
    │       └── index.html.j2
    │
    └── geerlingguy.git/
```

## Ansible Roles

An Ansible Role is a reusable collection of tasks, handlers, variables, templates, and other resources organized in a standard directory structure.

### Nginx Role

The custom Nginx role performs the following operations:

1. Installs Nginx using the APT module.
2. Creates `/var/www/demo`.
3. Deploys an HTML page using a Jinja2 template.
4. Deploys an Nginx configuration file.
5. Restarts Nginx when its configuration changes.
6. Ensures Nginx is running and enabled.

### Role Components

| Directory | Purpose |
|---|---|
| defaults | Stores default variables |
| tasks | Contains automation tasks |
| handlers | Executes actions when notified |
| templates | Stores Jinja2 templates |

## Ansible Galaxy

Ansible Galaxy is a community platform for sharing and downloading reusable Ansible roles.

This project uses:

`geerlingguy.git`

This community role automates Git installation on worker nodes.

### Install Galaxy Dependencies

```bash
ansible-galaxy role install -r requirements.yml -p roles
```

## Execution

### 1. Syntax Check

```bash
ansible-playbook site.yml --syntax-check
```

### 2. Execute Playbook

```bash
ansible-playbook site.yml
```

### 3. Verify Idempotency

```bash
ansible-playbook site.yml
```

The second execution should report:

```text
changed=0
failed=0
```

This demonstrates Ansible idempotency.

### 4. Verify Git Installation

```bash
ansible workers -a "git --version"
```

Git version information should be displayed for both worker nodes.

### 5. Verify Nginx

```bash
ansible workers -a "curl -s http://localhost:8080"
```

The custom HTML page should be returned from both workers.

## Idempotency

Idempotency means executing the same automation multiple times produces the same desired system state without making unnecessary changes.

In this project:

- First execution: Nginx configuration and required files are created.
- Second execution: No unnecessary changes are reported.
- The handler runs only when the Nginx configuration changes.

## Learning Outcomes

After completing this project, the following concepts were practiced:

- Creating custom Ansible Roles.
- Understanding the standard role directory structure.
- Using default variables.
- Working with Jinja2 templates.
- Configuring handlers.
- Installing community roles through Ansible Galaxy.
- Managing role dependencies through `requirements.yml`.
- Understanding Ansible idempotency.
- Automating configurations across multiple worker nodes.

## Conclusion

This project demonstrates reusable infrastructure automation using Ansible Roles and Ansible Galaxy. It improves maintainability, reduces duplication, and makes configuration management easier across multiple servers.
