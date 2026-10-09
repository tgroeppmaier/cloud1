# Cloud-1

Automated WordPress deployment on a remote Ubuntu server with Ansible and Docker.

The playbook installs Docker, then runs one container per service:

| Service | Image | Role |
| --- | --- | --- |
| MariaDB | `mariadb:11` | WordPress database (not published to the internet) |
| WordPress | `wordpress:php8.3-fpm` | PHP-FPM app |
| phpMyAdmin | `phpmyadmin:5-apache` | DB admin UI at `/phpmyadmin/` |
| Caddy | `caddy:2` | Reverse proxy, TLS, static files |
| WP-CLI | `wordpress:cli` | First-time WordPress install only |

After deploy:

| Host | Site | phpMyAdmin |
| --- | --- | --- |
| server1 | https://cloud1-inception.duckdns.org | https://cloud1-inception.duckdns.org/phpmyadmin/ |
| server2 | https://cloud2-inception.duckdns.org | https://cloud2-inception.duckdns.org/phpmyadmin/ |

Stack files on each server: `/opt/cloud1`

Containers use `restart: unless-stopped` and named volumes, so the site comes back after a reboot with data intact.

## Requirements

**Target host**

- Ubuntu 22.04 LTS (or similar)
- SSH access as root
- Python 3
- DNS A record for the domain pointing at the host (needed for Caddy TLS)

**Control machine**

- Python 3
- SSH private key that can log in as root

## Ansible layout

```text
ansible/
  ansible.cfg
  inventory.ini                 # hosts and SSH settings
  site.yml                      # playbook
  requirements.yml              # Ansible collections (community.docker, community.general)
  host_vars/
    server1.yml                 # domain for server1
    server2.yml                 # domain for server2
  group_vars/cloud1/
    cloud1.yml                  # shared non-secret vars (DB name, WP title)
    vault.yml                   # encrypted secrets
  roles/
    common/                     # apt update, utilities, reboot if needed
    docker/                     # Docker CE + Compose plugin
    stack/                      # compose stack + WordPress install
    firewall/                   # UFW: only 22/80/443 open
```

`site.yml` applies `common` → `docker` → `firewall` → `stack`.

## Ansible files

Ansible does not use a custom loader here. It follows the usual names:

- `inventory.ini` names the hosts (`server1`, `server2`) and puts them in group `cloud1`
- `host_vars/<hostname>.yml` is loaded automatically for that host
- `group_vars/cloud1/` is loaded automatically for every host in `cloud1`
- `site.yml` is the playbook you run; it applies roles in order
- role `tasks/main.yml` files are the steps; `templates/*.j2` are rendered onto the server

### `ansible.cfg`

Ansible defaults for this project: use `inventory.ini`, look for roles in `roles/`, pick the host Python with `interpreter_python = auto`, and keep SSH host key checking on.

### `inventory.ini`

List of target servers. Each line is an inventory **name** (`server1`) plus `ansible_host` (the IP Ansible SSHs to). `[cloud1:vars]` sets the SSH user and key for the whole group.

`site.yml` runs against group `cloud1`, so only hosts still listed here are deployed. Comment a host out (or use `--limit server1`) to deploy one server.

### `requirements.yml`

Collections the playbook needs beyond `ansible-core`: `community.docker` (Compose module) and `community.general` (UFW module). Installed with `ansible-galaxy collection install -r requirements.yml`.

### `site.yml`

The playbook. Targets `cloud1`, uses `become` (root), loads vault + group vars, then runs roles `common` → `docker` → `firewall` → `stack`.

### `host_vars/server1.yml` / `host_vars/server2.yml`

Per-host variables. The filename **must** match the inventory name. That is how each server gets its own `domain_name` for Caddy TLS, WordPress `--url`, and phpMyAdmin. Nothing in `site.yml` points at these files.

### `group_vars/cloud1/cloud1.yml`

Shared non-secret settings for every host in `cloud1`: database name/user, WordPress title, admin username, admin email (`admin@<domain_name>`, so no personal address is stored in the repo).

### `group_vars/cloud1/vault.yml`

Encrypted secrets (`ansible-vault`). Used as `vault_db_password`, `vault_db_root_password`, and `vault_wordpress_admin_password`. Edit only with `ansible-vault edit`; the playbook needs `--ask-vault-pass`.

### `roles/common/tasks/main.yml`

Refresh apt, install small utilities (`fish`, `tmux`, `jq`), reboot if `/var/run/reboot-required` exists.

### `roles/docker/tasks/main.yml`

Install Docker CE, the Compose plugin, and enable/start the Docker service so containers come back after a reboot.

### `roles/firewall/tasks/main.yml`

Installs UFW, allows TCP 22/80/443 (`firewall_allowed_tcp_ports` in `defaults/main.yml`), then sets default deny for incoming traffic and enables it. Allow rules are added first so the SSH session is not cut off. It uses `community.general.ufw` (see `requirements.yml`). The stack only publishes Caddy's 80/443 on the host, because ports published by Docker bypass UFW.

