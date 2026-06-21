# group_vars/webservers.yml — Configuration Guide

This file drives everything the playbook deploys for the `webservers` group.
It supports two webserver backends — **nginx** and **httpd (Apache)** — and a
single switch decides which one gets installed and configured.

---

## 1. The switch: `webserver_package`

```yaml
webserver_package: nginx   # or: httpd
```

This is the **only variable that determines which webserver runs** on a host.
Everything else in this file is grouped into `nginx_*` and `httpd_*`
variables, and only the block matching `webserver_package` is actually used —
the other block sits inert and is safely ignored by the playbook.

- `webserver_package: nginx` → playbook installs nginx, deploys
  `nginx.conf.j2` + `server_block.conf.j2` for each entry in
  `nginx_server_blocks`, validates with `nginx -t`.
- `webserver_package: httpd` → playbook installs Apache, deploys
  `httpd.conf.j2` + `vhost.conf.j2` for each entry in `httpd_vhosts`,
  validates with `apache2ctl -t` / `httpd -t`.

You don't need to touch `webserver_service`, package names, or service
names when switching — those resolve automatically (see §6).

**To test httpd:** change this one line, then see the worked example in §7.

---

## 2. Shared variables (apply to both backends)

```yaml
webserver_index_content: "Welcome to My Web Server"
webserver_index_path: "{{ ... }}"   # auto-resolved, don't edit directly
```

- `webserver_index_content` — text shown on the default index page. Safe to
  change freely; it's just a string interpolated into `index.html.j2`.
- `webserver_index_path` — computed automatically from `webserver_package`
  (nginx → `/usr/share/nginx/html/index.html`, httpd →
  `/var/www/html/index.html`). You shouldn't need to edit this directly. If
  you do need a custom docroot, add a third key to the lookup dict rather
  than overwriting the whole variable.

---

## 3. nginx configuration (`nginx_*`)

Only used when `webserver_package: nginx`.

| Variable | What it controls | Change it when... |
|---|---|---|
| `nginx_user` | OS user nginx worker processes run as | Rarely — auto-detects Debian vs other |
| `nginx_service_name` | OS service name (always `nginx`) | Never — included for symmetry with `httpd_service_name` |
| `nginx_worker_processes` | nginx `worker_processes` directive | You want a fixed count instead of `auto` |
| `nginx_worker_connections` | Max connections per worker | Tuning for high traffic |
| `nginx_keepalive_timeout` | Seconds to hold keep-alive connections | Tuning client connection behavior |
| `nginx_client_max_body_size` | Max upload/request body size (e.g. `10m`) | You need larger file uploads |
| `nginx_gzip_enabled` / `nginx_gzip_types` | Enables compression and which MIME types get it | Adding a new content type to compress |
| `nginx_server_blocks` | List of virtual hosts (see below) | Adding/removing/editing a site |
| `nginx_upstreams` | Reverse proxy backend pools | Adding/editing a load-balanced backend |
| `nginx_proxy_*` | Timeouts/buffers for `proxy_pass` | Tuning reverse proxy behavior |
| `nginx_ssl_*` | TLS cert paths, protocols, ciphers, HSTS | Changing certs or hardening TLS |

### `nginx_server_blocks` — adding a site

Each item becomes its own file: `/etc/nginx/conf.d/<name>.conf`.

```yaml
nginx_server_blocks:
  - name: app.example.com          # filename stem; also used as identifier
    listen_port: 80
    server_name: app.example.com
    root: /usr/share/nginx/html
    index: index.html
    redirect_to_https: true        # if true: emits an 80->443 redirect only,
                                    # ignores root/index/proxy_pass entirely

  - name: api.example.com
    listen_port: 443
    server_name: api.example.com
    ssl_enabled: false             # set true to enable SSL on this block
    proxy_pass: "http://api_backend"   # must match a name in nginx_upstreams
```

A block is one of three things, decided by which keys you set:
1. **Redirect block** — `redirect_to_https: true` (ignores everything else)
2. **Reverse proxy block** — set `proxy_pass` to an upstream name
3. **Static block** — set `root` + `index`, omit `proxy_pass`

### `nginx_upstreams` — adding a backend pool

```yaml
nginx_upstreams:
  - name: api_backend                # referenced by proxy_pass above
    strategy: least_conn             # round-robin (omit), least_conn, ip_hash
    servers:
      - address: 10.0.1.11:8080
        weight: 1
```

