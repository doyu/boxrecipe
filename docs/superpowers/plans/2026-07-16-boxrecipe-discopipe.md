# boxrecipe + discopipe-on-box Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn hetznerinit into a pure library, build the new `boxrecipe` nbdev repo that composes all doyu-box services (reconcile + the migrated discopipe bot) into one cloud-init recipe, and cut the box over.

**Architecture:** hetznerinit keeps only generic primitives (core/server/wireguard/caddy) and gains one feature: `caddy_site(vpn_subnet=None)` emits an ungated public site. boxrecipe owns the composition: a service-dict model with validation (`00_services`), the discopipe cloud-init fragment (`01_discopipe`), and `box_recipe()` + PEERS + the live rebuild flow (`10_box`). Secrets never enter user_data; the discopipe env file is installed manually post-boot.

**Tech Stack:** nbdev (notebook-first, hyphenated CLI: `nbdev-export`, `nbdev-test`, `nbdev-prepare`), Python ≥3.10, cloud-init, systemd, Caddy, WireGuard, Hetzner API via `hcloud`.

**Spec:** `docs/superpowers/specs/2026-07-15-boxrecipe-discopipe-design.md` (this repo).

## Global Constraints

- Two repos are touched. Every task states its working directory: `~/hetznerinit` or `~/boxrecipe`. The hetzner-rebuild skill lives in `~/agent-skills`.
- nbdev rules (nbdev-tdd skill): notebooks in `nbs/` are the source of truth; never hand-edit generated `.py` files. One `#| export` cell per public function, its assert test cell immediately after, `## \`funcname\`` markdown heading before each. Hyphenated CLI commands only.
- Activate the repo's `.venv` before any nbdev command: `source .venv/bin/activate`.
- Before committing notebook changes: `find nbs -name '*.ipynb' -print0 | xargs -0 python ~/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py && nbdev-prepare`.
- **Commit protocol (every commit step, even where not restated):** normalize → `nbdev-prepare` → `git add` → show `git diff --cached --stat` (full diff on request) to the user and **wait for approval** → commit. No unreviewed commits.
- **No secret values anywhere**: not in notebooks, not in user_data, not in commits. Env var *names* may appear only in test asserts that check their absence.
- Dependencies stay **unpinned** (`git+https://...` at HEAD) — design Decision 5. Do not add pins.
- hetznerinit merges via GitHub PR (no direct push to main). boxrecipe has no remote yet; commit locally to `master`.
- discopipe runs as the dedicated non-sudo user `discopipe` (design Decision 6). Never `User=doyu` in the unit.
- Task order deviates from the spec's migration list in one way: `10_deploy.ipynb` is removed (Task 6) only *after* boxrecipe's port is green (Tasks 2–5), so there is never a window without a working recipe.

---

### Task 1: hetznerinit — `caddy_site()` public support

**Working directory:** `~/hetznerinit`

**Files:**
- Modify: `nbs/03_caddy.ipynb` (the `caddy_site` export cell and its test cell)
- Generated: `hetznerinit/caddy.py` (via `nbdev-export`; do not edit by hand)

**Interfaces:**
- Produces: `caddy_site(domain:str, port:int, vpn_subnet:str|None="10.0.0.0/24") -> str`. `vpn_subnet=None` → site block with a bare `reverse_proxy 127.0.0.1:<port>`, no `@vpn` matcher, no `respond 403`. Default unchanged → existing behavior. Task 3 (`service_site`) relies on exactly this signature.

- [ ] **Step 1: Check the worktree, then branch**

```bash
cd ~/hetznerinit && git status --short
```

The worktree is currently **dirty** (`nbs/02_wireguard.ipynb`, `nbs/03_caddy.ipynb`, `nbs/10_deploy.ipynb` modified on `main`). Show the user `git diff` and resolve together — stash, commit separately, or discard — before branching. Never carry unexplained changes into the feature branch. Only then:

```bash
git checkout main && git pull && git checkout -b feat/caddy-public
```

- [ ] **Step 2: Add the failing test**

In `nbs/03_caddy.ipynb`, find the test cell directly after the `caddy_site` export cell (it starts with `block = caddy_site("reconcile.ninjalabo.ai", 5001)`). Use NotebookEdit to append to that cell:

```python

# public site: vpn_subnet=None drops the gate AND the 403 fallback, keeps the proxy
pub = caddy_site("app.example.com", 8000, vpn_subnet=None)
assert pub.splitlines()[0] == "app.example.com {"
assert "@vpn" not in pub and "remote_ip" not in pub
assert "respond 403" not in pub
assert "reverse_proxy 127.0.0.1:8000" in pub
assert pub.rstrip().endswith("}")
```

- [ ] **Step 3: Run test to verify it fails**

```bash
source .venv/bin/activate && nbdev-test --path nbs/03_caddy.ipynb
```

Expected: FAIL — current code calls `ipaddress.ip_network(None)`, raising an error other than the asserted output.

- [ ] **Step 4: Implement**

Replace the `caddy_site` function body in its `#| export` cell so the full cell reads (only `caddy_site` changes; `_domain`/`_svc_port` stay as they are):

```python
#| export
import ipaddress
import re

def _domain(value):
    # dotted hostname of DNS labels; lands in a Caddyfile site address
    if not isinstance(value, str) or not re.fullmatch(
        r"[A-Za-z0-9]([A-Za-z0-9-]{0,61}[A-Za-z0-9])?(\.[A-Za-z0-9]([A-Za-z0-9-]{0,61}[A-Za-z0-9])?)+", value):
        raise ValueError(f"domain must be a dotted hostname, got {value!r}")
    return value

def _svc_port(value):
    if isinstance(value, bool) or not isinstance(value, int) or not 1 <= value <= 65535:
        raise ValueError(f"port must be an int in 1-65535, got {value!r}")
    return value

def caddy_site(
    domain:str,                         # public subdomain, e.g. reconcile.ninjalabo.ai
    port:int,                           # local service port on 127.0.0.1
    vpn_subnet:str|None="10.0.0.0/24",  # source range allowed through; None → public site, no gate
)->str:                                 # one Caddyfile site block
    """Return a Caddyfile site block: TLS + reverse proxy, VPN-gated unless vpn_subnet is None."""
    _domain(domain)
    _svc_port(port)
    if vpn_subnet is None:
        return (
            f"{domain} {{\n"
            f"    reverse_proxy 127.0.0.1:{port}\n"
            f"}}\n"
        )
    if "%" in vpn_subnet:
        raise ValueError(f"vpn_subnet must not contain '%' (IPv6 zone-id): {vpn_subnet!r}")
    ipaddress.ip_network(vpn_subnet)  # raises ValueError on anything malformed
    return (
        f"{domain} {{\n"
        f"    @vpn remote_ip {vpn_subnet}\n"
        f"    handle @vpn {{\n"
        f"        reverse_proxy 127.0.0.1:{port}\n"
        f"    }}\n"
        f"    handle {{\n"
        f"        respond 403\n"
        f"    }}\n"
        f"}}\n"
    )
```

- [ ] **Step 5: Export and run the full notebook test**

