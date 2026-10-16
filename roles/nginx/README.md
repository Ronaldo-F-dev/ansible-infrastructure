# Nginx Role

Installs and configures Nginx web server sites and reverse proxies on managed Ubuntu servers.

The role supports multiple sites on the same server and provides explicit lifecycle management through `present` and `absent` states.

## Features

- Installs Nginx.
- Enables and starts the Nginx service.
- Manages multiple virtual hosts.
- Supports static websites.
- Supports reverse proxy configurations.
- Creates required document root directories.
- Manages Nginx `sites-available` configurations.
- Manages `sites-enabled` symlinks.
- Supports explicit site removal.
- Validates the complete Nginx configuration before reload.
- Reloads Nginx only when configuration changes.
- Keeps application backend ports private when Nginx acts as the public reverse proxy.

## Requirements

The managed host must have:

- Ubuntu or another compatible Debian-based system
- Python available for Ansible
- A user with privilege escalation through `become`
- Network access for package installation

The Ansible control node must be able to connect to the managed host through SSH.

## Role Variables

Role defaults are defined in:

```text id="v97ggr"
roles/nginx/defaults/main.yml
```

Environment-specific configuration should normally be stored under the corresponding inventory.

For production, for example:

```text id="fx6q09"
inventories/prod/group_vars/all/nginx.yml
```

## Site Configuration

Sites are managed as structured data instead of creating one role per website.

A site can represent either:

- a static website
- a reverse proxy

This allows the same role to manage several applications and domains on the same server.

### Static Site Example

```yaml id="9s2t3d"
nginx_sites:
  - name: ansible-demo
    state: present
    server_name: ansible.fr-host.fr
    listen_port: 80
    type: static
    root: /var/www/ansible-demo
```

Conceptually:

```text id="a0g3hn"
Client
  |
  v
Nginx :80
  |
  v
/var/www/ansible-demo
```

Nginx serves the files directly from the configured document root.

## Reverse Proxy Example

A reverse proxy site forwards incoming requests to an application running on an internal port.

Example structure:

```yaml id="6mkcyv"
nginx_sites:
  - name: my-api
    state: present
    server_name: api.example.com
    listen_port: 80
    type: proxy
    proxy_pass: http://127.0.0.1:3000
```

Conceptually:

```text id="3ypwws"
Internet
   |
   v
Nginx :80
   |
   v
127.0.0.1:3000
   |
   v
Application
```

The application backend port does not need to be publicly exposed when Nginx is the public entry point.

## Site Lifecycle

Sites have an explicit lifecycle.

### Present

```yaml id="4z8k9f"
state: present
```

The role ensures that the site configuration and required resources exist.

### Absent

```yaml id="p0j38x"
state: absent
```

The role removes the managed Nginx site configuration and enabled-site link.

When removing a site from infrastructure, prefer the following workflow:

```text id="5s1f2a"
state: present
      |
      v
state: absent
      |
      v
run Ansible
      |
      v
remove the site entry from inventory
```

Do not immediately delete a site definition from the inventory if Ansible still needs that definition to clean up resources previously created for it.

This explicit lifecycle prevents orphaned Nginx configurations.

## Multiple Sites

The role is designed to manage multiple sites on the same host.

For example:

```yaml id="z9v0q4"
nginx_sites:
  - name: frontend
    state: present
    server_name: example.com
    listen_port: 80
    type: static
    root: /var/www/frontend

  - name: api
    state: present
    server_name: api.example.com
    listen_port: 80
    type: proxy
    proxy_pass: http://127.0.0.1:3000
```

Nginx selects the appropriate virtual host primarily from the HTTP `Host` header.

This means multiple applications can share the same public ports while using different domain names.

## Configuration Validation

Configuration changes are not blindly reloaded.

The role uses a handler to validate the complete Nginx configuration before applying a reload.

The equivalent manual validation is:

```bash id="jgf45m"
nginx -t
```

The intended workflow is:

```text id="17spqw"
Ansible changes configuration
          |
          v
handler triggered
          |
          v
nginx -t
     |
     +---- valid ----> reload Nginx
     |
     +---- invalid --> fail
```

This reduces the risk of reloading Nginx with an invalid configuration.

## Handlers

Handlers are used because Nginx should not be reloaded when nothing has changed.

For example:

```text id="h6lyr3"
Playbook run #1
configuration changed
        |
        v
handler
        |
        v
reload

Playbook run #2
configuration unchanged
        |
        v
no handler execution
```

This contributes to idempotent infrastructure management.

## Example Playbook

```yaml id="68qlp5"
---
- name: Configure Nginx
  hosts: web
  become: true

  roles:
    - nginx
```

The role can also be executed as part of the main infrastructure playbook.

## Validation

Validate the Nginx configuration manually with:

```bash id="k5cw5b"
sudo nginx -t
```

Check the service:

```bash id="nrm7zh"
systemctl is-active nginx
systemctl is-enabled nginx
```

Inspect listening ports:

```bash id="goxqpr"
ss -lntp
```

Test a local virtual host:

```bash id="6jnnvw"
curl -H "Host: example.com" http://127.0.0.1
```

Using the `Host` header is useful when DNS or an upstream proxy has not yet been configured.

## Firewall Considerations

For a typical reverse proxy architecture, the firewall should expose only the public Nginx ports required by the application, commonly:

```text id="cv8x1b"
22/tcp   SSH
80/tcp   HTTP
443/tcp  HTTPS
```

An internal application listening on a port such as `3000`, `8000`, or `8080` does not normally need that port exposed publicly when Nginx proxies requests to it locally.

Firewall configuration belongs to the dedicated firewall role rather than this Nginx role.

## Upstream Proxies

A hosting provider or external reverse proxy may sit in front of Nginx.

In that situation, the request path can look like:

```text id="48uk8g"
Internet
   |
   v
Provider proxy / gateway
   |
   v
Nginx
   |
   v
Application
```

The external proxy and the Nginx configuration must agree on the expected domain and destination port.

## Idempotence

The role is designed to be idempotent.

Once the desired configuration is already applied, subsequent executions should not rewrite unchanged configuration or unnecessarily reload Nginx.

A stable execution should normally finish with no unexpected changes.

## Scope

This role manages Nginx itself.

It does not deploy application source code, build applications, manage application containers, configure databases, or manage DNS records.

Those responsibilities belong to dedicated infrastructure or application deployment roles.