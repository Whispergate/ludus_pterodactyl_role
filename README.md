# ludus_pterodactyl

A [Ludus](https://ludus.cloud) Ansible role that deploys the
[Pterodactyl](https://pterodactyl.io/) game server management panel (and,
optionally, the Wings daemon) onto a Debian/Ubuntu VM in your range.

It gives you a one-shot, repeatable install that lets you:

- **Access the game server interface**: installs the Panel behind nginx +
  php-fpm, provisions MariaDB and Redis, and creates an initial administrator so
  the web UI is ready to log in to.
- **Manage SSL provisioning**: choose `selfsigned` (works offline, the
  default), `letsencrypt` (trusted certs via certbot), or `none` (plain HTTP).
- **Upgrade the existing install**: set `ludus_pterodactyl_upgrade: true` to
  run Pterodactyl's official in-place upgrade to the latest (or a pinned)
  release.
- **Manage the web server's domain and internals**: set the served
  domain/IP, upload size, PHP version, TLS material and more.
- **Perform additional configuration**: database and admin credentials,
  timezone, Redis, and outbound mail settings.
- **Run game servers**: optionally installs Docker + the Wings daemon so the
  node can host containers.
- **Scale to multiple nodes**: deploy the role on extra VMs as Wings-only
  nodes (`install_panel: false`), optionally auto-registering them with the
  panel via an auto-deploy token.

## Requirements

- A Ludus range with a Debian 11/12 or Ubuntu 20.04/22.04/24.04 VM.
- The VM needs internet access during the install to fetch packages, the panel
  release, Composer and (for Wings) Docker.
- Runs with root privileges (the Ludus default for Linux VMs).

## Installation

From your Ludus host:

```bash
git clone <this repo> ludus_pterodactyl
ludus ansible roles add -d ./ludus_pterodactyl
```

## Role Variables

All variables are prefixed with `ludus_pterodactyl_`. The most useful ones (see
[`defaults/main.yml`](defaults/main.yml) for the full list and comments):

| Variable | Default | Description |
| --- | --- | --- |
| `ludus_pterodactyl_install_panel` | `true` | Install the web panel. Set `false` for Wings-only node VMs. |
| `ludus_pterodactyl_install_wings` | `true` | Install Docker + the Wings daemon. |
| `ludus_pterodactyl_wings_auto_configure` | `false` | Auto-register the node with the panel via an auto-deploy token. |
| `ludus_pterodactyl_wings_panel_url` | this VM's panel URL | Panel base URL the node talks to (needed on Wings-only VMs). |
| `ludus_pterodactyl_wings_node_id` | `""` | Numeric node ID from the panel (for auto-configure). |
| `ludus_pterodactyl_wings_node_token` | `""` | Auto-deploy token from the panel (for auto-configure). |
| `ludus_pterodactyl_wings_allow_insecure` | `true` if self-signed | Skip TLS check when fetching node config. |
| `ludus_pterodactyl_domain` | VM primary IPv4 | Hostname/IP the panel is served on. |
| `ludus_pterodactyl_ssl_mode` | `selfsigned` | `none`, `selfsigned`, or `letsencrypt`. |
| `ludus_pterodactyl_letsencrypt_email` | `admin@<domain>` | ACME registration email. |
| `ludus_pterodactyl_php_version` | `8.3` | PHP version (`8.2`/`8.3`). |
| `ludus_pterodactyl_client_max_body_size` | `100m` | nginx upload limit. |
| `ludus_pterodactyl_panel_version` | `latest` | Panel release tag or `latest`. |
| `ludus_pterodactyl_upgrade` | `false` | Run the in-place upgrade path. |
| `ludus_pterodactyl_db_password` | *changeme* | Panel database password. |
| `ludus_pterodactyl_admin_username` | `admin` | First admin username. |
| `ludus_pterodactyl_admin_password` | *changeme* | First admin password. |
| `ludus_pterodactyl_admin_email` | `admin@<domain>` | First admin email. |
| `ludus_pterodactyl_timezone` | `UTC` | Panel timezone. |
| `ludus_pterodactyl_mail_driver` | `log` | `log` or `smtp` (+ `mail_*` vars). |
| `ludus_pterodactyl_manage_firewall` | `false` | Open ports 80/443/8080/2022 with ufw. |

> **Change the default passwords** (`ludus_pterodactyl_db_password` and
> `ludus_pterodactyl_admin_password`) in your range config.

## Example Ludus range config

```yaml
ludus:
  - vm_name: "{{ range_id }}-pterodactyl"
    hostname: "pterodactyl"
    template: debian-12-x64-server-template
    vlan: 10
    ip_last_octet: 21
    ram_gb: 4
    cpus: 2
    linux: true
    roles:
      - ludus_pterodactyl
    role_vars:
      ludus_pterodactyl_domain: "10.2.10.21"
      ludus_pterodactyl_ssl_mode: "selfsigned"
      ludus_pterodactyl_admin_username: "gameadmin"
      ludus_pterodactyl_admin_password: "SuperSecretPassw0rd!"
      ludus_pterodactyl_db_password: "AnotherSecret!"
      ludus_pterodactyl_install_wings: true
```

Then deploy:

```bash
ludus range config set -f range-config.yml
ludus range deploy
```

When it finishes, browse to `https://10.2.10.21` (accept the self-signed cert)
and log in with the admin credentials you set.

## SSL modes

- **selfsigned**: generates a 10-year cert (with a SAN matching the domain or
  IP) under `/etc/pterodactyl/certs/`. Works with no internet or DNS, so it
  suits an isolated range. Browsers will warn about the untrusted cert.
- **letsencrypt**: installs certbot and issues a cert via the standalone
  challenge. Requires the domain to be a public FQDN that resolves to the VM and
  **inbound port 80 reachable from the internet** (in a range, the Ludus host
  must forward public :80 to the VM, not just the router firewall rule). A
  renewal hook reloads nginx automatically. If issuance fails, the role falls
  back to a self-signed cert so the panel still starts
  (`ludus_pterodactyl_letsencrypt_fallback_selfsigned`, default true); fix
  inbound :80 and re-run to get the trusted cert.
- **none**: serves plain HTTP on port 80 (no certificate).

## Wings / running game servers

By default (`install_panel: true` + `install_wings: true`) the panel and one
Wings node are installed on the same VM.

Wings is installed and enabled but **not started** until a node exists, because
it needs its node config. After the panel is up:

1. In the panel: **Admin → Nodes → Create New**, pointing at this VM's FQDN/IP.
2. On the node's **Configuration** tab, copy the generated YAML into
   `/etc/pterodactyl/config.yml`.
3. `systemctl start wings`.

## Multiple nodes (Wings-only VMs)

To scale out, deploy this same role onto additional VMs as **Wings-only** nodes
by turning the panel off:

```yaml
role_vars:
  ludus_pterodactyl_install_panel: false
  ludus_pterodactyl_install_wings: true
```

Those VMs install only Docker + Wings (no PHP/MariaDB/nginx). Register each one
in the panel exactly as above, or let the role do it automatically with an
auto-deploy token:

1. In the panel: **Admin → Nodes → Create New** for the node VM.
2. Open the node → **Configuration** tab → **Generate Token**. Note the panel
   URL, the numeric **node ID**, and the **token**.
3. Set these on the node's `role_vars` and re-run the role:

```yaml
role_vars:
  ludus_pterodactyl_install_panel: false
  ludus_pterodactyl_install_wings: true
  ludus_pterodactyl_wings_auto_configure: true
  ludus_pterodactyl_wings_panel_url: "https://10.2.10.21"
  ludus_pterodactyl_wings_node_id: "2"
  ludus_pterodactyl_wings_node_token: "<auto-deploy token>"
  # allow_insecure defaults to true when the panel uses a self-signed cert
```

The role runs `wings configure` to write `/etc/pterodactyl/config.yml` and starts
Wings. `--allow-insecure` is added automatically when the panel is self-signed.

### Example: panel + two nodes

```yaml
ludus:
  - vm_name: "{{ range_id }}-ptero-panel"
    hostname: "ptero-panel"
    template: debian-12-x64-server-template
    vlan: 10
    ip_last_octet: 21
    ram_gb: 4
    cpus: 2
    linux: true
    roles:
      - ludus_pterodactyl
    role_vars:
      ludus_pterodactyl_domain: "10.2.10.21"
      ludus_pterodactyl_install_wings: false   # keep the panel VM panel-only
      ludus_pterodactyl_admin_password: "SuperSecretPassw0rd!"
      ludus_pterodactyl_db_password: "AnotherSecret!"

  - vm_name: "{{ range_id }}-ptero-node1"
    hostname: "ptero-node1"
    template: debian-12-x64-server-template
    vlan: 10
    ip_last_octet: 31
    ram_gb: 8
    cpus: 4
    linux: true
    roles:
      - ludus_pterodactyl
    role_vars:
      ludus_pterodactyl_install_panel: false
      ludus_pterodactyl_install_wings: true

  - vm_name: "{{ range_id }}-ptero-node2"
    hostname: "ptero-node2"
    template: debian-12-x64-server-template
    vlan: 10
    ip_last_octet: 32
    ram_gb: 8
    cpus: 4
    linux: true
    roles:
      - ludus_pterodactyl
    role_vars:
      ludus_pterodactyl_install_panel: false
      ludus_pterodactyl_install_wings: true
```

Deploy the panel first, create each node in the panel UI (or generate tokens and
re-run with `wings_auto_configure`), and you have a multi-node cluster.

> **Node ↔ panel networking:** put nodes on the same VLAN as the panel (or open
> routing between them). Wings listens on `8080` (daemon API) and `2022` (SFTP);
> the panel must be able to reach the node on those ports and vice-versa.

## Upgrading

Set `ludus_pterodactyl_upgrade: true` (optionally pin
`ludus_pterodactyl_panel_version`) and re-run the role. It puts the panel in
maintenance mode, downloads the release, updates Composer dependencies, runs
migrations, and brings it back up. The first-time install path is skipped
automatically when a panel is already present.

## What gets installed

- **Panel**: PHP 8.3 + extensions, MariaDB, Redis, nginx, Composer, the
  panel source under `/var/www/pterodactyl`, a `pteroq.service` queue worker and
  a per-minute `schedule:run` cron entry.
- **Wings**: Docker CE, the `wings` binary at `/usr/local/bin/wings`, and a
  `wings.service` systemd unit.

## License

BSD-2-Clause. See [LICENSE](LICENSE).
