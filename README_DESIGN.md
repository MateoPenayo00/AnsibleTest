# AnsibleTest — Design Document

Web server provisioning playbook supporting nginx and Apache (httpd) on a
single switch. This document covers the system end to end: what it does,
how to run it, every setting it exposes, and why it's built the way it is.

For a deep, line-by-line reference of every `group_vars` setting, see
`README_group_vars.md`. This document is the higher-level picture.

---

## 1. What this is

A flat (non-role) Ansible project that installs and configures a web server
— either **nginx** or **Apache (httpd)** — on every host in the
`webservers` inventory group. Which backend runs is controlled by a single
variable, `webserver_package`, so the same playbook and the same inventory
serve both backends without branching at the playbook-invocation level.

```
AnsibleTest/
├── ansible.cfg              # points inventory at ./inventory.yaml
├── inventory.yaml           # hosts in the `webservers` group
├── playbook.yaml            # the only playbook
├── README.md                 # one-line repo description
├── README_group_vars.md      # detailed group_vars reference
├── README_DESIGN.md          # this file
├── group_vars/
│   └── webservers.yml        # ALL configuration lives here
└── templates/
    ├── index.html.j2         # shared default index page (both backends)
    ├── nginx.conf.j2          # nginx global config
    ├── server_block.conf.j2   # nginx per-site config (looped)
    ├── httpd.conf.j2           # httpd global config
    └── vhost.conf.j2           # httpd per-site config (looped)
```

---

## 2. Quick start

```bash
cd AnsibleTest
ansible-playbook playbook.yaml
```

That's it, for the default configuration (nginx, two example sites, no real
TLS certs). To target specific hosts or limit the run:

```bash
ansible-playbook playbook.yaml --limit ansible-managed-node
ansible-playbook playbook.yaml --check          # dry run
ansible-playbook playbook.yaml --syntax-check    # validate without connecting
ansible-playbook playbook.yaml -K                # if sudo needs a password prompt
```

### Switching backends

Edit one line in `group_vars/webservers.yml`:

```yaml
webserver_package: httpd   # or: nginx
```

Re-run the playbook. If the host previously ran the other backend, the
playbook automatically stops and disables it first — see §6.4.

### Adding a site

- **nginx**: add an entry to `nginx_server_blocks` in `group_vars/webservers.yml`.
- **httpd**: add an entry to `httpd_vhosts` in the same file.

Both follow the same three patterns: redirect-only, reverse-proxy (nginx
only) or static-content. See `README_group_vars.md` §2b/§3b for the exact
shape of each entry.

---

## 3. Settings reference (summary)

Full detail, including every individual variable, lives in
`README_group_vars.md`. Summary:

| Variable group | Purpose | Read when |
|---|---|---|
| `webserver_package`, `webserver_index_*` | The backend switch + shared settings | always |
| `nginx_*` (core, server blocks, upstreams, SSL) | Everything nginx-specific | `webserver_package == 'nginx'` |
| `httpd_*` (core, vhosts, SSL) | Everything httpd-specific | `webserver_package == 'httpd'` |

Both `nginx_*` and `httpd_*` blocks always exist in the file regardless of
which is active — only the matching block is actually consumed by the
playbook at runtime. This means switching backends never requires deleting
or commenting out the other block.

### Settings NOT covered by this project (by design)

- **Certificate issuance/renewal** — `nginx_ssl_cert_path` /
  `httpd_ssl_cert_path` and their `_key_path` counterparts assume a valid
  cert/key already exists at that path on the target. No certbot,
  `community.crypto`, or ACME integration is included. In the target
  real-world scenario (`test-ansible.mateopenayo.dev`), TLS termination
  would more likely happen at the reverse proxy in front of this host
  rather than here.
- **Firewall rules** — nothing here opens ports 80/443 at the OS firewall
  level (ufw, firewalld, iptables). Assumed to be handled separately, or
  already open.
- **DNS** — server_name / ServerName values are config-only; no DNS record
  management is included.

---

## 4. Execution flow

```
1. Stop + disable the INACTIVE backend (if installed)
        │
        ▼
2. Install the SELECTED backend's package
        │
        ▼
3. Deploy index page (shared template, both backends)
        │
        ▼
4. Deploy backend-specific global config (nginx.conf / httpd.conf)
        │
        ▼
5. Remove the package's default/placeholder site
        │
        ▼
6. [httpd only] Create document_root for each non-redirect vhost
        │
        ▼
7. Deploy per-site configs (server blocks / vhosts), looped
        │
        ▼
8. Validate full config (nginx -t / apache2ctl -t)
        │
        ▼
9. If validation failed → targeted failure message → abort
        │
        ▼
10. Start + enable the service
        │
        ▼
11. [handler] Reload service if any deploy task reported a change
```

All of steps 2-10 happen inside a single `block:` in `playbook.yaml`; see
§6.5 for the error-handling wrapper around this sequence.

---

## 5. Design decisions

