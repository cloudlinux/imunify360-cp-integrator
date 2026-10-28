# Imunify360's generic-panel runtime

What the runtime reads, how, and what it does on a real server. Sources: Imunify360's
public docs (the page "Generic panels and no-panel installation and integration"; the
site blocks bots, so read its source,
`https://raw.githubusercontent.com/cloudlinux/imunify360-documentation/master/docs/control_panel_integration/README.md`)
and the runtime itself, which ships as readable Python on every server running Imunify360.
**Where they disagree, the installed runtime wins.** Last checked against 8.14.1; once it
is installed, check any line you rely on:

```sh
P=$(/opt/imunify360/venv/bin/python3 -c 'import os, defence360agent as a; print(os.path.dirname(os.path.dirname(a.__file__)))')
# $P/defence360agent/subsys/panels/generic/panel.py   scripts, validation, users/domains/admins
# $P/defence360agent/subsys/panels/generic/users_script_schemas/   the output schemas
# $P/defence360agent/api/integration_conf.py   parsing   $P/defence360agent/api/pam_auth.py   UI login
# $P/im360/subsys/panels/generic/mod_security.py   ModSecurity, per-domain script
```

## integration.conf

`/etc/sysconfig/imunify360/integration.conf`. **Its existence alone** switches Imunify360
into generic-panel mode; the installer refuses to run without it (with a working
`panel_info` and `[paths] ui_path`). INI via `configparser`: section names are
case-sensitive.

| Key | Notes |
|---|---|
| `[integration_scripts] panel_info, users, domains, admins` | commands; see Scripts |
| `[integration_scripts] modsec_domain_config_script` | per-domain WAF; see ModSecurity |
| `[web_server] server_type` | `apache`, `nginx`, `litespeed` or `openlitespeed` |
| `[web_server] graceful_restart_script`, `config_test_script` | split on **whitespace** into argv, no shell. Prefer the panel's own reload command |
| `[web_server] modsec_audit_log` | must be set; `modsec_audit_logdir` only for `SecAuditLogType Concurrent` |
| `[malware] basedir` | realtime scanner's watch roots, **space**-separated (a comma makes one nonexistent path, silently). To exclude panel or database dirs under it, add them to `/etc/sysconfig/imunify360/malware-filters-admin-conf/ignored.txt` **after** Imunify360 is installed (the file is a package conffile; creating it first breaks the install), then `imunify360-agent malware rebuild patterns` |
| `[paths] ui_path` | where the installer copies the UI, at install time only: must be absent or an empty dir, not a symlink. Optional `ui_path_owner = user:group` |
| `[PAM] SERVICE_NAME` | PAM service for UI login. **Upper-case `[PAM]`**: the docs' `[pam]` is ignored and login silently uses `system-auth`, which does not exist on Debian/Ubuntu (use `common-auth` there) |

`[malware] pattern_to_watch`, `[panel]`, `[panel_ports]`, `[panel_login]`, `[features]`,
`[smtp]`, `[metadata]` are documented or parsed but nothing in the generic panel uses them.

## Scripts

The value is split shell-style into argv; argv[0] must be an **absolute path to an
existing executable**; it runs as root with **no shell**; non-zero exit is an error.
stdout must be one UTF-8 JSON object `{"data": ..., "metadata": {"result": "ok"}}`
(`result` other than `ok` is an error; `metadata.message` is logged). Output is cached for
90 s, and the CLI is answered by the separate `imunify360-agent` service: after changing a
script or the config, `systemctl restart imunify360 imunify360-agent`.

| Script | `data` | Required | Also used |
|---|---|---|---|
| `panel_info` | object | `name`, `version` | |
| `users` | list | `username` | `email`, `locale_code`, `home` |
| `domains` | object keyed by domain | `owner` | `document_root` (needed for scans and the UI), `is_main` |
| `admins` | list | `name`, `is_main` (bool) | `email` |

- **Silent fallback.** If `users` or `domains` fails in any way, the runtime logs a warning
  and uses the server's non-system Unix users instead. A broken integration then looks
  like a working one; only comparing names against the panel's own lists catches it.
- Every domain's `owner` is added as a user, so `users` and `domains` must agree on names.
- **Aliases sharing a document root.** Domain lists and each user's domain view take every
  domain `domains` reports, but the runtime's map from document root to domain keeps one
  domain per path: the last one emitted. Report aliases and parked domains, emit each
  site's primary domain after the aliases that share its root, and keep that order (no
  `sort_keys`, no `jq -S`). Per-domain WAF rules for an alias served by its primary's
  vhost apply to that whole vhost: keep each domain's overrides in a file of its own, so an
  alias's empty `content` cannot remove the primary's, and say so in the known limits.