---

## 4. httpd (Apache) configuration (`httpd_*`)

Only used when `webserver_package: httpd`. Mirrors the nginx structure above,
in Apache's own terms. **Scope note:** httpd support covers vhosts + SSL —
there's no reverse-proxy/upstream equivalent of `nginx_upstreams` yet.

| Variable | What it controls | Change it when... |
|---|---|---|
| `httpd_package_name` / `httpd_service_name` | OS package/service name (`apache2` on Debian, `httpd` on RHEL) | Auto-detected — don't edit |
| `httpd_user` / `httpd_group` | OS user/group Apache runs as | Auto-detected — don't edit |
| `httpd_server_admin` | `ServerAdmin` email in error pages | Set to your real ops contact |
| `httpd_start_servers`, `httpd_min_spare_threads`, `httpd_max_spare_threads`, `httpd_thread_limit`, `httpd_threads_per_child`, `httpd_max_request_workers` | MPM event worker tuning (rough equivalent of `nginx_worker_connections`) | Tuning for high traffic |
| `httpd_keepalive_timeout` | Seconds to hold keep-alive connections | Tuning client connection behavior |
| `httpd_client_max_body_size` | Max request body size, e.g. `10M` (converted to bytes for `LimitRequestBody`) | You need larger file uploads |
| `httpd_gzip_enabled` / `httpd_gzip_types` | Enables `mod_deflate` and which MIME types get it | Adding a new content type to compress |
| `httpd_vhosts` | List of virtual hosts (see below) | Adding/removing/editing a site |
| `httpd_ssl_*` | TLS cert paths, protocols, ciphers, session cache, HSTS | Changing certs or hardening TLS |

### `httpd_vhosts` — adding a site

Each item becomes its own file: `sites-enabled/<name>.conf` (Debian) or
`conf.d/<name>.conf` (RHEL-family). The playbook also creates each
non-redirect vhost's `document_root` automatically (owned by `httpd_user`/
`httpd_group`, mode `0755`) before deploying the vhost config — you don't
need to pre-create these directories yourself. Note this only creates the
empty directory; it does **not** deploy an index page into it (same as
nginx — `webserver_index_path` only covers the one default docroot), so a
freshly-created vhost will 403 until you put content there.

```yaml
httpd_vhosts:
  - name: app.example.com            # filename stem; also used as identifier
    listen_port: 80
    server_name: app.example.com
    document_root: /var/www/app.example.com
    redirect_to_https: true          # if true: emits an 80->443 redirect only,
                                      # ignores document_root/ssl_enabled

  - name: api.example.com
    listen_port: 443
    server_name: api.example.com
    ssl_enabled: true                # enables SSLEngine + cert directives
    document_root: /var/www/api.example.com
```

A vhost is one of two things, decided by `redirect_to_https`:
1. **Redirect vhost** — `redirect_to_https: true` (ignores `document_root`/`ssl_enabled`)
2. **Content vhost** — serves `document_root`; add `ssl_enabled: true` to turn on TLS

Note: unlike nginx, there's currently no `proxy_pass` equivalent for httpd
vhosts — only static content serving is supported.

> **Testing status:** the Debian (`apache2`) path has been exercised
> end-to-end (install → config → vhosts → `apache2ctl -t` → start →
> verified serving real HTTP responses). The RHEL-family (`httpd`) path is
> currently verified by template-rendering checks only, not a live install —
> treat it as best-effort until it's been run against a real RHEL/Rocky/Alma
> target.

---

## 5. SSL/TLS — both backends

Both `nginx_ssl_*` and `httpd_ssl_*` assume **certificates already exist on
the target host** at the configured paths — this playbook doesn't issue or
manage certs. In production, certs are typically delivered by a separate
process (certbot role, `community.crypto`, or pulled from a vault/secret
store), or terminated at a higher-level reverse proxy in front of these
hosts entirely.

To use real certs:
1. Get the cert/key onto the target at the paths in `nginx_ssl_cert_path` /
   `nginx_ssl_key_path` (or the `httpd_` equivalents), by whatever process
   you use for cert delivery.
2. Set `ssl_enabled: true` on the relevant server block / vhost.
3. Re-run the playbook.

---

