# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible role that installs and configures [Netdata](https://www.netdata.cloud/) monitoring, with optional support for a centralized Netdata registry exposed via nginx reverse proxy.

## Running the Role

Apply the role against an inventory (from the parent playbook directory):

```bash
ansible-playbook site.yml --tags netdata
# or target specific hosts
ansible-playbook site.yml -l <hostname> --tags netdata
# syntax check
ansible-playbook site.yml --syntax-check
# dry run
ansible-playbook site.yml --check -l <hostname>
```

Lint the role:

```bash
ansible-lint roles/netdata
```

## Architecture

The role has two distinct modes controlled by `netdata_registry`:

- **`netdata_registry: False`** (default) — standalone node; only `netdata.conf` is deployed, no nginx setup.
- **`netdata_registry: "<hostname>"`** — one host in the inventory is designated as the registry. That host gets the nginx vhost (`sites-available/netdata` symlinked to `sites-enabled/`) acting as a reverse proxy for all hosts in the `all` group. All other nodes point their Netdata registry announcement at `netdata_registry_to_annonce`.

### Key variables (`defaults/main.yml`)

| Variable | Default | Purpose |
|---|---|---|
| `netdata_registry` | `False` | Registry hostname or `False` to disable |
| `netdata_packages` | `netdata` | Package name(s) to install |

Additional variables expected in the playbook/inventory (no defaults provided):

- `netdata_registry_to_annonce` — URL of the registry (used in `netdata.conf.j2` when registry is enabled but this host is not the registry node)
- `netdata_nginx_server_name` — `server_name` for the nginx vhost (registry node only)

### Templates

- `netdata.conf.j2` — minimal Netdata global config; conditionally adds a `[registry]` section.
- `nginx.j2` — generates one `upstream` block per host in `all`, then a single HTTPS server block that proxies requests by hostname segment (`/hostname/path` → upstream `netdata-hostname`). Uses snakeoil self-signed certificates by default.

### Handlers

- `restart netdata` — restarts the `netdata` service.
- `check nginx` — runs `nginx -t`; on success notifies `restart nginx`.
- `restart nginx` — restarts the `nginx` service.

The nginx config change triggers `check nginx` (not a direct restart), so a broken config won't restart nginx.