- **Who sees what is by name.** The UI authenticates the typed name and password with PAM
  and issues a token for that name. Administrators are `root`, the `admins` names and the
  lines of `/etc/sysconfig/imunify360/auth.admin`; anyone else is a client and sees the
  Imunify360 user of the same name. So a tenant can use the UI only if their login name is
  a Unix account PAM accepts **and** a `username` your `users` script reports; a panel whose
  customers have no Unix login cannot give them the client view by password (a known limit,
  or a decision for Gate 1). As root, `imunify360-agent login get --username <unix user>`
  issues a token without a password, valid as the UI API's Bearer token (check before you
  document opening the UI with it as `<ui>/#/?token=<token>`). A panel admin whose name
  equals some tenant's Unix user would make that tenant an administrator: leave such names
  out of `admins`.
- `Empty integration script path for admins` in `/var/log/imunify360/error.log` is the
  runtime also asking the optional `/opt/cpvendor/etc/integration.ini`; harmless.

## ModSecurity

- Imunify360 writes its ruleset include to `/etc/sysconfig/imunify360/generic/modsec.conf`
  and rewrites it whole. The directory and an empty file must exist **before** Imunify360
  is installed, the web server must include it, and the package must never ship that file
  (it would overwrite the rules). `i360deploy.sh --purge` deletes it, after which the web
  server's config test fails until the include is removed.
- **On Apache, guard the include until its target exists.** During Imunify360's install the
  agent writes `generic/modsec.conf` with a plain `Include` of `generic/modsec2.imunify.conf`
  before the package that ships that file is unpacked; Apache's reload then fails and the
  agent restarts it, taking every site down for about 90 s. Wrap your include in
  `<IfFile /etc/sysconfig/imunify360/generic/modsec2.imunify.conf>`. Do not pre-create that
  file: it is a dpkg conffile, and the install then stops at a conffile prompt.
- Engine settings the docs require: `SecRuleEngine On`, `SecAuditEngine RelevantOnly`,
  `SecConnEngine Off`, `SecAuditLog` = `modsec_audit_log` (JSON format on nginx). Wrap
  Apache directives in `<IfModule security2_module>`.
- **Imunify360's must be the only ruleset.** Distribution packages pull in OWASP CRS as a
  recommended dependency (`libapache2-mod-security2` recommends `modsecurity-crs`); install
  without recommends and confirm with `apache2ctl -t -D DUMP_INCLUDES` / `nginx -T`.
- Engine for nginx: a packaged connector only loads into the nginx build it was compiled
  for. If the panel ships its own nginx, build ModSecurity-nginx against that exact version
  (needs `--with-compat` in `nginx -V`) and ship a guard for nginx upgrades. If neither works,
  the WAF is a known limit.
- With nginx in front of Apache, the WAF belongs in the backend that runs PHP; static files
  served by the frontend bypass it (a known limit). If the server trusts `X-Forwarded-For` /
  `X-Real-IP` from anywhere, clients can spoof the IP Imunify360 blocks: recommend trusting
  only the proxy's addresses.
- **`modsec_domain_config_script`** is split on whitespace, run without a shell, and gets
  **JSON lines on stdin**, one per domain: `{"domain": "...", "content": "..."}` (no `user`
  key, whatever the docs show). `content` is already wrapped for the server type (on Apache in
  `<IfModule security2_module>` / `security3_module` blocks) and may be empty, meaning "remove
  this domain's overrides". Write it, as given, where the panel includes it in that domain's
  vhost (a place that survives the panel regenerating the vhost), test the config, restore on
  failure, reload, exit 0; on failure exit 1 with the reason on stderr. stdout is ignored. Skip
  domains the panel does not own. If the panel has no surviving per-site include, write a
  server-wide file loaded before Imunify360's rules, keyed on the host, e.g.
  `SecRule REQUEST_HEADERS:Host "@rx ^(www\.)?example\.com(:\d+)?$" "id:10001,phase:1,pass,nolog,t:lowercase,ctl:ruleRemoveById=77140164"`.

## The UI

- The installer only copies the UI (a single-page app plus a PHP API) into `ui_path`;
  serving it is the integration's job. Create `ui_path` empty with mode 0755 yourself (a
  restrictive umask during install otherwise leaves it 0700).
- Its PHP calls `proc_open` to run `<ui_path>/bin/execute.py` with
  `/opt/imunify360/venv/bin/python3`, which talks to the agent over a world-writable socket,
  so any unprivileged PHP user works. Panels often put `proc_open` in `disable_functions` in
  a shared `php.ini`, and a pool can only add to that list. Do not weaken tenants' PHP: run
  the UI on its own PHP-FPM master with its own `php.ini`, systemd unit and
  `RuntimeDirectory=` name (`RuntimeDirectory=php` takes over `/run/php` and breaks every
  tenant pool).
