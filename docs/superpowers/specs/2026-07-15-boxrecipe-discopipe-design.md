# boxrecipe + discopipe-on-box — design

Date: 2026-07-15
Status: draft (pending review)

## Goal

Run multiple services on the single Hetzner VM ("doyu-box"), each with its
own domain, composed and deployed in one shot from a recipe. First new
service: discopipe (the Discord ↔ headless `claude -p` pass-through bot),
migrated from the laptop to the box. Some services stay VPN-only until
mature, then get published by flipping a flag.

## Decisions (made during brainstorming)

1. **Placement**: discopipe runs on the existing doyu-box (not a dedicated
   VM), alongside WireGuard + Caddy + reconcile.
2. **claude auth on the box**: `ANTHROPIC_API_KEY` in the discopipe env
   file (token-metered billing, simplest headless setup).
3. **Bot identity**: migrate the existing bot — port the laptop's
   `~/.config/discopipe/env` values (same `DISCORD_TOKEN`, channel,
   user; `DISCOPIPE_CWD` changes to the box's agent dir and
   `ANTHROPIC_API_KEY` is added); stop the laptop instance **before**
   the box bot starts — two gateway sessions on one token would both
   reply, duplicating every answer.
4. **Install method**: baked into the cloud-init recipe (not post-boot
   manual install, not Docker) so the box stays rebuildable from
   recipe + one secret file.
5. **Repo split — full separation**:
   - `hetznerinit` becomes a pure library (core / server / wireguard /
     caddy). `nbs/10_deploy.ipynb` is removed.
   - New nbdev repo **`boxrecipe`** owns the composition: SERVICES
     registry, per-service cloud-init fragments, `box_recipe()`, PEERS,
     and the live rebuild flow. Depends on hetznerinit via
     `git+https://github.com/doyu/hetznerinit.git@<tag-or-SHA>`
     (pinned; bumped deliberately so a rebuild is reproducible).
6. **Dedicated non-sudo service user** (panel review): the login user
   `doyu` has passwordless sudo (`user_config()`), so running
   `claude -p --dangerously-skip-permissions` as `doyu` would make the
   Discord-driven agent effectively root on a box shared with reconcile
   data and WireGuard keys. discopipe runs as a dedicated `discopipe`
   user instead. Trade-off accepted: the agent cannot do box admin over
   Discord.

## Architecture

```
hetznerinit (library)                boxrecipe (composition)
├── core       cloud-init compose    ├── 00_services.ipynb  service model + SERVICES
├── server     Hetzner API           ├── 01_discopipe.ipynb discopipe fragment
├── wireguard  wg fragments          └── 10_box.ipynb       box_recipe() + PEERS
└── caddy      caddy_site()                                  + rebuild flow
               (+ public support)
```

`box_recipe()` folds every SERVICES entry — install cmds, systemd units,
Caddy site blocks — plus WireGuard and the maintenance baseline into one
cloud-init YAML. Creating the VM with that user_data brings up everything
in one shot. Rough signature (generalizing the current
`hetznerinit.deploy.box_recipe`):

```python
def box_recipe(services:list[dict], peers:dict, ssh_key:str,
               hostname:str="doyu-box") -> str  # cloud-init YAML
```

### Service model (boxrecipe `00_services`)

A service is a plain dict:

```python
{"name": "reconcile", "domain": "reconcile.ninjalabo.ai",
 "port": 5001, "public": False, "packages": [...], "cmds": [...]}
```

- `domain=None` → no Caddy block (discopipe: outbound-only Discord
  websocket, no inbound HTTP, no domain needed).
- `public=False` → Caddy block keeps the `@vpn remote_ip` gate (403 for
  non-VPN sources). `public=True` → plain TLS reverse_proxy, no gate.
  Maturity path = flip the flag, rebuild.

A fragment expresses file writes (systemd unit files, CLAUDE.md, …) as
runcmd lines inside `cmds`, the same idiom the existing caddy layer uses
(`caddy_cmds()` writes the Caddyfile via runcmd). Multi-line writes keep
`caddy_cmds()`'s safeguard: raise `ValueError` if the payload contains
the heredoc delimiter, with a test asserting the guard.

Validation: `domain=None` with `public=True` raises `ValueError`
(nothing to publish). A service with a domain whose DNS record is
missing will fail ACME issuance — Caddy retries, but the site stays
unreachable until DNS lands.

### hetznerinit change (the only one)

`caddy_site()` learns to express a public site (`vpn_subnet` becomes
`str|None`; `None` → no `@vpn` matcher, direct `reverse_proxy`). TDD:
test cell asserts that no `@vpn` matcher block is emitted, that the
`respond 403` fallback is gone, and that the
`reverse_proxy 127.0.0.1:<port>` directive is still present. The new
parameter defaults to today's behavior; grep all `caddy_site()` callers
to confirm backward compatibility.

### Multi-domain routing (existing mechanism, unchanged)

Wildcard `*.ninjalabo.ai` A record → box IP. Each HTTP service binds
`127.0.0.1:<port>`; Caddy routes by hostname (SNI) and provisions TLS
per domain automatically. A domain outside ninjalabo.ai only needs its
own A record pointing at the same IP plus a SERVICES entry.

**VPN-only services need client-side split-DNS** (load-bearing caveat
from the hetznerinit caddy spec): the wildcard resolves to the public
IP, so a split-tunnel VPN client reaches Caddy with its public source
IP and the `@vpn` gate 403s it. Every `public=False` service with a
domain therefore requires an `/etc/hosts` override
(`<domain> -> 10.0.0.1`) on each VPN client — a runbook item whenever a
VPN-only SERVICES entry is added. Verification:
`curl --resolve <domain>:443:10.0.0.1 https://<domain>` must succeed
where a plain `curl https://<domain>` from outside the VPN gets 403.

