---
name: cp-integrator
description: >-
  Integrate Imunify360 with a hosting control panel it does not support natively, through
  its generic-panel integration (integration.conf plus users/domains/admins/panel_info
  scripts): discover how the panel works on a real test server, install and prove the
  integration there, and hand back an installable package with its README. Use when asked
  to integrate Imunify360 with a hosting panel, write integration.conf or its scripts for a
  panel, or validate such an integration on a server. Not for cPanel, Plesk or DirectAdmin,
  which Imunify360 supports directly.
---

# Integrate Imunify360 with a control panel

**Done means:** on the user's test server, Imunify360 runs in generic-panel mode and
sees the panel's real users and domains, detects malware in their sites, serves its UI
to the panel's users, and applies its WAF; and the user holds a package that reproduces
this on their other servers, plus a report that proves each of those claims with output
from the server. The package carries a selftest that re-runs those proofs on a server it is
installed on, so the user can check a new panel or Imunify360 version without you.

You have the panel's documentation, the server, and [`reference/runtime.md`](reference/runtime.md),
which carries what Imunify360's generic-panel runtime reads and the facts its public docs
get wrong. Read it before you design anything. The panel-specific work (where the panel
keeps users and sites, how it builds web-server config) is yours to find out.

## Inputs

Confirm what the user already gave you; ask only for what is missing:

- **The panel** and where its documentation is (a URL, local files, or none).
- **SSH access as root to a test server running the panel.** Ask whether it is
  disposable: the run creates panel accounts and sites, installs packages, changes
  web-server configuration and writes the harmless EICAR test file. Never use a
  production server.
- **An Imunify360 licence key in a local file** (a trial key works), unless Imunify360 is
  already installed there. Never ask for it in chat text, echo it, or write it into any
  file you produce.

## The run directory and the evidence log

Work in `cp-integrator-runs/<panel>-<YYYYMMDD-HHMM>/` under the current directory, and keep
every file you create on this machine inside it. Put two files there first: `ssh.conf`, an
ssh_config with a `Host target` entry (HostName, User root, Port, IdentityFile), and a copy
of [`scripts/remote`](scripts/remote). Then, from the run directory, run **every command on
the server** through it:

```sh
./remote '<command>'                                        # -n 40 prints only 40 lines; the log keeps all
./remote '(umask 077; cat > /root/.imunify360-key)' < key-file   # stdin passes through
```

It appends the command, its exit code and its verbatim output to `evidence.md`
as a numbered entry (`E12`) and prints its number and the link that cites it,
`[E12](evidence.md#e12)`. Every claim you make to the user cites the entry that proves it,
and in `facts.md` and `report.md` every citation is that link, so the user clicks through
to the output; that is how the user can tell a verified fact from a plausible one. Keep
both files in the run directory next to `evidence.md`, where the links resolve.

The log redacts nothing: keep secrets out of commands and outputs. Read them from files on
the server, print a config file's keys rather than its values, filter listings that print
passwords or API keys, and never type a secret's value into a command, not even one that
redacts it. Scope `umask 077` to the command that writes a secret, `(umask 077; ...)`:
panel tools run under it create directories their own services cannot read.

## 1. Discover, then stop at Gate 1

Find out, and prove on the server, everything the integration will rely on. Docs suggest;
the server decides. Start by **creating a test account with a website through the panel
itself** (its CLI or API, not `useradd`), and `stat` the document root it reports: every
enumeration you then find must return that account, because an empty or wrong-but-valid
result looks just like a panel with no users.

What you need to know:

- how to list the panel's users, its domains (each with owner and document root), its
  administrators, and its name and version, in a machine-readable way;
- how to create an account with a website (you will need more test accounts later);
- the web server that serves PHP sites, how the panel reloads it, how to test its config,
  whether a ModSecurity engine exists or can be installed for that exact build, and where
  the client IP comes from behind any proxy;
- where a per-domain config file can be included in one site's virtual host **so that it
  survives the panel regenerating that vhost** (prove it: add a comment-only include,
  make the panel rebuild the site, check it is still there, remove it);
- where tenant sites live (the malware scan roots), which PAM service exists for UI logins,
  and how the UI will be served (port or hostname, PHP with `proc_open`, TLS, firewall);