- The UI's `.htaccess` carries three things a non-Apache or `AllowOverride None` vhost must
  provide itself: `DirectoryIndex index.php`; the `Authorization` header passed to PHP
  (nginx: `fastcgi_param HTTP_AUTHORIZATION $http_authorization;`); `api/...` routed to
  `api.php`, with `api/` and `swagger/` refused from `127.0.0.1`/`::1`.
- Serve it preferably on its own port with TLS, opened through the panel's own firewall
  tool (or the panel's next firewall rebuild drops it), or on its own hostname; not as an
  alias inside a panel-managed vhost.
- After first login an administrator accepts the EULA in the UI; that is the user's call.

## Proactive Defense

Proactive Defense is a PHP module for Apache and LiteSpeed and needs Hardened PHP
(alt-php) on the host (public docs, Dashboard, Proactive Defense). If tenant sites run
another PHP, or run PHP in containers or namespaces, it is a known limit.

## Installing and uninstalling Imunify360

```sh
wget -q -O /root/i360deploy.sh https://repo.imunify360.cloudlinux.com/defence360/i360deploy.sh
bash /root/i360deploy.sh --key "$(cat /root/.imunify360-key)"   # needs integration.conf first; several minutes
bash /root/i360deploy.sh --uninstall --yes                       # keeps config and state
```

- Run the installer detached (`setsid nohup ... >/root/i360deploy.out 2>&1 </dev/null &`) and
  poll by the PID you saved (`pgrep -f i360deploy` matches your own polling command and never
  reports it done); it outlives an SSH session. Success ends with a line ending in `Success`.
- The key is in the installer's argv while it runs (never list process arguments then) and
  ends up in `/var/log/i360deploy.log` and `/var/log/imunify360/console.log`. Scrub by
  value afterwards: `K=$(cat /root/.imunify360-key); sed -i "s/$K/<REDACTED>/g" <both files>;
  grep -rlF -- "$K" /var/log | wc -l` must print 0.
- `--uninstall` without `--yes` prompts forever when stdin is closed; `--yes` also lets apt
  autoremove unrelated unused packages (say so in the README).
- Observed side effects: the agent stops and disables `fail2ban` and `--uninstall` does not
  re-enable it; `imunify360-dos-protection` may fail its first start (restart it once); the
  panel flushing its firewall drops Imunify360's chains until the agent restores them (up to
  a minute); with nginx, reloads during rule writes can fail transiently and leave
  `run-*.service` units in `systemctl --failed` (`systemctl reset-failed`).
- `--uninstall` leaves behind the `php-els` apt source
  (`/etc/apt/sources.list.d/php-els.list`, `/etc/apt/auth.conf.d/php-els.conf`), which then
  answers 403 and makes the next install abort: remove both before reinstalling. Also left:
  `imunify360.list`, `imunify-rollout.list`, `alt-common-els.list` sources, packages
  `alt-common-release`, `els-php-release`, `ipset`, packages in dpkg state `rc`. "Is
  Imunify360 still installed" is `dpkg-query -W -f '${db:Status-Abbrev}' imunify360-firewall`
  starting with `ii`, not `dpkg -s`.
- If Imunify360 was installed before `integration.conf` existed, its UI was never deployed
  to `ui_path`: reinstall it after the package.

## Package

```text
<panel>-imunify360-integration-<version>/
  README.md  VERSION  install.sh  uninstall.sh  selftest.sh
  files/     every file at the absolute path it installs to, e.g. files/etc/sysconfig/imunify360/integration.conf
```