## discopipe fragment (boxrecipe `01_discopipe`)

Cloud-init pieces returned by the fragment function:

- **packages**: `python3-venv` (not in the maintenance baseline; without
  it `python3 -m venv` fails on Ubuntu).
- **service user** (Decision 6):
  `useradd --create-home --shell /usr/sbin/nologin discopipe` — no sudo
  group, no SSH key; exists only to run the bot.
- **venv install**:
  `python3 -m venv /opt/discopipe && /opt/discopipe/bin/pip install
  'git+https://github.com/doyu/discopipe.git@<tag-or-SHA>'` (pinned so
  a rebuild reproduces the same bot).
- **claude CLI**: native installer run as the service user:
  `sudo -u discopipe bash -c 'curl -fsSL https://claude.ai/install.sh | bash'`
  → `/home/discopipe/.local/bin/claude`
- **secrets dir**: `mkdir -p /etc/discopipe` (root:root, mode 700) so
  the post-boot env-file install has a target.
- **agent CWD**: create `/home/discopipe/agent` (owned by discopipe) and
  write its `CLAUDE.md` (the ~1800-char reply-limit instructions, same
  content as the laptop's `~/discopipe-agent/CLAUDE.md`) via cloud-init.
- **systemd system unit** `/etc/systemd/system/discopipe.service`:
  - `After=network-online.target`, `Wants=network-online.target` (same
    network ordering as the laptop's user unit)
  - `User=discopipe`, `WorkingDirectory=/home/discopipe/agent`
  - `EnvironmentFile=/etc/discopipe/env` (root-owned 600 works: systemd
    reads it before dropping privileges)
  - `ConditionPathExists=/etc/discopipe/env` — before the secret file
    arrives the unit stays quietly inactive (no crash loop)
  - `Environment=PATH=/home/discopipe/.local/bin:/usr/local/bin:/usr/bin:/bin`
    so the unit finds `claude`
  - `ExecStart=/opt/discopipe/bin/discopipe`
  - hardening: `NoNewPrivileges=yes`, `ProtectSystem=strict`,
    `ReadWritePaths=/home/discopipe`, `PrivateTmp=yes`
  - `Restart=on-failure`, `RestartSec=5`, `StartLimitIntervalSec=60`,
    `StartLimitBurst=3`, enabled (`WantedBy=multi-user.target`)

System unit (not user unit as on the laptop): headless box, no lingering
setup needed, survives reboots unconditionally.

### Secrets

Never in user_data (instance metadata is readable from any process on the
box and from the console). The env file
(`DISCORD_TOKEN`, `DISCOPIPE_USER_ID`, `DISCOPIPE_CHANNEL_ID`,
`DISCOPIPE_CWD=/home/discopipe/agent`, `ANTHROPIC_API_KEY`) is ported
from the laptop's values (Decision 3: `DISCOPIPE_CWD` changes,
`ANTHROPIC_API_KEY` is added), copied post-boot — scp to the box, then
`sudo install -m 600 -o root -g root <tmp> /etc/discopipe/env` — then
`systemctl start discopipe`. This is a manual rebuild step of the same
rank as the GoDaddy DNS update, recorded in the hetzner-rebuild skill.

## Migration steps (high level; writing-plans will detail)

1. hetznerinit: add public support to `caddy_site()` (TDD); remove
   `nbs/10_deploy.ipynb` **and** its exported module
   `hetznerinit/deploy.py` (nbdev does not delete stale modules when a
   notebook disappears) — then `nbdev-prepare` to regenerate `_modidx`
   and confirm no remaining imports/docs reference it.
2. Scaffold boxrecipe (nbdev); depend on hetznerinit via git+https.
3. Implement `00_services`, `01_discopipe`, `10_box` (port `box_recipe`,
   PEERS, rebuild flow; generalize over SERVICES) — stub-first TDD per
   the nbdev-tdd skill.
4. Update the hetzner-rebuild skill: boxrecipe is the recipe home; add
   the env-file scp step.
5. Rebuild the box → verify DNS → install the env file → **stop the
   laptop bot first** (`systemctl --user stop/disable discopipe`; two
   gateway sessions on one token would both reply) →
   `systemctl start discopipe` on the box → E2E.

## Testing

- Literate style as today: every exported function's test cell sits right
  after its export cell.
- Recipe-level asserts on the generated YAML: discopipe unit file
  present; **no secret values in user_data**; per-service Caddy blocks
  present; `public=True` drops the `@vpn` gate; wildcard-independent
  services (domain=None) contribute no Caddy block; all service ports
  are unique.
- `check_cloud_config()` schema-validates the final YAML before any API
  call.
- E2E after rebuild: `cloud-init status`, `systemctl status discopipe`,
  one Discord round-trip message.

## Error handling

- cloud-init failures surface via the existing `check_cloud_init()` ssh
  check.
- discopipe: `Restart=on-failure`; missing env file is a wait-state
  (Condition), not a failure.
- If the box IP changes on rebuild, the GoDaddy wildcard update remains a
  manual gate before client verification (unchanged from today).

## Out of scope

- Docker/containerization (single-process services, plain systemd is
  enough).
- Automated DNS (GoDaddy's DNS API is unavailable on this account;
  manual DNS remains the gate).
- Rate limiting / cost control for `claude -p` calls (v2 if it hurts).
- Service health checks (`WatchdogSec`, external monitoring) — revisit
  when a service matters enough to page about.
- Publishing any current service (`public=True` exists but nothing flips
  yet).
- Per-service repos exporting their own deploy fragments (revisit if a
  service ever deploys to more than one box).
