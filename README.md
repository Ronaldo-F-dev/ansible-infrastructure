# Ansible Infrastructure

Reusable Ansible infrastructure for provisioning, securing, configuring, and deploying servers across multiple environments.

The repository is designed as a reusable DevOps foundation rather than configuration for a single server or application.

## Project Status

Work in progress.

The infrastructure currently supports:

- server bootstrap
- common system configuration
- human user management
- SSH hardening
- UFW firewall management
- Nginx web server and reverse proxy configuration
- Docker Engine and Docker Compose installation
- PostgreSQL installation and database management
- encrypted secret management with Ansible Vault

Application deployment, additional databases, backups, advanced deployment strategies, and CI/CD capabilities will be added progressively.

## Repository Structure

```text
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
│   ├── docker/
│   └── postgresql/
├── vault/
├── ansible.cfg
├── requirements.yml
├── .gitignore
└── README.md
```

The repository separates reusable automation logic from environment-specific configuration and encrypted secrets.

```text
roles/
   |
   +---- reusable infrastructure logic

inventories/
   |
   +---- environment-specific configuration

vault/
   |
   +---- encrypted secrets
```

## Documentation

The root README provides the global architecture and usage of the repository.

More detailed documentation is maintained close to the component it describes.

| Component | Documentation |
| --- | --- |
| Nginx | [`roles/nginx/README.md`](roles/nginx/README.md) |
| Docker | [`roles/docker/README.md`](roles/docker/README.md) |
| PostgreSQL | [`roles/postgresql/README.md`](roles/postgresql/README.md) |
| Ansible Vault | [`vault/README.md`](vault/README.md) |

Role-specific behavior, variables, requirements, security considerations, and examples should be documented in the corresponding role README rather than duplicated in this file.

As additional roles become substantial enough to require dedicated documentation, they should be added to this documentation index.

## Environments

Three environments are prepared:

```text
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
- PostgreSQL configuration
- application configuration

This allows the same roles to be reused without hardcoding production-specific values inside them.

Production currently contains the most complete configuration.

Environment-specific secret separation will be expanded as the dev and staging environments are implemented further.

## Requirements

The Ansible control node requires:

- Ansible
- SSH access to managed servers
- appropriate SSH keys
- Git

Required Ansible Collections are declared in:

```text
requirements.yml
```

Install them with:

```bash
ansible-galaxy collection install -r requirements.yml
```

The project currently uses Collections including:

```text
ansible.posix
community.general
community.postgresql
```

Declaring dependencies in `requirements.yml` ensures that a new control node can install the Ansible dependencies required by the repository.

## Inventory

Production inventory is currently located at:

```text
inventories/prod/hosts.ini
```

Example:

```ini
[web]
web-01 ansible_host=example.com ansible_port=22 ansible_user=ansible
```

Inventory groups describe the function of servers rather than application implementation details.

Environment variables are stored under the corresponding inventory directories.

For example:

```text
inventories/prod/group_vars/
inventories/prod/host_vars/
```

## Bootstrap

A new server may initially require access through the provider's administrative account.

The bootstrap playbook creates the dedicated account used for subsequent Ansible automation.

Example:

```bash
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

The main production infrastructure configuration is applied with:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml \
  --ask-vault-pass
```

The Vault password is required because the production infrastructure currently loads encrypted production variables.

The main playbook progressively applies:

```text
common
   ↓
users
   ↓
ssh
   ↓
firewall
   ↓
docker
   ↓
postgresql
   ↓
nginx
```

Individual components may also have dedicated playbooks for focused configuration and testing.

## Check Mode

Before making infrastructure changes, a dry run can be useful:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml \
  --check \
  --ask-vault-pass
```

Check mode predicts changes where supported.

It is not a complete replacement for testing because some tasks depend on resources that would only be created during a real execution.

A successful check mode run also does not automatically prove that a playbook is correctly designed for a completely new server. Task dependencies and first-run behavior must also be considered.

## Roles

### bootstrap

Prepares a new server for Ansible management.

Responsibilities include creating the dedicated Ansible administration account, installing its SSH key, and configuring required sudo privileges.

### common

Applies common base configuration and packages required across managed servers.

This includes dependencies needed by other roles, such as ACL support required when Ansible switches between unprivileged system users.

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

Detailed documentation:

[`roles/nginx/README.md`](roles/nginx/README.md)

### docker

Installs Docker Engine and Docker Compose from Docker's official Ubuntu repository.

The role also supports optional Docker group membership and validates supported host platforms.

Detailed documentation:

[`roles/docker/README.md`](roles/docker/README.md)

### postgresql

Installs and manages PostgreSQL.

The role currently supports:

- PostgreSQL package installation
- service management
- PostgreSQL application roles
- encrypted application passwords
- database creation
- database ownership
- idempotent database configuration

Application roles are created before databases so databases can safely reference their intended owners on a new server.

Detailed documentation:

[`roles/postgresql/README.md`](roles/postgresql/README.md)

## Variables