- `install.sh`: preflight (root, OS, the panel's own binary, the web server it was built
  for), then act. Idempotent; a newer version run over an older one is the update path and
  removes files the new one no longer ships. Record the version, every file and directory
  it created (`/etc/sysconfig` does not exist on Debian/Ubuntu) and every edit in
  `/var/lib/<package>/`, so `uninstall.sh` reverses exactly that. Back up any file it edits
  but does not own. Create `generic/modsec.conf` empty if missing and never overwrite it.
  After any web-server edit run the config test and roll back on failure. Record whether
  `fail2ban` was enabled and running, and the packages and modules it enables (Ubuntu's
  ModSecurity package also enables Apache's `unique_id`), so uninstall can restore them.
  Read `VERSION` into a variable of another name: sourcing `/etc/os-release` sets `VERSION`.
  Run Python scripts with `python3 -B` so no `__pycache__` is left for uninstall to miss.
- `uninstall.sh`: the exact reverse; refuses while Imunify360 is installed unless `--force`
  (removing `integration.conf` under a running agent switches it out of generic mode), and
  removes the `modsec.conf` include first so a purged Imunify360 cannot break the web server.
- `selftest.sh`: proofs 1–4 (enumeration, malware, UI, WAF) with the commands in §Checking
  it works, run as root on the server from the unpacked package, with no agent and nothing
  interactive. Not installed by `install.sh`; it tests whatever version is installed. It
  briefly gives a test account Imunify360 admin rights, disables a WAF rule for a test
  domain and triggers a malware alert, so it starts by saying so and refuses to run without
  `--yes`; the README says to run it on a test server or in a maintenance window.
  - **Test data:** each run creates its accounts and sites through the panel under a
    prefix no real account uses plus a run id, so nothing from an earlier run can satisfy a
    check (match malware hits by this run's path and time). It starts by removing leftovers
    of the prefix from an interrupted run, and on exit, after a failure or a signal too
    (trap `EXIT INT TERM HUP`), removes what it created and reverts only what it added: an
    `auth.admin` line, a rule it disabled, a whitelist entry that was not there before. If
    the panel cannot create accounts unattended, it takes prepared accounts as arguments
    and the README says how to prepare them. It changes nothing else: no fixes, and the
    only restart is `imunify360 imunify360-agent` after creating accounts, since script
    output is cached (§Scripts). Passwords and tokens go in `0600` files and are never
    printed, in the raw output either.
  - **HTTP checks** (UI, WAF) go to the server's own routable address, not loopback (the
    UI API refuses `127.0.0.1`/`::1`). Check on the first run that a WAF attack from that
    address is blocked and audit-logged; if it is not, the selftest takes a client host to
    send the requests from (`--client user@host`, over ssh) and the README says so.
  - **Output:** a header with what was tested (panel name and version from `panel_info`,
    Imunify360 version, the package version `install.sh` recorded on the server, OS, web
    server and its version), then one line per
    check, `PASS|FAIL|SKIP <check>: <the decisive value>`, followed by the raw output behind
    it. Plain text, so two runs on different versions can be diffed. Exit 0 only if no
    check failed; `SKIP` only for a documented known limit, with the reason.
- `README.md` for an admin who never saw this run, with sections **Requirements** (panel
  version, OS, web server it was proven on; engine, PHP, licence), **Install** (package, then
  Imunify360, then checks), **Update**, **Uninstall** (Imunify360 first, then the package;
  what remains), **Testing** (running `selftest.sh`, what it creates and changes for the
  duration of the run, reading its output, re-running it after any panel, Imunify360 or
  package update), **Configuration recommendations** (each with its reason: UI exposure and
  TLS, admins, re-running `install.sh` after panel upgrades, the key scrub), **Known limits**,
  and **Troubleshooting**. No secrets, server addresses, the test server's hostname, test
  account names or test passwords.

## Checking it works

These are the checks `selftest.sh` automates; run them by hand while you build.

**Each script through the runtime's own code** (argv rules, JSON, schemas), raising
instead of falling back:

```sh
/opt/imunify360/venv/bin/python3 - <<'EOF'
import asyncio, json
from defence360agent.api.integration_conf import IntegrationConfig
from defence360agent.subsys.panels.generic.panel import _get_conf_path, _get_integration_data
for s in ("panel_info", "users", "domains", "admins"):
    print(s, json.dumps(asyncio.run(_get_integration_data(s, _get_conf_path(IntegrationConfig(), s))))[:400])
EOF
```

- Enumeration as Imunify360 sees it: `imunify360-agent malware user list --json` (until the
  EULA is accepted, root's `--json` output carries its 50 KB text: keep only `items`), and the
  domains via the UI API below. Compare names with the panel's own CLI or API.
- Malware: `imunify360-agent malware malicious list --json` shows each hit's path and
  `scan_type` (realtime usually wins the race against an on-demand scan you start). An
  on-demand scan: `imunify360-agent malware on-demand start --path <dir>`.
- UI API, from a non-loopback address: `POST <ui>/api/login/pam` with
  `{"params": {"username": ..., "password": ...}}` answers `{"result": "success", "data":
  {"items": "<token>"}}`; send `Authorization: Bearer <token>` with `POST
  <ui>/api/malware/user/list` and `{"params": {}}`. An admin-only call with a client's token
  answers HTTP 502. Keep passwords in files on the server. On a key-only server root has no
  password: test the admin role by adding a test panel user to `auth.admin`, then restore it.
- WAF trigger from the public docs:
  `curl -s -o /dev/null -w '%{http_code}' 'http://<site>/?i360test=88ff0adf94a190b9d1311c8b50fe2891c85af732'`.
  Blocked (403 or 406, the engine's deny status) plus an audit-log entry with the client's
  real IP is the proof. Whitelist your own source IP before attacking from it, so Imunify360
  does not block your SSH. Requests from the server itself or from whitelisted IPs never
  become incidents (`imunify360-agent get --period 1h`); do not chase them. Per-domain:
  `imunify360-agent rules disable --id <id> --plugin modsec --name test --domains <domain>`,
  wait a second for the reload, then `imunify360-agent rules enable --id <id> --plugin modsec`.
