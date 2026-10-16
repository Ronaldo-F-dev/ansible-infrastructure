# SQLite Role

This Ansible role installs SQLite and manages local SQLite database files on Debian-based systems.

Unlike PostgreSQL and MySQL, SQLite does not run as a database server. Applications access the database directly through a file on the local filesystem.

## Features

- Installs the SQLite CLI package
- Creates database directories
- Initializes SQLite database files
- Manages database ownership and permissions
- Supports multiple SQLite databases
- Supports Ansible check mode
- Provides idempotent database creation

## Requirements

The role currently targets Debian-based systems using the `ansible.builtin.apt` module.

No external Ansible collection is required.

## Variables

Default variables are defined in:

```text
roles/sqlite/defaults/main.yml


Default configuration:

sqlite_packages:
  - sqlite3

sqlite_databases: []

sqlite_packages:
  - sqlite3

sqlite_databases: []

Databases can be configured from inventory variables:
sqlite_databases:
  - name: ansible_demo
    path: /var/lib/ansible-demo/ansible_demo.db
    owner: ronaldo
    group: ronaldo
    directory_mode: "0750"
    mode: "0640"
    state: present

Database options
- name: descriptive database name used by Ansible
- path: absolute path to the SQLite database file
- owner: Linux user owning the database
- group: Linux group owning the database
- directory_mode: permissions applied to the parent directory
- mode: permissions applied to the database file
- state: desired database state; currently present is supported
Filesystem permissions
SQLite databases are filesystem resources.
The application must have the required permissions not only on the database file but also on its parent directory.
SQLite may create additional files during transactions, including:
database.db-journal
database.db-wal
database.db-shm

Avoid world-writable permissions such as 0777.
Use dedicated users or groups and grant only the permissions required by the application.
Database initialization
The role initializes a database with:
sqlite3 <database> 'PRAGMA user_version;'

The Ansible creates parameter prevents the initialization command from running again when the database already exists.
Application schemas and migrations are intentionally not managed by this role. They should be handled by the application deployment process or migration framework.
Secrets
SQLite does not provide database users or password authentication like PostgreSQL or MySQL.
Access is controlled primarily through Linux filesystem permissions.
Therefore this role does not require Ansible Vault credentials.
Check mode
The role checks whether database files already exist before managing their permissions.
On a fresh server, Ansible can predict the package, directory, and database creation while skipping permission management for a database file that does not physically exist yet.
Usage
Run the dedicated playbook:
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/sqlite.yml

Dry-run:
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/sqlite.yml \
  --check

Idempotence
After the desired state has been applied, subsequent executions should complete with:
changed=0
failed=0

Database contents are preserved. Existing databases are not recreated or overwritten.

Il y a une décision importante dans ce README :

```text
state: present