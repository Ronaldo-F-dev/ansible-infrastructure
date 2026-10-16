# Docker Role

Installs and configures Docker Engine and Docker Compose on supported Ubuntu hosts using Docker's official APT repository.

The role is designed to be reusable across environments and keeps environment-specific configuration outside the role itself.

## Features

- Validates the target operating system and CPU architecture.
- Installs Docker APT dependencies.
- Configures Docker's official signing key.
- Configures Docker's official APT repository.
- Installs Docker Engine.
- Installs Docker CLI.
- Installs containerd.
- Installs Docker Buildx.
- Installs the Docker Compose plugin.
- Enables and starts the Docker service.
- Optionally grants selected users access to the Docker daemon.
- Avoids unnecessary APT cache updates when the repository configuration has not changed.

## Supported Platforms

The role currently supports:

- Ubuntu on `x86_64` (`amd64`)
- Ubuntu on `aarch64` (`arm64`)

Unsupported platforms are rejected before Docker configuration begins.

## Requirements

The managed host must have:

- Ubuntu
- Python available for Ansible
- Internet access to Docker's official package repository
- A user with privilege escalation through `become`

The Ansible control node must be able to connect to the managed host through SSH.

## Role Variables

Default variables are defined in:

```text
roles/docker/defaults/main.yml
```

### APT dependencies

```yaml
docker_apt_dependencies:
  - ca-certificates
  - curl
  - python3-debian
```

Packages required to configure Docker's official repository.

### Docker signing key

```yaml
docker_apt_key_url: https://download.docker.com/linux/ubuntu/gpg
docker_apt_key_path: /etc/apt/keyrings/docker.asc
```

Defines the source and destination of Docker's repository signing key.

### Docker repository

```yaml
docker_apt_repository_url: https://download.docker.com/linux/ubuntu
```

Docker's official Ubuntu package repository.

### Architecture mapping

```yaml
docker_architecture_map:
  x86_64: amd64
  aarch64: arm64
```

Maps Ansible architecture names to the architecture names expected by Docker's repository.

### Docker packages

```yaml
docker_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-buildx-plugin
  - docker-compose-plugin
```

Packages installed by the role.

### Docker service

```yaml
docker_service_name: docker
```

Name of the Docker systemd service.

### Docker users

```yaml
docker_users: []
```

Users listed here are added to the `docker` group.

For example:

```yaml
docker_users:
  - ronaldo
```

Environment-specific values should normally be defined in inventory variables rather than modifying the role defaults.

Example:

```text
inventories/prod/group_vars/all/docker.yml
```

```yaml
---
docker_users:
  - ronaldo
```

## Security Warning

Membership in the `docker` group grants significant control over the Docker daemon and should effectively be treated as root-level access.

Do not automatically add every system user to this group.

Ansible itself does not require its administration account to belong to the `docker` group when Docker operations are performed with privilege escalation.

After adding a user to the Docker group, the user should open a new login session before testing Docker access.

## Example Playbook

```yaml
---
- name: Configure Docker
  hosts: all
  become: true

  roles:
    - docker
```

Run it with:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/docker.yml
```

Check mode:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/docker.yml \
  --check
```

## Validation

Check the installed Docker version:

```bash
docker --version
```

Check Docker Compose:

```bash
docker compose version
```

Check the Docker service:

```bash
systemctl is-active docker
systemctl is-enabled docker
```

Check access to the Docker daemon:

```bash
docker ps
```

On a host capable of running Docker containers, an end-to-end test can be performed with:

```bash
docker run --rm hello-world
```

## Idempotence

The role is designed to be idempotent.

After the host has been configured, running the playbook again should normally produce:

```text
changed=0
failed=0
```

The APT cache is refreshed after the Docker signing key or repository configuration changes instead of being refreshed unnecessarily on every execution.

## LXC and Nested Containerization

Installing Docker successfully does not guarantee that the underlying virtualization platform permits Docker containers to run.

This is particularly important for VPS instances running inside LXC containers.

For example, Docker Engine may install successfully and the daemon may start correctly while container creation fails with an OCI runtime error such as:

```text
error mounting "proc" to rootfs at "/proc": permission denied
```

This can occur when the LXC host does not permit the operations required for nested containerization.

In this situation, the restriction must generally be addressed at the virtualization provider or Proxmox/LXC host level. It should not be bypassed by weakening the security configuration of the managed VPS.

A virtual machine based on technologies such as KVM generally provides stronger isolation from these host-level container restrictions.

## Dependencies

This role does not depend on another Ansible role.

It uses modules from `ansible-core`.

## Scope

This role configures the Docker platform only.

Application deployment does not belong in this role. Application containers, Compose projects, images, health checks, deployment strategies, and rollbacks should be handled by dedicated application deployment roles or playbooks.