```bash
nbdev-export && nbdev-test --path nbs/03_caddy.ipynb
```

Expected: PASS (all existing asserts — including the malformed-subnet raises — plus the new public-site asserts).

- [ ] **Step 6: Run the whole suite (callers must be unaffected)**

```bash
nbdev-test
grep -rn "caddy_site" nbs/ hetznerinit/ --include='*.py' --include='*.ipynb' -l
```

Expected: all notebooks PASS. Callers are `nbs/10_deploy.ipynb` / `hetznerinit/deploy.py` (default arg → unchanged behavior) — confirmed by the passing suite.

- [ ] **Step 7: Normalize, prepare, commit**

```bash
find nbs -name '*.ipynb' -print0 | xargs -0 python ~/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
nbdev-prepare
git add nbs/03_caddy.ipynb hetznerinit/caddy.py
git commit -m "feat: caddy_site public sites — vpn_subnet=None drops the @vpn gate"
```

**Show the user the staged diff before committing** (dev-workflow PR rules). Do not push/PR yet — Task 6 lands on the same branch.

---

### Task 2: boxrecipe — scaffold the nbdev repo

**Working directory:** `~/boxrecipe`

**Files:**
- Create: `pyproject.toml`, `README.md`, `.gitignore`, `boxrecipe/__init__.py`, `nbs/nbdev.yml`, `nbs/_quarto.yml`, `nbs/styles.css`, `nbs/index.ipynb`, `.venv/`

