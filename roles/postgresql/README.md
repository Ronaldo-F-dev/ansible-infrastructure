# PostgreSQL Role

Installs and configures PostgreSQL on managed Ubuntu servers.

The role manages PostgreSQL packages, the PostgreSQL service, application roles, and databases in an idempotent way.

## Features

- Installs PostgreSQL and PostgreSQL contrib packages
- Installs the Python PostgreSQL dependency required by Ansible
- Enables and starts the PostgreSQL service
- Creates PostgreSQL application roles
- Creates application databases
- Assigns database ownership
- Supports encrypted passwords through Ansible Vault
- Keeps PostgreSQL listening locally by default
- Supports idempotent execution

## Requirements

The `community.postgresql` collection is required.

Install project dependencies with:

```bash
ansible-galaxy collection install -r requirements.yml
```

The managed host also requires ACL support because Ansible switches from the `ansible` system user to the `postgres` system user when executing PostgreSQL modules.

The project's `common` role installs the required `acl` package.

## Default Variables

Default variables are defined in:

```text
roles/postgresql/defaults/main.yml
```

Example:

```yaml
postgresql_packages:
  - postgresql
  - postgresql-contrib
  - python3-psycopg2

postgresql_service_name: postgresql

postgresql_databases: []

postgresql_users: []
```

## PostgreSQL Users

Application roles can be declared in environment variables:

```yaml
postgresql_users:
  - name: ansible_demo_app
    state: present
```

Passwords are not stored in the inventory variables.

They are retrieved from encrypted Vault variables:

```yaml
vault_postgresql_users:
  ansible_demo_app:
    password: "secret"
```

The Vault file must remain encrypted when stored in Git.

## PostgreSQL Databases

Example:

```yaml
postgresql_databases:
  - name: ansible_demo
    owner: ansible_demo_app
    state: present
```

The PostgreSQL role is created before the database so that a newly created database can immediately be assigned to its intended owner.

## Security Model

Application PostgreSQL roles are not granted global administrative privileges by default.

An application role does not need:

- `SUPERUSER`
- `CREATEDB`
- `CREATEROLE`

For a dedicated application database, the application role can instead own its database.

Example:

```text
PostgreSQL
│
├── postgres
│   └── administration
│
└── ansible_demo
    └── owner: ansible_demo_app
```

PostgreSQL currently listens on localhost only.

Database port `5432` is therefore not exposed through the firewall.

Applications running on the same server can connect using:

```text
127.0.0.1:5432
```

Do not expose PostgreSQL publicly unless the infrastructure architecture explicitly requires remote database access.

## Vault

The production playbook loads encrypted variables from:

```text
vault/prod.yml
```

Run the PostgreSQL playbook with:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/postgresql.yml \
  --ask-vault-pass
```

The Vault password itself must never be committed to Git.

## Idempotence

Running the role repeatedly should result in no changes when the server already matches the declared configuration.

Expected result:

```text
changed=0
failed=0
```