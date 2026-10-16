# MySQL Role

This role installs and configures MySQL on Debian-based managed hosts.

It provides declarative management of:

- MySQL server packages
- MySQL service state
- MySQL databases
- MySQL application users
- Database-scoped privileges
- Application passwords stored with Ansible Vault

## Requirements

The role requires the `ansible.mysql` collection.

Install repository dependencies with:

```bash
ansible-galaxy collection install -r requirements.yml

The managed host also requires a supported Debian-based operating system.
The role installs python3-pymysql automatically.
Variables
Default variables are defined in:
roles/mysql/defaults/main.yml

Example:
mysql_packages:
  - mysql-server
  - python3-pymysql

mysql_service_name: mysql

mysql_databases: []

mysql_users: []

Environment-specific configuration should be defined in inventory variables.
Example:
mysql_databases:
  - name: my_application
    state: present

mysql_users:
  - name: my_application
    host: localhost
    database: my_application
    privileges: "ALL"
    state: present

Secrets
Passwords must not be stored in inventory variables.
They are retrieved from Ansible Vault using the following structure:
vault_mysql_users:
  my_application:
    password: "secret"

The Vault file itself must remain encrypted.
Do not commit Vault passwords or unencrypted secret files.
Privilege Model
MySQL accounts are identified by both a username and a host.
For example:
my_application@localhost

A configuration such as:
database: my_application
privileges: "ALL"

grants privileges on:
my_application.*

It does not grant global administrative privileges on *.*.
Administrative Access
The role manages MySQL locally through the Unix socket:
/var/run/mysqld/mysqld.sock

This avoids requiring a MySQL root password for Ansible automation.
Network Security
The current production MySQL installation listens locally on:
127.0.0.1:3306

MySQL is not exposed through the firewall.
Applications running on the same server can connect using the configured application credentials.
Remote database access should only be enabled intentionally and with appropriate network and firewall controls.
Check Mode
The role supports Ansible check mode.
On a fresh server, package installation can only be simulated. Database and user management tasks are therefore skipped until MySQL is actually installed.
Example:
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/mysql.yml \
  --check \
  --ask-vault-pass

Usage
Run the dedicated MySQL playbook:
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/mysql.yml \
  --ask-vault-pass

MySQL is also included in the main infrastructure playbook:
ansible-playbook \
  -i inventories/prod/hosts.ini \
  playbooks/site.yml \
  --ask-vault-pass

Idempotence
The role is designed to be idempotent.
Once the desired MySQL state is configured, subsequent executions should complete with:
changed=0
failed=0