### 5.1 Single switch variable (`webserver_package`)

Considered alternatives: a separate `webserver_type` variable distinct from
the package name, or per-backend boolean flags (`nginx_enabled` /
`httpd_enabled`). Rejected both in favor of reusing the existing
`webserver_package` variable as the explicit switch — it already existed
from the nginx-only version of this project, and introducing a second
"which backend" concept would create two sources of truth that could
drift out of sync. The cost is that `webserver_package` now does double
duty (it's both a literal value sometimes used directly and a switch), but
that's mitigated by always resolving the *actual* package/service name
through `_webserver_package_name` / `_webserver_service_name` in the
playbook's `vars:` rather than ever using `webserver_package` directly as
an OS package name (see §5.3 for why that distinction matters).

### 5.2 Flat structure, no roles

This remains a flat (non-role) playbook by deliberate choice, consistent
with the project's original structure. A role-based layout (`roles/nginx/`,
`roles/httpd/`) would scale better if this grows further (e.g. adding a
third backend, or reusing this across multiple unrelated playbooks), but
for two backends and one playbook the flat structure keeps everything
visible in one place, which matched the project's "learn by doing,
understand every line" goal better than the abstraction a role would add.
**If this project grows a third backend or gets reused elsewhere, roles
would be the natural next refactor.**

### 5.3 Resolving package/service names instead of reusing `webserver_package` directly

nginx's package name, service name, and the value of `webserver_package`
are all the literal string `"nginx"` — that coincidence let earlier
versions of this playbook just use `webserver_package` directly as the
package/service name. httpd breaks that coincidence: the package is
`apache2` on Debian but `httpd` on RHEL-family, neither of which equals
`webserver_package`'s value of `"httpd"`. Rather than special-case httpd
while leaving nginx's old shortcut in place (which would leave the two
backends asymmetric and harder to reason about), both are now resolved the
same way: `nginx_service_name` was added (a static `"nginx"`) purely for
symmetry with `httpd_service_name` (which IS OS-conditional), so the
playbook's resolution logic (`_webserver_package_name` /
`_webserver_service_name` in `vars:`) treats both backends identically.

### 5.4 Scoping httpd to vhosts + SSL (no reverse proxy)

nginx's feature set here includes `nginx_upstreams` — named backend pools
that a server block can reverse-proxy to via `proxy_pass`. httpd has an
equivalent capability (`mod_proxy_balancer`), but it was deliberately left
out of scope for httpd in this pass. Reasoning: the project's near-term
target use case (publishing `test-ansible.mateopenayo.dev`) doesn't need
httpd-side load balancing, and adding it would roughly double the surface
area of `httpd.conf.j2`/`vhost.conf.j2` for a capability that isn't
currently needed. The nginx upstream pattern (`nginx_upstreams` +
`proxy_pass`) is left as a reference for what it would look like if this
need arises later.

### 5.5 OS-family branching inside templates, not separate template files per OS

`httpd.conf.j2` branches internally on `ansible_os_family` (`{% if
ansible_os_family == 'Debian' %} ... {% else %} ... {% endif %}`) rather
than having `httpd_debian.conf.j2` and `httpd_rhel.conf.j2` as separate
files selected by the playbook. This was chosen because most of the
file's content (MPM tuning, gzip, HSTS, SSL directives) is identical
between the two — only the structural bookending (ServerRoot, PidFile,
module-loading mechanism, port declarations) differs. Splitting into two
files would duplicate the ~70% that's shared and risk the two copies
drifting apart over time. The tradeoff is that the single file is denser
and requires reading the `{% if %}` blocks carefully — mitigated with
inline comments at each branch point.

### 5.6 `failed_when: false` on validation tasks, with an explicit follow-up `fail:` task

The config-validation tasks (`nginx -t`, `apache2ctl -t`) use
`failed_when: false` rather than letting Ansible's default failure
detection apply. This is *not* "ignore the failure" — it's "don't fail
*yet*". It lets the play continue one more step to a dedicated `fail:` task
that includes the validator's actual stderr in its message, giving a
specific, actionable error before the generic `rescue:` handler (which
only knows "some task failed") takes over. Without this two-step pattern,
a validation failure would jump straight to `rescue:` and the specific
stderr would only be visible in the raw task output above it, not
highlighted in the rescue summary.

### 5.7 Stopping the inactive backend automatically

Early in development, switching `webserver_package` on an
already-provisioned host caused the new backend to fail starting with
`Address already in use` — the previously active service was still bound
to port 80. Two options were considered: (a) document this as a manual
step ("stop the old service yourself before switching"), or (b) handle it
in the playbook. (b) was chosen because the failure mode is non-obvious
the first time you hit it (the error is about a port bind, not about the
fact that a *different* service still owns it), and because automating it
correctly is straightforward: resolve "the other backend's service name"
the same way `_webserver_service_name` resolves the active one, and
tolerate "service doesn't exist" (a host that's never run the other
backend) while still surfacing genuine failures like permission errors.

