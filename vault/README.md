# Ansible Vault

This directory contains encrypted secrets used by the infrastructure playbooks.

## Production Secrets

Production secrets are stored in:

```text
vault/prod.yml
```

The file is encrypted with Ansible Vault and can safely be versioned while encrypted.

Never commit an unencrypted version of this file.

## Editing Secrets

Use:

```bash
ansible-vault edit vault/prod.yml
```

## Viewing Secrets

When necessary:

```bash
ansible-vault view vault/prod.yml
```

Avoid displaying secrets unnecessarily in terminals, logs, screenshots, or CI output.

## Running Playbooks

Playbooks requiring production secrets can be executed with:

```bash
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml \
  --ask-vault-pass
```

## Vault Password

The password used to decrypt Ansible Vault is separate from the passwords stored inside the Vault.

The Vault password must never be committed to this repository.

Local Vault password files are excluded through `.gitignore`.

Examples of ignored filenames include:

```text
vault-password
vault-password.txt
production.vault-password
```



CI/CD systems should obtain the Vault password from their secure secret-management mechanism rather than storing it in the repository.


## MySQL Secrets

MySQL application passwords are stored in the encrypted production Vault.

Expected structure:

```yaml
vault_mysql_users:
  application_user:
    password: "secret"

The MySQL role retrieves passwords dynamically using the configured MySQL username.
For example, an inventory user:
mysql_users:
  - name: application_user
    host: localhost
    database: application_database
    privileges: "ALL"
    state: present

uses the corresponding Vault entry:
vault_mysql_users.application_user.password

PostgreSQL and MySQL credentials should use different passwords, even when the application username is the same.
Never store database passwords directly in inventory variables, role defaults, playbooks, or documentation.

Le point important est que nous séparons maintenant clairement :

```text
inventories/
│
├── postgresql.yml ─── configuration non secrète
└── mysql.yml      ─── configuration non secrète

vault/prod.yml
│
├── vault_postgresql_users
│      └── ansible_demo_app.password
│
└── vault_mysql_users
       └── ansible_demo_app.password