Reusable defaults belong inside roles:

```text
roles/<role>/defaults/main.yml
```

Environment-specific values belong inside inventories:

```text
inventories/prod/group_vars/
inventories/prod/host_vars/
```

This separation is fundamental to keeping roles reusable.

For example:

```text
roles/docker/defaults/main.yml
        |
        +---- docker_users: []

inventories/prod/group_vars/all/docker.yml
        |
        +---- docker_users:
                - example_admin
```

The role provides generic behavior while the inventory defines the environment-specific desired state.

The same principle applies to PostgreSQL:

```text
roles/postgresql/defaults/main.yml
        |
        +---- generic PostgreSQL defaults

inventories/prod/group_vars/all/postgresql.yml
        |
        +---- production databases and roles
```

## Secret Management

Secrets must not be stored in plaintext inventory variables.

Production secrets are stored encrypted with Ansible Vault:

```text
vault/prod.yml
```

The encrypted Vault file may be versioned.

The password used to decrypt the Vault must never be committed.

For example:

```bash
ansible-vault edit vault/prod.yml
```

Playbooks requiring the production Vault can be executed with:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml \
  --ask-vault-pass
```

Sensitive Ansible tasks should additionally use `no_log: true` where appropriate so decrypted secrets do not appear in task output.

Detailed Vault documentation:

[`vault/README.md`](vault/README.md)

## PostgreSQL Architecture

The current PostgreSQL configuration follows a dedicated application database model.

Example:

```text
PostgreSQL
│
├── postgres
│   └── administrative role
│
└── application_database
    └── owner: application_role
```

Application roles should not receive global administrative privileges unless explicitly required.

In particular, application roles do not require privileges such as:

```text
SUPERUSER
CREATEDB
CREATEROLE
```

for normal use of their dedicated database.

PostgreSQL currently listens locally.

Typical same-server communication therefore follows:

```text
Application
     |
     | TCP
     v
127.0.0.1:5432
     |
     v
PostgreSQL
```

Port `5432` should not be exposed publicly when remote database access is unnecessary.

## Security Principles

This repository follows several infrastructure security principles:

- SSH key authentication is preferred over passwords.
- Direct root administration is avoided after bootstrap.
- Privilege escalation is explicit.
- Host key checking remains enabled.
- Firewall rules expose only required services.
- Internal application ports should not be publicly exposed unnecessarily.
- PostgreSQL should remain private unless remote access is explicitly required.
- Application database roles follow least-privilege principles.
- Docker group membership is treated as privileged access.
- Secrets are never committed in plaintext.
- Ansible Vault protects secrets stored in the repository.
- Sensitive task output is protected with `no_log` where appropriate.
- Security changes are validated before removing existing administrative access.

## Docker and Virtualization

Docker installation and Docker container execution are separate concerns.

Docker Engine can install successfully while the underlying virtualization provider prevents nested container execution.

This is especially relevant on LXC-based VPS infrastructure.

For example:

```text
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

See [`roles/docker/README.md`](roles/docker/README.md) for Docker-specific documentation.

## Nginx Architecture

A typical application deployment will eventually follow:

```text
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
PostgreSQL / other database
```

Application backend ports should normally remain private when Nginx provides the public reverse proxy.

See [`roles/nginx/README.md`](roles/nginx/README.md) for Nginx-specific configuration.

## Idempotence

Roles are designed to describe desired state rather than execute arbitrary imperative scripts.

A correctly configured server should therefore produce minimal or no changes when the same playbook is executed again.

Example:

```text
ok=10
changed=0
failed=0
```

Unexpected changes on every execution should be investigated rather than automatically accepted.

Database users, databases, package installation, services, firewall configuration, and other managed resources should converge toward their declared state.

## Validation

Check playbook syntax before deployment:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml \
  --syntax-check \
  --ask-vault-pass
```

Inspect the inventory:

```bash
ansible-inventory \
  -i inventories/prod/hosts.ini \
  --graph
```

Test connectivity:

```bash
ansible all \
  -i inventories/prod/hosts.ini \
  -m ansible.builtin.ping
```

Check the repository for whitespace errors before committing:

```bash
git diff --check
```

Infrastructure changes should also be tested for idempotence whenever possible.

## Git Workflow

Infrastructure changes should be developed incrementally.

The recommended workflow is:

```text
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
verify real state
    |
    v
verify idempotence
    |
    v
update documentation
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

Documentation is considered part of the infrastructure implementation and should be updated before the corresponding feature is committed.

## Roadmap

Planned capabilities include:

- application deployment
- Docker Compose application deployment
- MySQL
- SQLite
- backups
- environment-specific Vault organization
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

Application source code does not need to live inside this repository.

Applications may later be deployed from:

```text
Git repositories
local build artifacts
container registries
```

while Ansible remains responsible for orchestrating infrastructure and deployment.

The long-term objective is to keep the infrastructure reusable enough that a new project or server can be introduced primarily through inventory variables rather than by rewriting the infrastructure roles.