## 6. Things that resolve automatically — don't hand-edit

These are computed from `webserver_package` and OS facts, so you never need
to set them per-environment when switching backends:

- `webserver_index_path` — dict lookup keyed on `webserver_package`
- `nginx_service_name` — static (`nginx` is the package and service name on
  every distro this playbook targets)
- `httpd_package_name`, `httpd_service_name` — `apache2` on Debian, `httpd`
  on RHEL-family
- `httpd_user`, `httpd_group`, `nginx_user` — OS-family-conditional

The playbook itself picks between the `nginx_*` and `httpd_*` versions of
package/service name based on `webserver_package`, so **switching the one
switch is genuinely the only required edit** — nothing else in this file
needs to change in lockstep with it.

If you need a one-off override for a specific environment (e.g. a custom
service name on an unusual distro), prefer setting it at a higher precedence
level (e.g. `host_vars` for that host) rather than editing the expression
itself, so the default stays correct for everyone else.

---

## 7. Switching backends on an already-configured host

If a host has previously run nginx and you switch it to httpd (or vice
versa), the playbook automatically stops and disables the service you're
switching *away* from before installing/starting the new one. This prevents
the new webserver failing to bind to port 80/443 because the old one is
still holding it — without this, you'd see an error like:

```
(98)Address already in use: AH00072: make_sock: could not bind to address [::]:80
```

This is fully automatic; you don't need to manually stop the old service
before re-running the playbook. On a host that's never run the other
backend at all, this task is a harmless no-op (it tolerates "service not
found" but will still surface a real error, e.g. a permissions problem, if
one occurs).

## 8. Worked example: switching to httpd for a test run

Minimal change — just the switch:

```yaml
webserver_package: httpd      # was: nginx
```

For a *first* test run, it's easiest to avoid SSL entirely until you've
confirmed Apache installs and serves correctly, since `api.example.com`
ships with `ssl_enabled: true` and no real cert will exist on a fresh host:

```yaml
httpd_vhosts:
  - name: app.example.com
    listen_port: 80
    server_name: app.example.com
    document_root: /var/www/app.example.com
    redirect_to_https: true

  - name: api.example.com
    listen_port: 80              # was: 443
    server_name: api.example.com
    ssl_enabled: false           # was: true
    document_root: /var/www/api.example.com
```

Run it:

```bash
ansible-playbook playbook.yaml
```

Once that succeeds, drop real certs on the target and flip `ssl_enabled`
back to `true` and `listen_port` back to `443` to bring TLS online.

To switch back to nginx at any point, it's the same one-line change in
reverse — `webserver_package: nginx` — no other edits required.

---

## 9. Error handling in `playbook.yaml`

All provisioning tasks are wrapped in a `block: / rescue: / always:`
structure (this is `playbook.yaml` structure, not a `group_vars` setting,
but documented here since it affects how failures during a run surface):

- **`block:`** — the normal task sequence (install, deploy configs,
  validate, start service). If any task in here fails, Ansible stops the
  block immediately and jumps to `rescue:`.
- **`rescue:`** — runs only on failure. Prints which task failed
  (`ansible_failed_task.name`) and the underlying error
  (`ansible_failed_result.msg`), then explicitly re-fails the play via
  `ansible.builtin.fail` so the run still ends in a failed state — this
  block reports the failure, it doesn't silently swallow it.
- **`always:`** — runs every time, success or failure. Currently just
  prints which `webserver_package` was targeted on which host, useful for
  scanning a long CI log.

There's also a **targeted failure check** specific to config validation,
separate from the generic `rescue:`: the `nginx -t` / `apache2ctl -t`
validation tasks use `failed_when: false` so they don't immediately trigger
`rescue:`, and are instead followed by a dedicated debug task
(`Report nginx/httpd configuration validation failure`) that prints the
actual validator stderr, then an explicit `ansible.builtin.fail` task that
stops the play. This gives a more specific error message for the single
most common failure mode (a bad template render or typo'd directive) before
falling through to the generic rescue message.

In practice, on a real failure you'll see, in this order:
1. The specific validation failure message (if it was a config error), or
   the raw task failure if it was something else (e.g. package not found)
2. The generic "Web server provisioning FAILED for `<package>` on
   `<host>`" rescue message
3. The play ending with `failed=1`, `rescued=1` in the recap — not a silent
   success