### `roles/stack/tasks/main.yml`

Create `/opt/cloud1`, render the Compose file and Caddyfile, start the stack and wait for the containers to be healthy, then run WP-CLI `wp core install` only if WordPress is not already installed. A real error from `wp core is-installed` (for example a missing `wp-config.php` or no DB connection) fails the play instead of triggering an install.

### `roles/stack/templates/docker-compose.yml.j2`

Jinja template for the Compose stack (MariaDB, WordPress FPM, phpMyAdmin, Caddy, WP-CLI). Written to `/opt/cloud1/docker-compose.yml`. `{{ domain_name }}` and vault passwords are filled in per host. MariaDB and WordPress have health checks; Caddy and WP-CLI wait for them, so WP-CLI never runs before `wp-config.php` exists.

### `roles/stack/templates/Caddyfile.j2`

Jinja template for Caddy: TLS for `{{ domain_name }}`, WordPress on `/`, phpMyAdmin on `/phpmyadmin/`. Written to `/opt/cloud1/caddy/Caddyfile`.

## Setup (control machine)

```bash
cd ansible
python3 -m venv .venv
source .venv/bin/activate
pip install ansible
ansible-galaxy collection install -r requirements.yml
```

## Configuration

1. Edit `ansible/inventory.ini`: host IPs, SSH user, and key path.
2. Edit `ansible/host_vars/<host>.yml`: that server's DuckDNS (or other) domain.
3. Edit `ansible/group_vars/cloud1/cloud1.yml`: shared DB name/user, WordPress title/admin user (the admin email is derived from the domain).
4. Put secrets in the vault (never in git plaintext). The vault is an encrypted file, `ansible/group_vars/cloud1/vault.yml`, that Ansible decrypts at run time with a password you choose.

Create it (or replace the existing one with your own; delete `vault.yml` first if it exists, since you cannot read the committed one without its password):

```bash
cd ansible
ansible-vault create group_vars/cloud1/vault.yml
```

Ansible asks for a new vault password twice, then opens your editor. Enter the three secrets and save:

```yaml
vault_db_password: <random string>
vault_db_root_password: <random string>
vault_wordpress_admin_password: <random string>
```

Generate each value with `openssl rand -hex 16`. Remember the vault password: the file cannot be recovered without it, and you need it for every deploy (`--ask-vault-pass`).

Change or inspect the file later:

```bash
ansible-vault edit group_vars/cloud1/vault.yml
ansible-vault view group_vars/cloud1/vault.yml
```

Point each DuckDNS name at the matching server IP before the first deploy so Caddy can get a certificate.

## Deploy

Before the first deploy (and again after you recreate a server), add the servers' SSH host keys. `ansible.cfg` keeps `host_key_checking = True`, so Ansible refuses to connect to a host it does not know.

```bash
ssh-keygen -R YOUR_VPS_IP          # drop a stale key (prints "not found" if there is none)
ssh-keyscan -H YOUR_VPS_IP >> ~/.ssh/known_hosts
```

Do this for every IP in `inventory.ini`. If `ansible cloud1 -m ping --ask-vault-pass` fails with `Host key verification failed`, the key is missing or outdated; repeat the two commands above. To see the exact SSH error, run `ssh -i ~/.ssh/cloud1 root@YOUR_VPS_IP`.

Then, from `ansible/` with the venv active:

```bash
ansible cloud1 -m ping --ask-vault-pass
ansible-playbook site.yml --ask-vault-pass
```

`ping` needs `--ask-vault-pass` too: Ansible loads every file in `group_vars/cloud1/` for the group, including the encrypted `vault.yml`, even for ad-hoc commands.

That runs on every host in `[cloud1]`. Limit to one server with `--limit server2`.

The playbook is meant to be idempotent: run it again without wiping WordPress data.

Each host needs its own domain in `host_vars`. Two servers sharing one DNS name will fight over TLS.

## Useful commands

```bash
# SSH known_hosts
ssh-keyscan -H YOUR_VPS_IP >> ~/.ssh/known_hosts
ssh-keygen -R YOUR_VPS_IP

# Vault
ansible-vault view group_vars/cloud1/vault.yml
ansible-vault edit group_vars/cloud1/vault.yml

# On the server
systemctl status docker
docker compose -f /opt/cloud1/docker-compose.yml ps

# From your machine: HTTP redirects to HTTPS, site and phpMyAdmin answer 200
curl -sI http://YOUR_DOMAIN/ | head -n 1
curl -sI https://YOUR_DOMAIN/ | head -n 1
curl -sI https://YOUR_DOMAIN/phpmyadmin/ | head -n 1

# From your machine: 22/80/443 should be open, 3306/8080 should not
nmap -Pn -p 22,80,443,3306,8080 YOUR_VPS_IP
```

## AI usage

AI was used for this README and for general consultation.