**Interfaces:**
- Produces: a repo where `nbdev-export` and `nbdev-test` run clean, with `hetznerinit` importable (local editable install so Task 1's unmerged change is visible).

- [ ] **Step 1: Write `pyproject.toml`**

```toml
[build-system]
requires = ["setuptools>=64"]
build-backend = "setuptools.build_meta"

[project]
name = "boxrecipe"
dynamic = ["version"]
description = "The doyu-box composition: SERVICES registry, per-service cloud-init fragments, box_recipe()"
readme = "README.md"
requires-python = ">=3.10"
license = {text = "Apache-2.0"}
authors = [{name = "Hiroshi Doyu", email = "hiroshi.doyu@ninjalabo.ai"}]
keywords = ['nbdev']
classifiers = [
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3 :: Only",
]
dependencies = [
    "hetznerinit @ git+https://github.com/doyu/hetznerinit.git",
]

[project.urls]
Repository = "https://github.com/doyu/boxrecipe"

[project.entry-points.nbdev]
boxrecipe = "boxrecipe._modidx:d"

[tool.setuptools.dynamic]
version = {attr = "boxrecipe.__version__"}

[tool.setuptools.packages.find]
include = ["boxrecipe"]

[tool.nbdev]
```

(Same shape as hetznerinit's — empty `[tool.nbdev]`, config derived from `[project]`. Dependency deliberately unpinned, design Decision 5.)

- [ ] **Step 2: Write the support files**

`boxrecipe/__init__.py`:

```python
__version__ = "0.0.1"
```

`README.md`:

```markdown
# boxrecipe

The doyu-box composition: SERVICES registry, per-service cloud-init
fragments, and `box_recipe()` — one function that folds every service
plus WireGuard and the maintenance baseline into a single cloud-init
YAML. Library primitives live in
[hetznerinit](https://github.com/doyu/hetznerinit).

Design: `docs/superpowers/specs/2026-07-15-boxrecipe-discopipe-design.md`
```

`.gitignore`:

```
_docs/
_proc/
.venv/
__pycache__/
*.py[cod]
*.egg-info/
.ipynb_checkpoints/
```

`nbs/nbdev.yml`:

```yaml
project:
  output-dir: _docs

website:
  title: "boxrecipe"
  site-url: "https://doyu.github.io/boxrecipe"
  description: "The doyu-box composition recipe"
  repo-branch: master
  repo-url: "https://github.com/doyu/boxrecipe"
```

Copy `nbs/_quarto.yml` and `nbs/styles.css` verbatim from `~/hetznerinit/nbs/`:

```bash
mkdir -p ~/boxrecipe/nbs
cp ~/hetznerinit/nbs/_quarto.yml ~/hetznerinit/nbs/styles.css ~/boxrecipe/nbs/
```

- [ ] **Step 3: Create `nbs/index.ipynb`**

Run this script (system python has nbformat):

```bash
python - <<'EOF'
import nbformat as nbf
nb = nbf.v4.new_notebook()
nb.cells = [
    nbf.v4.new_markdown_cell("# boxrecipe\n\n> The doyu-box composition: SERVICES registry + box recipe"),
    nbf.v4.new_markdown_cell("Composes every doyu-box service (Caddy sites, systemd units, install cmds) plus WireGuard and the maintenance baseline into one cloud-init YAML. See `10_box.ipynb` for the recipe and the live rebuild flow."),
]
nbf.write(nb, "/home/doyu/boxrecipe/nbs/index.ipynb")
EOF
```

- [ ] **Step 4: Create the venv and install**

```bash
cd ~/boxrecipe
python3 -m venv .venv && source .venv/bin/activate
pip install nbdev
pip install -e .
pip install -e ~/hetznerinit   # override the GitHub dep with the local checkout (Task 1's branch)
```

Expected: all installs succeed; `python -c "from hetznerinit.caddy import caddy_site; print(caddy_site('a.b.c', 80, vpn_subnet=None))"` prints a gate-free block.

- [ ] **Step 5: Verify nbdev runs clean**

```bash
nbdev-export && nbdev-test
```

Expected: exit 0 (only index.ipynb exists; nothing to test yet is fine).

- [ ] **Step 6: Normalize, prepare, commit**

```bash
find nbs -name '*.ipynb' -print0 | xargs -0 python ~/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
nbdev-prepare
git add pyproject.toml README.md .gitignore boxrecipe/ nbs/
git commit -m "chore: scaffold boxrecipe nbdev repo (hetznerinit as unpinned git dep)"
```

---

### Task 3: boxrecipe `00_services` — service model

**Working directory:** `~/boxrecipe`

**Files:**
- Create: `nbs/00_services.ipynb`
- Generated: `boxrecipe/services.py`

**Interfaces:**
- Consumes: `hetznerinit.caddy.caddy_site` (Task 1 signature).
- Produces (Tasks 4–5 import these from `boxrecipe.services`):
  - `check_service(svc:dict) -> dict` — validates, returns the dict unchanged, raises `ValueError` on bad input. Notably `domain=None` + `public=True` raises.
  - `service_site(svc:dict) -> str|None` — Caddy block, or `None` when `domain` is None. `public=True` → `caddy_site(..., vpn_subnet=None)`.
  - `write_file_cmd(path:str, content:str, owner:str="root:root", mode:str="0644") -> str` — one runcmd line: guarded heredoc write + chown + chmod. Raises `ValueError` if content contains `WF_EOF`, or if path/owner/mode fail their root-shell whitelists (absolute `[A-Za-z0-9._/-]` path, `user:group`, 3-digit octal).

- [ ] **Step 1: Create the notebook with stubs and tests**

```bash
cd ~/boxrecipe && source .venv/bin/activate
python - <<'EOF'
import nbformat as nbf
nb = nbf.v4.new_notebook()
c = nb.cells = []
md = lambda s: c.append(nbf.v4.new_markdown_cell(s))
code = lambda s: c.append(nbf.v4.new_code_cell(s))

md("# services\n\n> The service model: a plain dict per service, validated, rendered to Caddy blocks and guarded file-write cmds")
code("#| default_exp services")
code("#| hide\nfrom nbdev.showdoc import *")

md("A service is `{\"name\", \"domain\", \"port\", \"public\", \"packages\", \"cmds\"}`.\n`domain=None` → no Caddy block (outbound-only services like discopipe).\n`public=False` → `@vpn` gate; `public=True` → plain TLS reverse_proxy.\nFragments express file writes (systemd units, CLAUDE.md, ...) as runcmd\nlines via `write_file_cmd`, the same heredoc idiom as `caddy_cmds()`.")

md("## `check_service`")
code("""#| export
def check_service(
    svc:dict,  # service dict: name, domain, port, public, packages, cmds
)->dict:       # the same dict, validated
    \"\"\"Validate a service dict against the service model; return it unchanged.\"\"\"
    raise NotImplementedError""")
code("""ok = {"name": "reconcile", "domain": "reconcile.ninjalabo.ai",
      "port": 5001, "public": False, "packages": [], "cmds": []}
assert check_service(ok) is ok

nodomain = {"name": "discopipe", "domain": None, "port": None, "public": False,
            "packages": ["python3-venv"], "cmds": ["true"]}
assert check_service(nodomain) is nodomain

for bad in (
    {},                                                        # no name
    {"name": ""},                                              # empty name
    {"name": "x", "domain": None, "public": True},             # nothing to publish
    {"name": "x", "domain": "a.b.c", "port": None},            # domain needs a port
    {"name": "x", "domain": "a.b.c", "port": 70000},           # port out of range
    {"name": "x", "domain": "a.b.c", "port": 80, "public": 1}, # public not a bool
    {"name": "x", "domain": None, "packages": "notalist"},     # packages not a list
    {"name": "x", "domain": None, "cmds": [1]},                # cmds items must be str
    "not-a-dict",
):
    try:
        check_service(bad); assert False, f"must raise: {bad!r}"
    except ValueError: pass""")

md("## `service_site`")
code("""#| export
from hetznerinit.caddy import caddy_site

def service_site(
    svc:dict,     # a service dict (validated here via check_service)
)->str|None:      # Caddyfile site block, or None when the service has no domain
    \"\"\"Render a service's Caddy site block; VPN-gated unless public.\"\"\"
    raise NotImplementedError""")
code("""assert service_site({"name": "d", "domain": None, "public": False}) is None

gated = service_site({"name": "r", "domain": "reconcile.ninjalabo.ai", "port": 5001, "public": False})
assert "@vpn remote_ip 10.0.0.0/24" in gated
assert "reverse_proxy 127.0.0.1:5001" in gated and "respond 403" in gated

pub = service_site({"name": "r", "domain": "reconcile.ninjalabo.ai", "port": 5001, "public": True})
assert "@vpn" not in pub and "respond 403" not in pub
assert "reverse_proxy 127.0.0.1:5001" in pub""")

md("## `write_file_cmd`")
code("""#| export
def write_file_cmd(
    path:str,                 # absolute path to write on the box
    content:str,              # file content (must not contain the heredoc delimiter)
    owner:str="root:root",    # chown owner:group
    mode:str="0644",          # chmod mode
)->str:                       # one cloud-init runcmd line (heredoc write + chown + chmod)
    \"\"\"Return a guarded-heredoc runcmd line that writes content to path, then sets owner and mode.\"\"\"
    raise NotImplementedError""")
code("""cmd = write_file_cmd("/etc/foo.conf", "line1\\nline2\\n", owner="discopipe:discopipe", mode="0600")
assert cmd.startswith("cat > /etc/foo.conf <<'WF_EOF'\\n")
assert "line1\\nline2\\nWF_EOF" in cmd
assert "chown discopipe:discopipe /etc/foo.conf" in cmd
assert "chmod 0600 /etc/foo.conf" in cmd

# same guard as caddy_cmds: delimiter in the payload must raise
try:
    write_file_cmd("/etc/foo", "evil\\nWF_EOF\\nrm -rf /"); assert False, "guard must raise"
except ValueError: pass

# root-shell inputs are whitelisted, not quoted (same rule as wg_server_cmds)
for bad_path in ("etc/foo", "/etc/foo; rm -rf /", "/etc/$(reboot)", "/etc/a b", ""):
    try:
        write_file_cmd(bad_path, "x"); assert False, f"bad path must raise: {bad_path!r}"
    except ValueError: pass
for bad_owner in ("root", "root:root; reboot", "root root", ""):
    try:
        write_file_cmd("/etc/foo", "x", owner=bad_owner); assert False, f"bad owner must raise: {bad_owner!r}"
    except ValueError: pass
for bad_mode in ("777x", "9", "rwxr-xr-x", "0 644"):
    try:
        write_file_cmd("/etc/foo", "x", mode=bad_mode); assert False, f"bad mode must raise: {bad_mode!r}"
    except ValueError: pass""")

code("#| hide\nimport nbdev; nbdev.nbdev_export()")
nbf.write(nb, "nbs/00_services.ipynb")
EOF
```

- [ ] **Step 2: Export, run tests, verify they fail**

```bash
nbdev-export && nbdev-test --path nbs/00_services.ipynb
```

Expected: FAIL with `NotImplementedError`.

- [ ] **Step 3: Implement `check_service`**

Replace the `check_service` export cell body (NotebookEdit) with:

```python
#| export
def check_service(
    svc:dict,  # service dict: name, domain, port, public, packages, cmds
)->dict:       # the same dict, validated
    """Validate a service dict against the service model; return it unchanged."""
    if not isinstance(svc, dict):
        raise ValueError(f"service must be a dict, got {type(svc).__name__}")
    name = svc.get("name")
    if not isinstance(name, str) or not name.strip():
        raise ValueError(f"service name must be a non-empty string, got {name!r}")
    domain, port = svc.get("domain"), svc.get("port")
    public = svc.get("public", False)
    if not isinstance(public, bool):
        raise ValueError(f"{name}: public must be a bool, got {public!r}")
    if domain is None:
        if public:
            raise ValueError(f"{name}: domain=None with public=True (nothing to publish)")
    else:
        if isinstance(port, bool) or not isinstance(port, int) or not 1 <= port <= 65535:
            raise ValueError(f"{name}: a service with a domain needs a port in 1-65535, got {port!r}")
    for key in ("packages", "cmds"):
        val = svc.get(key, [])
        if not isinstance(val, list) or not all(isinstance(x, str) for x in val):
            raise ValueError(f"{name}: {key} must be a list of str, got {val!r}")
    return svc
```

(Domain *format* is validated downstream by `caddy_site` — DRY.)

- [ ] **Step 4: Implement `service_site`**

```python
#| export
from hetznerinit.caddy import caddy_site

def service_site(
    svc:dict,     # a service dict (validated here via check_service)
)->str|None:      # Caddyfile site block, or None when the service has no domain
    """Render a service's Caddy site block; VPN-gated unless public."""
    svc = check_service(svc)
    if svc.get("domain") is None:
        return None
    if svc.get("public", False):
        return caddy_site(svc["domain"], svc["port"], vpn_subnet=None)
    return caddy_site(svc["domain"], svc["port"])
```

- [ ] **Step 5: Implement `write_file_cmd`**

```python
#| export
import re

def write_file_cmd(
    path:str,                 # absolute path to write on the box
    content:str,              # file content (must not contain the heredoc delimiter)
    owner:str="root:root",    # chown owner:group
    mode:str="0644",          # chmod mode
)->str:                       # one cloud-init runcmd line (heredoc write + chown + chmod)
    """Return a guarded-heredoc runcmd line that writes content to path, then sets owner and mode."""
    # every value below lands in a root shell: whitelist instead of quoting
    # (same rule as hetznerinit's wg_server_cmds)
    if not isinstance(path, str) or not re.fullmatch(r"/[A-Za-z0-9._/-]+", path):
        raise ValueError(f"path must be absolute using [A-Za-z0-9._/-], got {path!r}")
    if not isinstance(owner, str) or not re.fullmatch(r"[a-z_][a-z0-9_-]*:[a-z_][a-z0-9_-]*", owner):
        raise ValueError(f"owner must be user:group, got {owner!r}")
    if not isinstance(mode, str) or not re.fullmatch(r"0?[0-7]{3}", mode):
        raise ValueError(f"mode must be 3 octal digits (optional 0 prefix), got {mode!r}")
    if "WF_EOF" in content:
        raise ValueError("content must not contain the heredoc delimiter 'WF_EOF'")
    return (f"cat > {path} <<'WF_EOF'\n{content.rstrip(chr(10))}\nWF_EOF\n"
            f"chown {owner} {path}\nchmod {mode} {path}")
```

- [ ] **Step 6: Export and run tests to verify they pass**

```bash
nbdev-export && nbdev-test --path nbs/00_services.ipynb
```

Expected: PASS.

- [ ] **Step 7: Normalize, prepare, commit**

```bash
find nbs -name '*.ipynb' -print0 | xargs -0 python ~/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
nbdev-prepare
git add nbs/00_services.ipynb boxrecipe/services.py boxrecipe/_modidx.py
git commit -m "feat: service model — check_service, service_site, write_file_cmd"
```

---

### Task 4: boxrecipe `01_discopipe` — the discopipe fragment

**Working directory:** `~/boxrecipe`

**Files:**
- Create: `nbs/01_discopipe.ipynb`
- Generated: `boxrecipe/discopipe.py`

**Interfaces:**
- Consumes: `check_service`, `write_file_cmd` from `boxrecipe.services` (Task 3).
- Produces: `discopipe_service() -> dict` — a validated service dict (`domain=None`, `port=None`, `public=False`, `packages=["python3-venv"]`, `cmds=[...]`). Task 5 puts it in `SERVICES`.

- [ ] **Step 1: Create the notebook with stub and tests**

```bash
cd ~/boxrecipe && source .venv/bin/activate
python - <<'EOF'
import nbformat as nbf
nb = nbf.v4.new_notebook()
c = nb.cells = []
md = lambda s: c.append(nbf.v4.new_markdown_cell(s))
code = lambda s: c.append(nbf.v4.new_code_cell(s))

md("# discopipe\n\n> Cloud-init fragment for the discopipe bot: dedicated non-sudo user, venv install, claude CLI, agent CWD, hardened systemd unit")
code("#| default_exp discopipe")
code("#| hide\nfrom nbdev.showdoc import *")

md("Runs as the dedicated `discopipe` user (design Decision 6): the login\nuser has passwordless sudo, and the bot drives\n`claude -p --dangerously-skip-permissions`. Secrets live in\n`/etc/discopipe/env` (installed manually post-boot, never in user_data);\nuntil it exists the unit stays inactive via `ConditionPathExists`.")

md("## `discopipe_service`")
code("""#| export
from boxrecipe.services import check_service, write_file_cmd

def discopipe_service(
)->dict:  # validated service dict for the discopipe bot (no domain, install+unit cmds)
    \"\"\"Service dict for discopipe: outbound-only Discord bot, no Caddy block.\"\"\"
    raise NotImplementedError""")
code("""svc = discopipe_service()
assert svc["name"] == "discopipe" and svc["domain"] is None and svc["public"] is False
assert "python3-venv" in svc["packages"]

joined = "\\n".join(svc["cmds"])
# user exists before anything installs or writes into its home
assert "useradd --create-home --shell /usr/sbin/nologin discopipe" in joined
assert joined.index("useradd") < joined.index("claude.ai/install.sh")
# installs: bot venv (unpinned, Decision 5) + claude CLI as the service user
assert "python3 -m venv /opt/discopipe" in joined
assert "/opt/discopipe/bin/pip install git+https://github.com/doyu/discopipe.git" in joined
assert "sudo -H -u discopipe bash -c 'curl -fsSL https://claude.ai/install.sh | bash'" in joined
# secrets dir (root-only) and agent CWD
assert "install -d -m 700 -o root -g root /etc/discopipe" in joined
assert "install -d -o discopipe -g discopipe /home/discopipe/agent" in joined
# agent instructions landed
assert "/home/discopipe/agent/CLAUDE.md" in joined and "1800" in joined
# the unit: identity, wait-state, network ordering, hardening, restart policy
for token in ("/etc/systemd/system/discopipe.service",
              "User=discopipe",
              "WorkingDirectory=/home/discopipe/agent",
              "EnvironmentFile=/etc/discopipe/env",
              "ConditionPathExists=/etc/discopipe/env",
              "After=network-online.target", "Wants=network-online.target",
              "Environment=PATH=/home/discopipe/.local/bin:/usr/local/bin:/usr/bin:/bin",
              "ExecStart=/opt/discopipe/bin/discopipe",
              "Restart=on-failure", "RestartSec=5",
              "StartLimitIntervalSec=60", "StartLimitBurst=3",
              "NoNewPrivileges=yes", "ProtectSystem=strict",
              "ReadWritePaths=/home/discopipe", "PrivateTmp=yes",
              "WantedBy=multi-user.target"):
    assert token in joined, token
assert "systemctl daemon-reload" in joined
assert "systemctl enable discopipe" in joined
# never a secret value or env-var assignment in the fragment
for name in ("DISCORD_TOKEN", "DISCOPIPE_USER_ID", "DISCOPIPE_CHANNEL_ID", "ANTHROPIC_API_KEY"):
    assert name not in joined, name""")

code("#| hide\nimport nbdev; nbdev.nbdev_export()")
nbf.write(nb, "nbs/01_discopipe.ipynb")
EOF
```

- [ ] **Step 2: Export, run tests, verify they fail**

```bash
nbdev-export && nbdev-test --path nbs/01_discopipe.ipynb
```

Expected: FAIL with `NotImplementedError`.

- [ ] **Step 3: Implement**

Replace the `discopipe_service` export cell (NotebookEdit) with:

```python
#| export
from boxrecipe.services import check_service, write_file_cmd

_UNIT = """[Unit]
Description=discopipe - Discord passthrough to a headless coding agent CLI
After=network-online.target
Wants=network-online.target
ConditionPathExists=/etc/discopipe/env
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
User=discopipe
WorkingDirectory=/home/discopipe/agent
EnvironmentFile=/etc/discopipe/env
Environment=PATH=/home/discopipe/.local/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=/opt/discopipe/bin/discopipe
Restart=on-failure
RestartSec=5
NoNewPrivileges=yes
ProtectSystem=strict
ReadWritePaths=/home/discopipe
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
"""

_CLAUDE_MD = """Replies are read on a phone via Discord. Keep every reply under 1800
characters. Be terse. For long output, write it to a file and reply with
the path and a 3-line summary.
"""

def discopipe_service(
)->dict:  # validated service dict for the discopipe bot (no domain, install+unit cmds)
    """Service dict for discopipe: outbound-only Discord bot, no Caddy block."""
    cmds = [
        "useradd --create-home --shell /usr/sbin/nologin discopipe",
        "python3 -m venv /opt/discopipe",
        "/opt/discopipe/bin/pip install git+https://github.com/doyu/discopipe.git",
        "sudo -H -u discopipe bash -c 'curl -fsSL https://claude.ai/install.sh | bash'",
        "install -d -m 700 -o root -g root /etc/discopipe",
        "install -d -o discopipe -g discopipe /home/discopipe/agent",
        write_file_cmd("/home/discopipe/agent/CLAUDE.md", _CLAUDE_MD,
                       owner="discopipe:discopipe"),
        write_file_cmd("/etc/systemd/system/discopipe.service", _UNIT),
        "systemctl daemon-reload",
        "systemctl enable discopipe",
    ]
    return check_service({"name": "discopipe", "domain": None, "port": None,
                          "public": False, "packages": ["python3-venv"], "cmds": cmds})
```

- [ ] **Step 4: Export and run tests to verify they pass**

```bash
nbdev-export && nbdev-test --path nbs/01_discopipe.ipynb
```

Expected: PASS.

- [ ] **Step 5: Normalize, prepare, commit**

```bash
find nbs -name '*.ipynb' -print0 | xargs -0 python ~/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
nbdev-prepare
git add nbs/01_discopipe.ipynb boxrecipe/discopipe.py boxrecipe/_modidx.py
git commit -m "feat: discopipe cloud-init fragment — dedicated user, hardened unit, no secrets"
```

---

### Task 5: boxrecipe `10_box` — `box_recipe()`, SERVICES, PEERS, rebuild flow

**Working directory:** `~/boxrecipe`

**Files:**
- Create: `nbs/10_box.ipynb`
- Generated: `boxrecipe/box.py`

**Interfaces:**
- Consumes: `check_service`, `service_site` (Task 3); `discopipe_service` (Task 4); from hetznerinit: `cloud_init_dict`, `cloud_init_yaml` (core), `wg_server_cmds` (wireguard), `caddyfile`, `caddy_cmds` (caddy), and in live cells `get_client`/`server_spec`/`check_cloud_config`/`create_server`/`delete_server`/`wait_running`/`check_cloud_init` (server), `get_server_pubkey`/`wg_keypair`/`wg_client_conf` (wireguard).
- Produces: `box_recipe(services:list, peers:dict, ssh_key:str, hostname:str="doyu-box") -> str` (full cloud-init YAML); `RECONCILE:dict`; `SERVICES:list`.

- [ ] **Step 1: Create the notebook with stub, tests, registry, and the ported live flow**

```bash
cd ~/boxrecipe && source .venv/bin/activate
python - <<'EOF'
import nbformat as nbf
nb = nbf.v4.new_notebook()
c = nb.cells = []
md = lambda s: c.append(nbf.v4.new_markdown_cell(s))
code = lambda s: c.append(nbf.v4.new_code_cell(s))

md("# box\n\n> box_recipe() folds every SERVICES entry plus WireGuard and the maintenance baseline into one cloud-init YAML — plus the live rebuild flow")
code("#| default_exp box")
code("#| hide\nfrom nbdev.showdoc import *")

md("## `box_recipe`")
code("""#| export
from hetznerinit.core import cloud_init_dict, cloud_init_yaml
from hetznerinit.wireguard import wg_server_cmds
from hetznerinit.caddy import caddyfile, caddy_cmds
from boxrecipe.services import check_service, service_site
from boxrecipe.discopipe import discopipe_service

def box_recipe(
    services:list,             # service dicts (see boxrecipe.services)
    peers:dict,                # {"e15": {"ip": "10.0.0.2", "pubkey": "..."}}
    ssh_key:str,               # SSH public key line for the login user
    hostname:str="doyu-box",   # cloud-init hostname
)->str:                        # full cloud-init YAML for the box
    \"\"\"Compose the box's cloud-init: every service + WireGuard + a maintenance baseline.\"\"\"
    raise NotImplementedError""")
code("""DEMO_PEERS = {"e15": {"ip": "10.0.0.2", "pubkey": "A" * 43 + "="}}
DEMO_KEY = "ssh-ed25519 AAAADEMO doyu@example"
_RECONCILE = {"name": "reconcile", "domain": "reconcile.ninjalabo.ai",
              "port": 5001, "public": False, "packages": [], "cmds": []}

y = box_recipe([_RECONCILE, discopipe_service()], DEMO_PEERS, DEMO_KEY)
assert y.startswith("#cloud-config\\n")
# maintenance baseline + wireguard (unchanged from the old hetznerinit deploy)
assert "- byobu" in y and "- wireguard" in y and "- ufw" in y
assert "ufw allow 51820/udp" in y
assert "A" * 43 + "=" in y
# per-service pieces folded in
assert "reconcile.ninjalabo.ai {" in y
assert "@vpn remote_ip 10.0.0.0/24" in y
assert "systemctl reload caddy" in y
assert "- python3-venv" in y
assert "/etc/systemd/system/discopipe.service" in y
assert "User=discopipe" in y and "ProtectSystem=strict" in y
assert "systemctl enable discopipe" in y
# no secret names/values ever reach user_data
for name in ("DISCORD_TOKEN", "DISCOPIPE_USER_ID", "DISCOPIPE_CHANNEL_ID", "ANTHROPIC_API_KEY"):
    assert name not in y, name

# public=True drops the gate and the 403 fallback for that site
y_pub = box_recipe([dict(_RECONCILE, public=True)], DEMO_PEERS, DEMO_KEY)
assert "@vpn" not in y_pub and "respond 403" not in y_pub
assert "reverse_proxy 127.0.0.1:5001" in y_pub

# domain=None services contribute no Caddy site; no sites → Caddy not even installed
y_d = box_recipe([discopipe_service()], DEMO_PEERS, DEMO_KEY)
assert "reverse_proxy" not in y_d
assert "apt-get install -y caddy" not in y_d

# port collisions are rejected
try:
    box_recipe([_RECONCILE, dict(_RECONCILE, name="other")], DEMO_PEERS, DEMO_KEY)
    assert False, "duplicate ports must raise"
except ValueError: pass

# ssh_key validation (unchanged from the old deploy)
for bad in ("", "   ", None):
    try:
        box_recipe([_RECONCILE], DEMO_PEERS, bad); assert False, f"must raise: {bad!r}"
    except ValueError: pass""")

md("## SERVICES\\n\\nThe registry: everything that runs on doyu-box. Adding a service = one\\nentry here (plus its fragment notebook if it needs install cmds).\\nMaturity path: flip `public` to True, rebuild.")
code("""#| export
RECONCILE = {"name": "reconcile", "domain": "reconcile.ninjalabo.ai",
             "port": 5001, "public": False, "packages": [], "cmds": []}

SERVICES = [RECONCILE, discopipe_service()]""")
code("""assert [s["name"] for s in SERVICES] == ["reconcile", "discopipe"]
for s in SERVICES:
    check_service(s)""")

md("## Peers\\n\\nCommitted device list (public keys only — config, not secret). Kept out of the\\npublished docs with `#| hide`. Seed the real pubkeys once from the running box:\\n`ssh doyu@box.ninjalabo.ai sudo cat /etc/wireguard/wg0.conf` (read each [Peer]'s\\n`# name` + AllowedIPs so e15↔10.0.0.2 / phone↔10.0.0.3 is not transposed).")
code("""#| hide
PEERS = {
    "e15":   {"ip": "10.0.0.2", "pubkey": "awFS9HoxtslvUcFQ3fNOTPFUFl3F9N4Z9Jq+59xmbWM="},   # laptop
    "phone": {"ip": "10.0.0.3", "pubkey": "aoj3tHT4it+cHmZhhIyY6n20qp9wjNsilryS7dy701M="},   # phone
}""")

md("## Dry run\\n\\nBuild the full recipe without touching the API.")
code("""DEMO = {"e15": {"ip": "10.0.0.2", "pubkey": "A" * 43 + "="}}
print(box_recipe(SERVICES, DEMO, "ssh-ed25519 AAAADEMO doyu@example"))""")

md("## Live run: rebuild the box\\n\\nAll cells below are `#| eval: false`. Run in order. This DESTROYS and recreates\\ndoyu-box. Fill `PEERS` with real pubkeys first. If the new IP differs, update the\\nGoDaddy wildcard `*` A record (see the hetzner-rebuild skill) before the client\\nsteps.")
code("""#| eval: false
import os
from pathlib import Path
from hetznerinit.server import (get_client, server_spec, check_cloud_config,
                                create_server, delete_server, wait_running, check_cloud_init)
from hetznerinit.wireguard import get_server_pubkey

assert all(p["pubkey"] != "..." for p in PEERS.values()), "fill PEERS first"

def _ssh_pubkey():
    env = os.environ.get("SSH_PUBKEY_PATH")
    cands = [Path(env)] if env else [Path.home()/".ssh"/n for n in ("id_ed25519.pub", "id_rsa.pub")]
    for p in cands:
        if p.exists(): return p.read_text().strip()
    raise FileNotFoundError(f"no SSH public key found (tried {[str(c) for c in cands]}); set SSH_PUBKEY_PATH")

ssh_key = _ssh_pubkey()
user_data = box_recipe(SERVICES, PEERS, ssh_key)
print(check_cloud_config(user_data))
print(user_data)  # inspect the REAL user_data before the create cell""")
code("""#| eval: false
client = get_client()
old = client.servers.get_by_name("doyu-box")
if old is not None:
    delete_server(old, confirm=True)
    print(f"deleted old box id={old.id}")
srv = create_server(client, server_spec("doyu-box", user_data), confirm=True)
print(f"created: id={srv.id} name={srv.name} ip={srv.public_net.ipv4.ip}")
print("cleanup if anything below fails:")
print("  srv = client.servers.get_by_name('doyu-box'); delete_server(srv, confirm=True)")""")
code("""#| eval: false
import subprocess as _sp
ip = wait_running(srv)
for target in (ip, "box.ninjalabo.ai"):
    _sp.run(["ssh-keygen", "-R", target], capture_output=True)
dns_ip = _sp.run(["dig", "+short", "box.ninjalabo.ai"], capture_output=True, text=True).stdout.strip()
print(f"box ip: {ip}  /  DNS says: {dns_ip}")
if dns_ip != ip:
    print(f"UPDATE GoDaddy: set the wildcard '*' A record to {ip}, wait for TTL, then continue")""")
code("""#| eval: false
print(check_cloud_init(ip, "doyu"))""")
code("""#| eval: false
# verify Caddy and the discopipe unit (waiting on its env file is expected here)
import subprocess as _sp
def _ssh(*remote):
    return _sp.run(["ssh", "-o", "BatchMode=yes", "-o", "StrictHostKeyChecking=accept-new",
                    "doyu@" + ip, *remote], capture_output=True, text=True).stdout.strip()
print("caddy active:", _ssh("systemctl", "is-active", "caddy"))
print("config valid:", _ssh("sudo", "caddy", "validate", "--config", "/etc/caddy/Caddyfile"))
print("discopipe enabled:", _ssh("systemctl", "is-enabled", "discopipe"))
print("discopipe active (inactive until env file lands):", _ssh("systemctl", "is-active", "discopipe"))""")

md("## Install the discopipe env file (manual, secrets)\\n\\nSame rank as the GoDaddy DNS step: user_data never carries secrets, so the\\nenv file is installed by hand after boot. **Port** the laptop's values —\\n`DISCOPIPE_CWD` changes to `/home/discopipe/agent`, `ANTHROPIC_API_KEY` is\\nadded (design Decision 3). Run on the laptop:\\n\\n```sh\\numask 077\\ncp ~/.config/discopipe/env /tmp/box-env\\nsed -i 's|^DISCOPIPE_CWD=.*|DISCOPIPE_CWD=/home/discopipe/agent|' /tmp/box-env\\nread -rs ANTHROPIC_API_KEY   # paste the key: no echo, never in shell history\\nprintf 'ANTHROPIC_API_KEY=%s\\\\n' \\"$ANTHROPIC_API_KEY\\" >> /tmp/box-env\\nunset ANTHROPIC_API_KEY\\nscp /tmp/box-env doyu@<IP>:/tmp/env && rm /tmp/box-env\\nssh doyu@<IP> 'sudo install -m 600 -o root -g root /tmp/env /etc/discopipe/env && rm /tmp/env \\\\\\n  && sudo systemctl start discopipe && systemctl is-active discopipe'\\n```\\n\\nThen send one Discord message in the bot's channel and confirm a reply\\n(the laptop bot is already stopped — `systemctl --user is-enabled discopipe`\\nmust report disabled).")

code("""#| eval: false
# server key is regenerated on every rebuild → refresh the SERVER pubkey in each client
spub = get_server_pubkey(ip, "doyu")
print(f"new server pubkey: {spub}")
print("laptop: update /etc/wireguard/box.conf then restart the tunnel:")
print(f"  sudo sed -i 's|PublicKey = .*|PublicKey = {spub}|' /etc/wireguard/box.conf")
print("  sudo wg-quick down box && sudo wg-quick up box")
print("phone: in the WireGuard app, set the peer's Public key to the value above")""")
code("""#| eval: false
# after the laptop tunnel is back up
import subprocess as _sp
print(_sp.run(["ping", "-c", "3", "10.0.0.1"], capture_output=True, text=True).stdout)
print(_sp.run(["ssh", "-o", "BatchMode=yes", "-o", "StrictHostKeyChecking=accept-new",
               "doyu@10.0.0.1", "hostname"], capture_output=True, text=True).stdout)""")

md("## Add a new WireGuard client (only when needed)\\n\\nThe ONLY place a client keypair is generated. Off the normal rebuild path.")
code("""#| eval: false
from hetznerinit.wireguard import wg_keypair, wg_client_conf, get_server_pubkey
import subprocess as _sp

name, ip_addr = "newdev", "10.0.0.4"          # pick an unused name + 10.0.0.x
spub = get_server_pubkey("box.ninjalabo.ai", "doyu")
priv, pub = wg_keypair()
print(f'add to PEERS, commit, then rebuild:  "{name}": {{"ip": "{ip_addr}", "pubkey": "{pub}"}}')
conf = wg_client_conf(name, priv, spub, ip_addr)
# the QR encodes this client's PRIVATE key — scan it on the phone, then CLEAR
# this cell's output before saving/committing (do not leave it in the notebook)
_sp.run(["qrencode", "-t", "ansiutf8"], input=conf, text=True)""")

md("## Client access (per VPN device)\\n\\nEvery `public=False` service with a domain needs a client-side split-DNS\\noverride: an `/etc/hosts` line `10.0.0.1 <domain>` on each VPN device,\\nor Caddy's `remote_ip` gate returns 403 (traffic to the public IP\\nbypasses the tunnel). Verify per domain:\\n`curl --resolve <domain>:443:10.0.0.1 https://<domain>`.")

code("#| hide\nimport nbdev; nbdev.nbdev_export()")
nbf.write(nb, "nbs/10_box.ipynb")
EOF
```

- [ ] **Step 2: Export, run tests, verify they fail**

```bash
nbdev-export && nbdev-test --path nbs/10_box.ipynb
```

Expected: FAIL with `NotImplementedError` (the `#| eval: false` live cells are skipped).

- [ ] **Step 3: Implement `box_recipe`**

Replace the `box_recipe` export cell (NotebookEdit) with:

```python
#| export
from hetznerinit.core import cloud_init_dict, cloud_init_yaml
from hetznerinit.wireguard import wg_server_cmds
from hetznerinit.caddy import caddyfile, caddy_cmds
from boxrecipe.services import check_service, service_site
from boxrecipe.discopipe import discopipe_service

def box_recipe(
    services:list,             # service dicts (see boxrecipe.services)
    peers:dict,                # {"e15": {"ip": "10.0.0.2", "pubkey": "..."}}
    ssh_key:str,               # SSH public key line for the login user
    hostname:str="doyu-box",   # cloud-init hostname
)->str:                        # full cloud-init YAML for the box
    """Compose the box's cloud-init: every service + WireGuard + a maintenance baseline."""
    if not isinstance(ssh_key, str) or not ssh_key.strip():
        raise ValueError("ssh_key must be a non-empty string")
    services = [check_service(s) for s in services]
    ports = [s["port"] for s in services if s.get("port") is not None]
    if len(ports) != len(set(ports)):
        raise ValueError(f"service ports must be unique, got {sorted(ports)}")
    sites = [b for b in (service_site(s) for s in services) if b]
    caddy = caddy_cmds(caddyfile(sites)) if sites else []  # no sites → skip Caddy entirely
    packages = ["wireguard", "byobu", "curl", "git",
                "debian-keyring", "debian-archive-keyring", "apt-transport-https"]
    for s in services:
        for p in s.get("packages", []):
            if p not in packages:
                packages.append(p)
    svc_cmds = [c for s in services for c in s.get("cmds", [])]
    return cloud_init_yaml(cloud_init_dict(
        hostname, "doyu", ssh_key,
        packages=packages,
        cmds=[*wg_server_cmds(peers), *caddy, *svc_cmds],
        udp_ports=51820,
    ))
```

- [ ] **Step 4: Export and run tests to verify they pass**

```bash
nbdev-export && nbdev-test --path nbs/10_box.ipynb
```

Expected: PASS.

- [ ] **Step 5: Run the whole boxrecipe suite**

```bash
nbdev-test
```

Expected: all notebooks PASS.

- [ ] **Step 6: Normalize, prepare, commit**

```bash
find nbs -name '*.ipynb' -print0 | xargs -0 python ~/.claude/skills/nbdev-tdd/scripts/normalize_notebooks.py
nbdev-prepare
git add nbs/10_box.ipynb boxrecipe/box.py boxrecipe/_modidx.py
git commit -m "feat: box_recipe over SERVICES + PEERS + live rebuild flow (ported from hetznerinit)"
```

---

### Task 6: hetznerinit — remove the deploy layer

**Working directory:** `~/hetznerinit` (branch `feat/caddy-public` from Task 1)

**Files:**
- Delete: `nbs/10_deploy.ipynb`, `hetznerinit/deploy.py` (nbdev does not remove stale modules when a notebook disappears)
- Regenerated: `hetznerinit/_modidx.py`

**Interfaces:**
- Consumes: nothing. Precondition: Task 5 committed (boxrecipe's port is green), so no working-recipe gap.

- [ ] **Step 1: Remove notebook and stale module**

```bash
cd ~/hetznerinit && source .venv/bin/activate
git rm nbs/10_deploy.ipynb hetznerinit/deploy.py
```

- [ ] **Step 2: Regenerate and verify nothing references deploy**

```bash
nbdev-export && nbdev-test
git grep -n "deploy" -- hetznerinit nbs README.md llms.txt || echo CLEAN   # tracked files only (skips .ipynb_checkpoints)
python -c "import hetznerinit._modidx as m; assert not any('deploy' in k for k in m.d['syms']); print('modidx clean')"
```

Expected: tests PASS, `CLEAN`, `modidx clean`. If `llms.txt` matches, update its module list to drop deploy.

- [ ] **Step 3: Prepare and commit**

```bash
nbdev-prepare
git add -A
git commit -m "refactor!: drop the deploy layer — box composition moves to the boxrecipe repo"
```

- [ ] **Step 4: PR**

Push `feat/caddy-public` and open a PR (both commits: public caddy support + deploy removal; they are one coherent "become a pure library" change, ~small diff). **Ask the user to review/merge** — no direct push to main. After merge, refresh boxrecipe's dep:

```bash
cd ~/boxrecipe && source .venv/bin/activate
pip install --force-reinstall --no-deps "hetznerinit @ git+https://github.com/doyu/hetznerinit.git"
nbdev-test
```

Expected: PASS against the published library (no longer relying on the local editable override).

---

### Task 7: update the hetzner-rebuild skill

**Working directory:** `~/agent-skills`

**Files:**
- Modify: `skills/hetzner-rebuild/SKILL.md`

**Interfaces:**
- Consumes: the boxrecipe live flow (Task 5 cell layout).

- [ ] **Step 1: Point the skill at boxrecipe**

In the frontmatter `description:`, replace `from an updated hetznerinit recipe` with `from an updated boxrecipe recipe (composition of hetznerinit primitives)`.

In the intro paragraph, replace:

```
Rebuild the Hetzner box from the current hetznerinit recipe and make DNS +
access work again.
```

with:

```
Rebuild the Hetzner box from the current boxrecipe recipe
(~/boxrecipe/nbs/10_box.ipynb — the SERVICES composition; hetznerinit is
just the library underneath) and make DNS + access work again.
```

In Procedure step 1, replace `The relevant `nbs/*.ipynb` is committed` with `The relevant `~/boxrecipe/nbs/*.ipynb` is committed`.

- [ ] **Step 2: Extend step 6's SSH verification**

In the step-6 ssh block, after the `systemctl is-active caddy` line, add:

```
     systemctl is-enabled discopipe;          # enabled (inactive until env file lands)
```

- [ ] **Step 3: Add the env-file step**

Insert a new step between steps 6 and 7 (renumber the rest):

```markdown
7. **Install the discopipe env file (user does this — it holds secrets).**
   user_data never carries secrets; until this file exists the unit waits
   (`ConditionPathExists`). Values are *ported* from the laptop's
   `~/.config/discopipe/env`: `DISCOPIPE_CWD` becomes
   `/home/discopipe/agent`, `ANTHROPIC_API_KEY` is added. Point the user
   at the "Install the discopipe env file" section of
   `~/boxrecipe/nbs/10_box.ipynb`, then verify:
   ```
   ssh doyu@<IP> 'systemctl is-active discopipe; journalctl -u discopipe -n 20 --no-pager'
   ```
   and ask the user to send one Discord message and confirm the reply.
```

- [ ] **Step 4: Add the split-DNS reminder**

In the (now) step-8 VPN-client step, append:

```
   Every `public=False` service with a domain also needs its `/etc/hosts`
   split-DNS line on each VPN client (`10.0.0.1 <domain>`) — verify with
   `curl --resolve <domain>:443:10.0.0.1 https://<domain>`.
```

- [ ] **Step 5: Show the diff to the user**

`git -C ~/agent-skills diff` — per skill-authoring rules, the user decides whether/when to commit `~/agent-skills`.

---

### Task 8: rebuild the box and cut over (E2E)

**Working directory:** `~/boxrecipe` (+ the user's Jupyter)

This task follows the updated hetzner-rebuild skill. Manual gates are marked **[user]** — they involve `HCLOUD_TOKEN`, GoDaddy login, or secret values; the agent never touches those.

- [ ] **Step 1: Preconditions**

- Task 6's PR is merged and boxrecipe tests pass against published hetznerinit.
- `PEERS` in `nbs/10_box.ipynb` holds real pubkeys (they were ported in Task 5).
- Laptop bot is stopped and cannot come back — a manually started instance would double-reply on the shared token, and `is-enabled` alone doesn't catch that:

```bash
systemctl --user disable --now discopipe 2>/dev/null || true
systemctl --user is-active discopipe    # expect: inactive (or unknown)
pgrep -af discopipe || echo "no stray instance"
```

- [ ] **Step 2: [user] Dry run + create**

User runs, in Jupyter with the boxrecipe repo: the dry-run cell, then the **first live cell** — it validates (`check_cloud_config`) and prints the *real* `user_data` (PEERS + real SSH key). Inspect that YAML before running the create cell. User pastes the `created: ... ip=<IP>` line.

- [ ] **Step 3: DNS gate**

Follow skill steps 3–5 (dig against `@ns01.domaincontrol.com`, GoDaddy wildcard update if the IP changed, wait for propagation).

- [ ] **Step 4: Verify the box**

```bash
ssh-keygen -R <IP>; ssh-keygen -R box.ninjalabo.ai
ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new doyu@<IP> '
  cloud-init status;
  sudo ufw status | head;
  sudo wg show 2>/dev/null | head;
  systemctl is-active caddy;
  systemctl is-enabled discopipe;
  systemctl is-active discopipe'
```

Expected: cloud-init `done` (rc 2 "degraded done" OK on Hetzner), ufw active, wg up, caddy `active`, discopipe `enabled` but `inactive` (env file not yet installed — this is the designed wait-state).

Also verify the claude binary landed where the unit's `Environment=PATH` expects (the installer's target is not guaranteed):

```bash
ssh doyu@<IP> "sudo -H -u discopipe bash -lc 'command -v claude'"
```

Expected: `/home/discopipe/.local/bin/claude`. If it resolves elsewhere (e.g. `~/.claude/bin/claude`), fix the unit's `Environment=PATH` in `01_discopipe` (and rebuild or patch the unit on-box) **before** starting the bot.

- [ ] **Step 5: [user] Install the env file and start**

User follows the "Install the discopipe env file" section of `nbs/10_box.ipynb` (port values, scp, `sudo install -m 600 -o root -g root`, `systemctl start discopipe`).

- [ ] **Step 6: Verify discopipe**

```bash
ssh doyu@<IP> 'systemctl is-active discopipe; journalctl -u discopipe -n 40 --no-pager'
```

Expected: `active`; journal shows the Discord gateway connect, no restart loop.

- [ ] **Step 7: [user] Discord round-trip**

User sends one message in the bot's channel; a single reply arrives (no duplicates — the laptop bot is off).

- [ ] **Step 8: VPN clients**

Skill steps: refresh server pubkey in laptop/phone WireGuard configs, restart tunnels, then verify reconcile through the gate:

```bash
curl --resolve reconcile.ninjalabo.ai:443:10.0.0.1 https://reconcile.ninjalabo.ai -s -o /dev/null -w '%{http_code}\n'
```

Expected: an app response (not 403) from inside the VPN; 403 from outside.

- [ ] **Step 9: Record completion**

Report the E2E checklist results to the user. boxrecipe remote/GitHub Pages setup is out of scope here — offer it as follow-up.

---

## Self-Review Notes

- **Spec coverage:** service model + validation (T3), heredoc guard (T3), discopipe fragment incl. dedicated user/hardening/no-secrets (T4), `box_recipe` + SERVICES + PEERS + port uniqueness + rebuild flow + split-DNS doc (T5), `caddy_site` public + TDD asserts incl. 403-fallback (T1), deploy removal incl. stale module + `_modidx` (T6), skill update incl. env-file step + split-DNS (T7), E2E incl. `cloud-init status` / unit checks / Discord round-trip / laptop-bot-disabled check (T8). Out-of-scope items (Docker, automated DNS, rate limiting, health checks) have no tasks — correct.
- **Deviation from spec order:** deploy removal moved after the boxrecipe port (see Global Constraints).
- **Type consistency:** `check_service`/`service_site`/`write_file_cmd` signatures match across T3→T4→T5; `box_recipe(services, peers, ssh_key, hostname)` matches the spec's rough signature; `caddy_site(vpn_subnet:str|None)` matches T1→T3.
