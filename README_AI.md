## Overview

Automated deployment of WordPress on one or more remote Ubuntu servers using Ansible.
Every process runs in its own container (1 process = 1 container):

| Container    | Image                  | Role                                                  |
|--------------|------------------------|-------------------------------------------------------|
| `caddy`      | `caddy:2`              | Reverse proxy, automatic HTTPS (Let's Encrypt), only container with published ports (80, 443) |
| `wordpress`  | `wordpress:php8.3-fpm` | WordPress (PHP-FPM)                                   |
| `mariadb`    | `mariadb:11`           | Database, reachable only from the internal Docker network |
| `phpmyadmin` | `phpmyadmin:5-apache`  | DB admin UI, served by Caddy under `/phpmyadmin/`     |
| `wpcli`      | `wordpress:cli`        | One-shot container used to install WordPress          |

Routing: `https://<domain>/` -> WordPress, `https://<domain>/phpmyadmin/` -> phpMyAdmin.
Data lives in Docker named volumes (`mariadb_data`, `wordpress_data`) and survives reboots.

### Ansible roles

| Role       | What it does                                                              |
|------------|---------------------------------------------------------------------------|
| `common`   | apt update/upgrade, utilities, reboot if required                         |
| `firewall` | UFW: allow 22, 80, 443, deny everything else                              |
| `docker`   | Docker repository + Docker CE + Compose plugin, service enabled at boot   |
| `stack`    | Renders `docker-compose.yml` and `Caddyfile`, starts the stack, installs WordPress with wp-cli |

## Requirements

- **Control machine:** Python 3 and Ansible.
- **Target servers:** Ubuntu 22.04 LTS with an SSH daemon and Python, nothing else. SSH access as `root` with your key.
- **One domain per server** (e.g. from DuckDNS) with an A record pointing to that server's IP. Do this *before* deploying, otherwise Caddy cannot get a certificate.

## Setup

### 1. Install Ansible

```bash
cd cloud1/ansible
python3 -m venv .venv
source .venv/bin/activate
pip install ansible
```

`pip install ansible` is the full package and already includes the `community.docker` and `community.general` collections.
If you install only `ansible-core`, also run:
`ansible-galaxy collection install community.docker community.general`

### 2. Create the vault (secrets)

All secrets are stored encrypted in `ansible/group_vars/cloud1/vault.yml`. It must define:

| Variable                         | Used for                        |
|----------------------------------|---------------------------------|
| `vault_db_password`              | WordPress database user         |
| `vault_db_root_password`         | MariaDB root                    |
| `vault_wordpress_admin_password` | WordPress admin account         |

```bash
openssl rand -hex 16                                  # generate a random password
ansible-vault create group_vars/cloud1/vault.yml      # create (asks for the vault password)
ansible-vault edit   group_vars/cloud1/vault.yml      # edit
ansible-vault view   group_vars/cloud1/vault.yml      # view
```

Never commit the file unencrypted. Remember the vault password, it is needed for every run.

### 3. Configure inventory and per-server variables

- `ansible/inventory.ini`: one line per server.
- `ansible/host_vars/<name>.yml`: **required for every server**, contains its own domain:
  ```yaml
  domain_name: my-site.duckdns.org
  ```
- `ansible/group_vars/cloud1/cloud1.yml`: settings shared by all servers (DB name/user, WordPress title, admin user and e-mail).

Example for two servers:

```ini
[cloud1]
server1 ansible_host=203.0.113.10 ansible_user=root ansible_ssh_private_key_file=~/.ssh/cloud1
server2 ansible_host=203.0.113.20 ansible_user=root ansible_ssh_private_key_file=~/.ssh/cloud1
```

with `host_vars/server1.yml` and `host_vars/server2.yml` each defining their own `domain_name`.
Delete or comment out inventory lines for servers you do not have, otherwise Ansible will try to connect to them.

### 4. Trust the server's SSH host key

Host key checking is enabled, so add every server once:

```bash
ssh-keyscan -H <SERVER_IP> >> ~/.ssh/known_hosts
```

## Deployment

All commands are run from the `ansible/` directory.

```bash
# check connectivity (the vault is loaded automatically, so the password is needed)
ansible cloud1 -m ping --ask-vault-pass
```

### Deploy to ALL servers (in parallel)

```bash
ansible-playbook site.yml --ask-vault-pass
```

Ansible works on up to 5 servers at the same time by default. Change it with `-f`, e.g. `-f 10`.

### Deploy to ONE server only

```bash
ansible-playbook site.yml --ask-vault-pass --limit server1
```

Several selected servers: `--limit server1,server2`.

### Adding a new server

1. Create the instance (Ubuntu 22.04, SSH key, Python).
2. Create a DNS A record for its domain pointing to the new IP.
3. Add a line to `inventory.ini`.
4. Create `host_vars/<name>.yml` with its `domain_name`.
5. `ssh-keyscan -H <IP> >> ~/.ssh/known_hosts`
6. Deploy only the new server: `ansible-playbook site.yml --ask-vault-pass --limit <name>`

### Idempotency

The playbook can be run repeatedly. A second run on an already deployed server should report `changed=0`
(WordPress is installed only if it is not installed yet).

## Verification

```bash
# HTTP must redirect to HTTPS
curl -I http://<domain>

# Site and phpMyAdmin respond over HTTPS
curl -I https://<domain>
curl -I https://<domain>/phpmyadmin/

# Only 22, 80 and 443 may be open
nmap -Pn -p 22,80,443,3306,8080 <SERVER_IP>

# On the server
ssh root@<SERVER_IP>
cd /opt/cloud1 && docker compose ps
ufw status verbose
```

**Reboot test:** create a post or a user, run `ssh root@<SERVER_IP> reboot`, wait about a minute and check that the site
is back and the data is still there.

## Stopping and cleaning up

Cloud instances cost money. Stop or delete them when you are done.

```bash
# stop the stack, keep the data
ssh root@<SERVER_IP> 'cd /opt/cloud1 && docker compose down'

# stop the stack AND delete all data (database, uploads)
ssh root@<SERVER_IP> 'cd /opt/cloud1 && docker compose down -v'
```

## Security notes

- Only ports 22, 80 and 443 are open (UFW, default deny incoming).
- Docker bypasses UFW for published ports, so the protection also relies on the compose file: only `caddy` has `ports:`. The database and phpMyAdmin are reachable only through the internal Docker network.
- No secrets in the repository: passwords are in the Ansible Vault, and the WordPress install task uses `no_log`.

## AI usage

<!-- The subject requires transparency. Describe honestly which parts were generated or suggested by AI
     and which parts you wrote, reviewed or changed yourself. -->