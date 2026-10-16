# Ansible Infrastructure

Reusable Ansible infrastructure for provisioning, securing, configuring, and deploying servers across multiple environments.

The repository is designed as a reusable DevOps foundation rather than configuration for a single server or application.

## Project Status

Work in progress.

The infrastructure currently supports server bootstrap, common system configuration, user management, SSH hardening, firewall management, Nginx, and Docker.

Additional application deployment, database, backup, secret management, and CI/CD capabilities will be added progressively.

## Repository Structure

```text id="2ugf57"
ansible-infrastructure/
├── inventories/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── playbooks/
├── roles/
│   ├── bootstrap/
│   ├── common/
│   ├── users/
│   ├── ssh/
│   ├── firewall/
│   ├── nginx/
│   └── docker/
├── ansible.cfg
├── requirements.yml
├── .gitignore
└── README.md
```

The repository separates reusable automation logic from environment-specific configuration.

```text id="wufymd"
roles/
   |
   +---- reusable infrastructure logic

inventories/
   |
   +---- environment-specific configuration
```

## Environments

Three environments are prepared:

```text id="klpsa5"
inventories/
├── dev/
├── staging/
└── prod/
```

Each environment can define its own:

- servers
- groups
- variables
- users
- firewall rules
- Nginx sites
- Docker configuration
- application configuration

This allows the same roles to be reused without hardcoding production-specific values inside them.

## Requirements

The Ansible control node requires:

- Ansible
- SSH access to managed servers
- appropriate SSH keys
- Git

Install required Ansible collections with:

```bash id="gnt1vc"
ansible-galaxy collection install -r requirements.yml
```

## Inventory

Production inventory is currently located at:

```text id="ehpxbm"
inventories/prod/hosts.ini
```

Example:

```ini id="0bqx82"
[web]
web-01 ansible_host=example.com ansible_port=22 ansible_user=ansible
```

Inventory groups describe the function of servers rather than application implementation details.

## Bootstrap

A new server may initially require access through the provider's administrative account.

The bootstrap playbook creates the dedicated account used for subsequent Ansible automation.

Example:

```bash id="7yppzh"
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/bootstrap.yml \
  -e ansible_user=root
```

The bootstrap process creates a dedicated Ansible administration account, installs its SSH public key, and grants the privileges required for infrastructure automation.

After bootstrap, normal infrastructure management should use the dedicated `ansible` account instead of connecting directly as root.

Before disabling root SSH access, always verify that the dedicated administration account can:

1. connect through SSH
2. authenticate using its SSH key
3. use privilege escalation successfully

## Main Infrastructure Playbook

The main infrastructure configuration is applied with:

```bash id="ytrzqx"
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml
```

The main playbook progressively applies reusable roles to managed servers.

## Check Mode

Before making infrastructure changes, a dry run can be useful:

```bash id="ycf0xv"
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml \
  --check
```

Check mode predicts changes where supported.

It is not a complete replacement for testing because some tasks depend on resources that would only be created during a real execution.

## Roles

### bootstrap

Prepares a new server for Ansible management.

Responsibilities include creating the dedicated Ansible administration account, installing its SSH key, and configuring required sudo privileges.

### common

Applies common base configuration and packages required across managed servers.

### users

Manages human system users, SSH public keys, and administrative privileges.

Human accounts are managed independently from the dedicated Ansible automation account.

### ssh

Hardens SSH configuration.

Security changes should only be applied after alternate administrative access has been successfully verified.

### firewall

Manages host firewall rules through UFW.

Publicly exposed ports should be explicitly defined rather than opening application ports unnecessarily.

### nginx

Installs and configures Nginx.

The role supports static sites, reverse proxies, multiple virtual hosts, explicit `present`/`absent` lifecycle management, configuration validation, and controlled reloads.

See:

```text id="h24n7e"
roles/nginx/README.md
```

### docker

Installs Docker Engine and Docker Compose from Docker's official Ubuntu repository.

The role also supports optional Docker group membership and validates supported host platforms.

See:

```text id="bf1n8f"
roles/docker/README.md
```

## Variables

Reusable defaults belong inside roles:

```text id="bq7c24"
roles/<role>/defaults/main.yml
```

Environment-specific values belong inside inventories:

```text id="dfidq7"
inventories/prod/group_vars/
inventories/prod/host_vars/
```

This separation is fundamental to keeping roles reusable.

For example:

```text id="9yxx98"
roles/docker/defaults/main.yml
        |
        +---- docker_users: []

inventories/prod/group_vars/all/docker.yml
        |
        +---- docker_users:
                - ronaldo
```

The role provides the generic behavior while the inventory defines the production-specific desired state.

## Security Principles

This repository follows several infrastructure security principles:

- SSH key authentication is preferred over passwords.
- Direct root administration is avoided after bootstrap.
- Privilege escalation is explicit.
- Host key checking remains enabled.
- Firewall rules expose only required services.
- Internal application ports should not be publicly exposed unnecessarily.
- Docker group membership is treated as privileged access.
- Secrets should never be committed in plaintext.
- Security changes are validated before removing existing administrative access.

## Docker and Virtualization

Docker installation and Docker container execution are separate concerns.

Docker Engine can install successfully while the underlying virtualization provider prevents nested container execution.

This is especially relevant on LXC-based VPS infrastructure.

For example:

```text id="2i9sx5"
Physical host
     |
     v
Proxmox
     |
     v
LXC VPS
     |
     v
Docker
     |
     v
Application containers
```

The LXC host must permit the operations required by Docker.

Provider-level restrictions should not be bypassed by weakening the security configuration of the managed VPS.

## Nginx Architecture

A typical application deployment will eventually follow:

```text id="c0bcpl"
Internet
   |
   v
Firewall
   |
   v
Nginx :80/:443
   |
   v
Application
   |
   v
Database
```

Application backend ports should normally remain private when Nginx provides the public reverse proxy.

## Idempotence

Roles are designed to describe desired state rather than execute arbitrary imperative scripts.

A correctly configured server should therefore produce minimal or no changes when the same playbook is executed again.

Example:

```text id="y1u0y5"
ok=10
changed=0
failed=0
```

Unexpected changes on every execution should be investigated rather than automatically accepted.

## Validation

Check playbook syntax before deployment:

```bash id="7ukmrv"
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml \
  --syntax-check
```

Inspect the inventory:

```bash id="v9clwp"
ansible-inventory \
  -i inventories/prod/hosts.ini \
  --graph
```

Test connectivity:

```bash id="1ququf"
ansible all \
  -i inventories/prod/hosts.ini \
  -m ansible.builtin.ping
```

## Git Workflow

Infrastructure changes should be developed incrementally.

The recommended workflow is:

```text id="s1h3jq"
understand
    |
    v
implement
    |
    v
syntax check
    |
    v
check / test
    |
    v
verify idempotence
    |
    v
review diff
    |
    v
commit
    |
    v
push
```

Commits should be small, understandable, and written in English using Conventional Commit style where appropriate.

## Roadmap

Planned capabilities include:

- application deployment
- Docker Compose application deployment
- PostgreSQL
- MySQL
- SQLite
- backups
- Ansible Vault and secret management
- tags
- structured error handling with `block`, `rescue`, and `always`
- health checks
- rolling deployments
- rollback strategies
- load balancing
- CI/CD
- additional reusable roles

## Important

This repository manages infrastructure.

Application source code does not need to live inside this repository. Applications may later be deployed from Git repositories, local build artifacts, or container registries while Ansible remains responsible for orchestrating the infrastructure and deployment process.