### 5.8 `block:` / `rescue:` / `always:` around the whole provisioning sequence

All provisioning tasks (install through service start) are wrapped in one
`block:` rather than scattering individual error-handling on each task.
This gives one place to define "what does a failure during provisioning
mean" (the `rescue:` handler) and "what should always happen regardless of
outcome" (the `always:` handler — currently just a run summary, but a
natural place to add e.g. a notification webhook later). The alternative
— per-task `ignore_errors` / `failed_when` everywhere — was rejected
because it spreads the failure-handling logic across the file instead of
having one clear boundary.

---

## 6. Known gaps and caveats

Documented here explicitly rather than left implicit, since these are the
things most likely to surprise someone extending this project.

### 6.1 RHEL-family httpd is template-verified only, not live-tested

The Debian (`apache2`) path has been run end-to-end against a real install:
package install, config deploy, `apache2ctl -t`, service start, and a
`curl` check confirming actual HTTP responses. The RHEL-family (`httpd`)
path has only been verified by rendering the Jinja templates with sample
data and checking the output is structurally sane — it has not been run
against a real RHEL/Rocky/Alma host. Treat it as best-effort until tested.

### 6.2 `mod_ssl` is not auto-enabled on Debian

Debian's `apache2` package does not enable `mod_ssl` by default
(`a2enmod ssl` is required). The playbook does not currently run this for
you. An SSL-enabled vhost will deploy and even pass `apache2ctl -t`
*degraded* (the `<IfModule mod_ssl.c>` guards mean a missing module is
silently skipped rather than erroring) — but it won't actually serve HTTPS
until the module is enabled manually:

```bash
sudo a2enmod ssl headers deflate
sudo systemctl restart apache2
```

### 6.3 Per-vhost/server-block document roots are not seeded with content

`webserver_index_path` only deploys `index.html.j2` to the *default*
docroot (`/usr/share/nginx/html` or `/var/www/html`). Any additional site
defined in `nginx_server_blocks` or `httpd_vhosts` gets its directory
created (httpd) or is expected to already exist (nginx), but nothing
deploys an index file into it — visiting that site will 403 (httpd) or
404/whatever nginx does for an empty static root, until something else
puts content there (e.g. a separate app-deployment playbook).

### 6.4 Certificates are not managed by this playbook

Both `nginx_ssl_cert_path`/`key_path` and `httpd_ssl_cert_path`/`key_path`
assume the cert/key already exist on the target. See §3, "Settings NOT
covered by this project."

### 6.5 Single shared TLS config per backend, not per-site

If multiple SSL-enabled sites exist, they currently all share the same
`nginx_ssl_*` / `httpd_ssl_*` cert and protocol settings (defined once
globally, not per-`nginx_server_blocks`/`httpd_vhosts` entry). This is fine
for the current example (one SSL site per backend) but would need
restructuring into per-site SSL variables if multiple sites with different
certs are needed on the same host.

---

## 7. Testing this project

### Validating without a real target

```bash
ansible-playbook playbook.yaml --syntax-check
python3 -c "import yaml; yaml.safe_load(open('group_vars/webservers.yml'))"
```

### Forcing a deliberate failure (to test the block/rescue/always handling)

Two ready-made broken configs exist for this:

- `webservers_FAIL_nginx.yml` — invalid `nginx_ssl_session_cache` value
- `webservers_FAIL_httpd.yml` — invalid `httpd_ssl_session_cache` value
  (note: on a fresh Debian host with `mod_ssl` not yet enabled, you'll see
  a different, also-real error about `mod_ssl` not being loaded — see §6.2)

```bash
cp group_vars/webservers.yml group_vars/webservers.yml.bak
cp webservers_FAIL_nginx.yml group_vars/webservers.yml   # or _httpd
ansible-playbook playbook.yaml
# Expect PLAY RECAP to show failed=1, rescued=1 — that's success for this test.
cp group_vars/webservers.yml.bak group_vars/webservers.yml
```

---

## 8. Where this could go next

Not built yet, deliberately deferred — listed here as the logical
continuation points if this project keeps growing:

- **Ansible Vault** for cert/key material instead of assuming they're
  pre-placed on the target.
- **Live RHEL/Rocky/Alma testing** of the httpd path (§6.1).
- **`a2enmod` automation** for Debian SSL (§6.2).
- **Reverse proxy / upstream support for httpd** (`mod_proxy_balancer`),
  mirroring `nginx_upstreams` (§5.4).
- **Per-site SSL config** instead of one shared cert/protocol set per
  backend (§6.5).
- **The real target scenario**: publishing `test-ansible.mateopenayo.dev`
  via the reverse proxy at `192.168.100.11` (holding the wildcard
  `*.mateopenayo.dev` cert) in front of this web server VM at
  `192.168.100.14`.
- **Roles**, if a third backend or reuse across playbooks ever becomes a
  real need (§5.2).
