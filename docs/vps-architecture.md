# VPS Architecture Handoff

Last updated from the infrastructure repository on 2026-06-26.

This document describes the current Hetzner VPS setup so future project agents can
deploy new apps without rediscovering the server layout. Treat this as context,
not as a complete deployment spec for any one application.

## Server

- Provider: Hetzner VPS.
- Inventory host alias: `hetzner`.
- Public IPv4: `178.105.241.87`.
- SSH user: `root`.
- SSH port: `22`.
- Ansible group: `vps`.
- Python interpreter for Ansible: `/usr/bin/python3`.

The local Ansible inventory is intentionally not committed. The committed
template is `inventory.example.yml`; the real file is `inventory.yml`.

## Source Of Truth

The VPS base configuration is managed from the infrastructure repository:

- `setup-vps.yml`: Ansible playbook for base packages, Docker, app directories,
  Caddy, and services.
- `files/Caddyfile`: Caddy reverse proxy configuration.
- `inventory.example.yml`: Safe template for the private inventory.
- `inventory.yml`: Local private inventory, ignored by Git.

Before changing the server manually, prefer updating the Ansible playbook and
running it. Manual changes can drift from the repository and surprise future
deployments.

## Base System

The playbook installs and enables:

- Caddy as the public HTTP/HTTPS reverse proxy.
- Docker CE from Docker's official APT repository.
- Docker Compose v2 plugin via `docker-compose-plugin`.
- `goaccess` for access log inspection.
- `ufw`, installed but not currently configured in the playbook.
- `unattended-upgrades`, installed with package defaults.
- `ca-certificates` and `python3-debian`.

The playbook removes Ubuntu/Debian Docker-related packages that conflict with
Docker CE:

- `containerd`
- `docker-compose`
- `docker-compose-v2`
- `docker.io`
- `podman-docker`
- `runc`

Docker and Caddy are both enabled and started by Ansible.

## Network And Routing

Caddy is the only configured public reverse proxy. It terminates HTTPS
automatically for configured domains and proxies traffic to apps bound on
localhost.

Current routes:

| Public host | Local upstream | Access log |
| --- | --- | --- |
| `gunfight.mkjems.dk` | `127.0.0.1:8080` | `/var/log/caddy/gunfight.mkjems.dk-access.log` |
| `mastermind.mkjems.dk` | `127.0.0.1:8081` | `/var/log/caddy/mastermind.mkjems.dk-access.log` |
| `mkjems.dk` | `127.0.0.1:8082` | `/var/log/caddy/mkjems.dk-access.log` |

Application containers should generally bind to `127.0.0.1:<port>` and let
Caddy expose them publicly. Avoid publishing application ports to `0.0.0.0`
unless there is a specific reason.

## Application Directories

The playbook creates these directories:

- `/opt/homepage`
- `/opt/gunfight`
- `/opt/mastermind`

Each directory is owned by the Ansible SSH user from inventory. In the current
inventory that user is `root`, so the directories are root-owned unless the
inventory changes.

The base playbook only creates directories. It does not currently define or
start app containers, Docker Compose files, systemd units, deploy keys, or
application environment files. Those details belong to each project unless they
become shared infrastructure.

## Logging

Caddy access logs live in `/var/log/caddy`. The playbook ensures the directory
exists and touches the current per-site log files with owner/group `caddy:caddy`
and mode `0640`.

Useful commands on the VPS:

```sh
sudo tail -q -F /var/log/caddy/*-access.log | goaccess - --log-format=CADDY
```

Inspect one site:

```sh
sudo goaccess /var/log/caddy/gunfight.mkjems.dk-access.log --log-format=CADDY
```

## Applying Infrastructure Changes

From the infrastructure repository, preview changes first:

```sh
ansible-playbook -i inventory.yml setup-vps.yml --check --diff
```

Then apply:

```sh
ansible-playbook -i inventory.yml setup-vps.yml
```

The Caddy config is validated before installation with:

```sh
/usr/bin/caddy adapt --adapter caddyfile --config <candidate-file>
```

When Caddy-related files or logs change through Ansible, the handler reloads
Caddy.

## Adding A New Project

When deploying another app to this VPS, keep these conventions in mind:

1. Pick a unique local port, preferably the next free app port after `8082`.
2. Create a dedicated app directory under `/opt/<project-name>`.
3. Run the app in Docker or Docker Compose from that directory.
4. Bind the app to localhost, for example `127.0.0.1:8083`.
5. Add a Caddy site block in `files/Caddyfile` for the public domain.
6. Add the matching access log path to `caddy_access_logs` in `setup-vps.yml`.
7. Add the app directory to `application_directories` in `setup-vps.yml`.
8. Run the Ansible check command and inspect the diff before applying.

Example Caddy block shape:

```caddyfile
example.mkjems.dk {
	log {
		output file /var/log/caddy/example.mkjems.dk-access.log
	}
	reverse_proxy 127.0.0.1:8083
}
```

## Agent Notes

- Do not assume app deployment is already standardized beyond Docker and Caddy.
- Do not replace Caddy with another reverse proxy unless explicitly requested.
- Do not install Ubuntu's `docker.io`; the server is intentionally on Docker CE.
- Keep secrets out of Git. `inventory.yml`, `.env`, `*.key`, `*.pem`,
  `vault-password*`, and `secrets.yml` are ignored.
- If a project needs persistent data, place it deliberately under its project
  directory or another clearly documented path, then document backup needs.
- If a project needs public ingress, update the infrastructure repo rather than
  only editing `/etc/caddy/Caddyfile` by hand.
