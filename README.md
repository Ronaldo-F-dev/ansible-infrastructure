# Ansible Infrastructure

Reusable Ansible infrastructure for provisioning, configuring and deploying applications across multiple environments and servers.

## Status

## Bootstrap

New servers are initially bootstrapped using the provider's administrative user.

Example:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/bootstrap.yml \
  -e ansible_user=root

The bootstrap process creates a dedicated ansible administration account, installs its SSH public key and grants the privileges required for Ansible automation.
Normal infrastructure management should then use the dedicated ansible account instead of connecting directly as root.
