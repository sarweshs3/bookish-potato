
# Node.js application — Ansible project

This project reorganizes the supplied standalone playbooks into reusable Ansible roles.
It targets Rocky Linux 9 and keeps the original intent: server updates, SSH/Fail2ban
hardening, firewalld, a dedicated `nodeapp` user, Node.js 20, app deployment,
environment configuration, systemd, Nginx reverse proxy, log rotation and health checks.

## Project layout

```text
ansible-project/
├── ansible.cfg
├── requirements.yml
├── site.yml
├── maintenance.yml
├── inventories/production/hosts.ini
├── group_vars/nodeservers.yml
├── group_vars/nodeservers/vault.yml.example
├── roles/
│   ├── common/
│   ├── firewall/
│   ├── security/
│   ├── app_user/
│   ├── nodejs/
│   ├── app_deploy/
│   ├── app_environment/
│   ├── app_service/
│   ├── nginx/
│   ├── logrotate/
│   └── healthcheck/
└── .gitignore
```

## 1. Requirements

Use a control machine with Ansible installed. Install the collection used by firewalld:

```bash
ansible-galaxy collection install -r requirements.yml
```

Check connectivity and facts:

```bash
ansible nodeservers -m ping
ansible nodeservers -m setup -a 'filter=ansible_distribution*'
```

The inventory currently contains the uploaded host `192.168.100.218` and SSH user
`sarvesh`. Change these to match your environment before running.

## 2. Configure variables and secrets

Edit `group_vars/nodeservers.yml` for your domain, certificate paths, repository,
health-check email, and other environment-specific settings.

Create the vault file from the example:

```bash
cp group_vars/nodeservers/vault.yml.example group_vars/nodeservers/vault.yml
ansible-vault encrypt group_vars/nodeservers/vault.yml
```

Edit it securely with:

```bash
ansible-vault edit group_vars/nodeservers/vault.yml
```

The vault file must define `vault_database_url`, `vault_redis_url`, and `vault_app_secret`.
Do not commit an unencrypted secrets file. Supply the vault password interactively with
`--ask-vault-pass`, or configure a protected vault-password file in your environment.

## 3. Important pre-flight checks

- **SSH hardening can lock you out.** Before applying `site.yml`, verify key-based SSH
  login works for the Ansible connection user and keep an existing session or console access.
  The security role disables SSH password authentication and root login.
- Nginx is configured for HTTPS but does **not** create TLS certificates. Ensure the
  configured certificate and private key exist before enabling the Nginx role. Replace the
  example `example.com` domain and self-signed paths with real values as appropriate.
- The health-check role uses the system `mail` command. Configure a working mail transport
  on the server; installing `s-nail` alone does not guarantee email delivery.
- The app role assumes the repository contains `package.json`, preferably `package-lock.json`,
  and the configured entry point (default `server.js`). Update `app_entrypoint` if needed.
- The firewall role allows only the listed ports as explicit additions (22/tcp, 80/tcp,
  443/tcp). Review the active zone and existing rules before applying on a remote server.
- The NodeSource setup script is downloaded from NodeSource and executed with root privileges.
  Review the source and your organization's software supply-chain policy before production use.

## 4. Run safely

Syntax-check first:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook maintenance.yml --syntax-check
```

Preview changes:

```bash
ansible-playbook site.yml --check --diff --ask-vault-pass
```

Run the full configuration:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Run selected tags, for example:

```bash
ansible-playbook site.yml --tags firewall
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --tags healthcheck
```

Run OS updates separately because they can reboot the host:

```bash
ansible-playbook maintenance.yml
```

Check mode is useful but is not a perfect simulation for every command or external repository
setup. Test on a disposable Rocky Linux 9 VM before production.

## 5. Roles and execution order

`site.yml` runs roles in this order:

1. `common` — package updates and optional reboot check.
2. `firewall` — firewalld and allowed ports.
3. `security` — SSH settings and Fail2ban.
4. `app_user` — service user/group and optional authorized key.
5. `nodejs` — Node.js 20.
6. `app_deploy` — clone/update source and install npm dependencies.
7. `app_environment` — deploy `.env` with secret values and restrictive permissions.
8. `app_service` — install/reload systemd unit and restart the app when it changes.
9. `nginx` — deploy and validate reverse-proxy configuration, then reload/start Nginx.
10. `logrotate` — configure app log rotation.
11. `healthcheck` — install the check script and schedule it every five minutes.

Use `maintenance.yml` for OS updates independently if you prefer not to update/reboot as part
of every application configuration run.

## 6. Notes on idempotency and operations

- Roles use handlers for service reload/restart when configuration changes.
- Nginx configuration is validated before a reload; a bad config should fail the role.
- `npm ci` is preferred when `package-lock.json` exists.
- Environment values are hidden from Ansible output with `no_log: true`.
- Keep secrets encrypted with Ansible Vault and restrict access to vault passwords.