- what installing Imunify360 will change on this panel (firewall, fail2ban, packages).

Before Gate 1 you may only make reversible changes: test accounts and sites created
through the panel, a graceful web-server reload, temporary probe files or comment-only
includes that you remove again. Install nothing.

Write `facts.md` for the user: one row per fact with its value, the evidence entry that
proves it, and a short quote of the decisive output line; flag anything that is a guess
or depends on version. Then the plan: the scripts and what each wraps, `integration.conf`
values, the WAF and per-domain approach, how the UI is served, how the selftest gets its
test accounts (created through the panel and removed again, or prepared by the admin),
what will be installed and changed on the server, and what is out of scope. **Present both
and stop until the user approves.** If they change the design, update `facts.md` and the plan before going on.

## 2. Build, install, check

Write the package in `<run-dir>/package/` (layout and rules in
[`reference/runtime.md`](reference/runtime.md) §Package): `integration.conf`, the
integration scripts (Python 3 stdlib or POSIX sh only), `install.sh`, `uninstall.sh`,
`selftest.sh`, `VERSION`, `README.md`. Install it on the server with its own `install.sh`,
exactly as the user will, then install Imunify360 with its installer, then check every script
through the runtime's own code (the check in §Checking it works). Fix the package, never
the server by hand: a fix that is not in the package is one the user never gets.

The scripts run as root over names panel users control. Treat every username, domain and
ModSecurity content as hostile: no shell, no string-built commands, validate names before
they touch a path, write atomically, and test the web-server config after any write,
restoring the previous file if the test fails.

## 3. Prove it works

Proofs 1–4 are the package's `selftest.sh` (rules in [`reference/runtime.md`](reference/runtime.md)
§Package): it creates its test accounts through the panel, at least one with something an
integration can get wrong (a subdomain, an alias, or an FTP/mail-only user), confirms each
test site serves PHP, checks the four proofs and removes what it created. Run it on the
server through `./remote`: that entry is the evidence for proofs 1–4 and the proof that
the selftest itself works. A check that cannot fail proves nothing, so compare against
the panel's own lists, never against your scripts' output, and see each check fail once
(say, with the test rule still disabled) before you trust its pass.

1. **Enumeration.** The users and domains Imunify360 reports through the runtime equal
   the panel's own lists, by name; root and the `admins` names are administrators.
2. **Malware.** EICAR written into a panel user's document root is detected; name the
   scanner that found it.
3. **UI.** It is served where the plan said; a panel user logs in through PAM and sees
   their own data, an administrator sees everyone's.
4. **WAF.** Imunify360's rules are the only ruleset loaded; a test attack on a tenant site
   is blocked and audit-logged; a rule disabled for one domain lets that domain through
   while another stays blocked. If the panel has no per-domain hook, say so with the Gate-1
   evidence.
5. **Package lifecycle.** A second `install.sh` changes nothing; uninstalling in the
   README's order leaves the panel serving its sites with nothing left behind; reinstalling
   brings everything back: the selftest passes again.

Finish with a selftest run of the final version that passes, and every test site still
serving PHP. When a proof fails, fix the package (the selftest included), bump `VERSION`,
install it as an update, and re-run the selftest.
Whatever the generic-panel surface cannot do is a known limit, not something to work
around by changing Imunify360.

## 4. Deliver

Write `report.md`: environment (panel, OS, web server, Imunify360 versions), each proof
with its result and the evidence entries behind it (for proofs 1–4, the final selftest
run), the fixes made on the way, and the known limits. Finish the package README
(§Package) with the exact commands that worked, and pack
`<panel>-imunify360-integration-<version>.tar.gz` (owner root, no build droppings). Leave the integration installed and working on the test server.

In your final message give the user the tarball and report paths, the known limits in
two or three lines, how to open the UI, how to re-run the selftest after a panel or
Imunify360 upgrade, and a reminder that the run directory quotes live server output and is
theirs to keep private. Offer to remove the test accounts and the
key file from the server.

## When you are blocked

Stop and ask when you need something you cannot get from the docs and the server: a
licence, a credential, a decision between two plausible panel conventions, an install
that needs a browser. Before asking for a panel credential or API token, look for a way
root can create one on the server with the panel's own CLI or API; write it to a file only
root can read and never print it. Do not guess and do not change Imunify360 itself.
