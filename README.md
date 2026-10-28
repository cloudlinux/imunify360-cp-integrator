# Imunify360 control-panel integration skill

An agent skill that integrates **Imunify360** with a hosting control panel it does not
support natively, through Imunify360's generic-panel integration. You point it at a test
server running your panel and at the panel's docs; it works out how the panel stores
users, domains and web-server configuration, shows you what it found with the server
output that proves each fact, and after your approval writes the integration, installs it
with Imunify360, proves it works, and hands you an installable package.

It uses public material only: [Imunify360's documentation](https://docs.imunify360.com/control_panel_integration/),
your panel's documentation, and the server.

## What you need

- An agent that runs skills, such as [Claude Code](https://docs.claude.com/en/docs/claude-code).
- A **disposable test server** running your panel, reachable over SSH as root. The run
  creates panel accounts and websites, installs packages, changes web-server configuration
  and writes the harmless EICAR test file. Never point it at a production server.
- An **Imunify360 licence key** in a file (a trial key works), unless Imunify360 is
  already installed there.
- `ssh` on the machine the agent runs on.

## Install the skill

In Claude Code:

```text
/plugin marketplace add cloudlinux/imunify360-cp-integrator
/plugin install imunify360-cp-integrator@imunify360-cp-integrator
```

Or copy the skill where Claude Code looks for skills:

```sh
git clone https://github.com/cloudlinux/imunify360-cp-integrator
mkdir -p ~/.claude/skills && cp -r imunify360-cp-integrator/skills/cp-integrator ~/.claude/skills/
```

Other agents: point them at [`AGENTS.md`](AGENTS.md).

## Use it

> Integrate Imunify360 with AcmePanel on root@203.0.113.10 (key `~/.ssh/id_ed25519`), a
> disposable test VM. The panel's docs are at <https://docs.acmepanel.example/>. My
> Imunify360 key is in `~/imunify360.key`.

The run stops once for you:

1. **Discover.** The agent probes the server and the docs, creates a test account through
   the panel, and shows you `facts.md`: every fact the integration relies on, each with the
   numbered entry in `evidence.md` (the exact command and its verbatim output) that proves
   it, plus the plan of what it will install and change. **You approve or correct it.**
2. **Build and prove.** It writes the package, installs it and Imunify360 on the server,
   and proves that Imunify360 sees the panel's users and domains, detects malware in their
   sites, serves its UI to panel users and applies its WAF, and that the package installs,
   updates and uninstalls cleanly. The first four proofs run as the package's own
   `selftest.sh`, which you keep.
3. **Deliver.** You get `report.md` (each proof with its evidence) and
   `<panel>-imunify360-integration-<version>.tar.gz` with install, update and uninstall
   steps, configuration recommendations and known limits. The integration stays installed
   on the test server for you to try.

Everything lands in `cp-integrator-runs/<panel>-<timestamp>/` in your working directory.
It quotes live server output: keep it private.

## What it does not do

- It does not change Imunify360. What the generic-panel integration cannot reach is
  reported as a known limit.
- It integrates the web server your test server runs; other stacks need a run on a server
  with that stack.
- It does not touch your production servers; you deploy the package yourself.

## Issues and licence

Report problems and ideas in this repository's issues. Run directories quote live server
output: never attach one whole. Released under the [MIT licence](LICENSE).
