# Brief: s1-opmc-ansible — how it works, how it's structured, and what's worth pulling into OPMC

**Audience note:** this reads as an engineering/product review doc, not an operator guide (that's what README/INSTALL/UPGRADE.md are for). It's organized to answer "what is this," "how is it built," and "what should we steal."

---

## 1. How this project works

### The problem it sits on top of

SentinelOne's self-hosted Management Console ships as a multi-GB TGZ containing **its own bundled Ansible playbook** (`docker_ansible/deploy_mgmt.yml`) plus a pinned Ansible/Python that the package installs onto the target itself. That vendor playbook is the actual installer — this repo does not reimplement it or fork it. What it automates is the **~15 manual steps documented around it**: OS/resource gating, package staging, cert placement, config editing, Python/Docker/Ansible bootstrapping, the handoff to the vendor playbook, and post-deploy fixups — the part that's currently a runbook a human executes by hand, one `docs` article at a time.

### Execution model

Two playbooks, run from a **control node over SSH** — deliberately never on the target box itself, because the vendor package installs its own pinned Ansible/Python onto the target, and running this playbook there would collide with it over `/usr/bin/python` alternatives and `$PATH`.

- **`install.yml`** — clean install, one run.
- **`upgrade.yml`** — **one version hop per run**, chain-enforced (`S-25.3.2 → O-25.4.4 → O-25.4.6 → O-26.1.1 → O-26.3.1`); skipping a hop is refused outright unless a human overrides it.

Each playbook is a thin wrapper: a `raw`-only Python bootstrap play (has to run before Ansible facts can even be gathered on RHEL 8, which ships Python 3.6), then a role. Inside the role, work is split into **numbered, independently-taggable stages** (`00_facts` → `10_preflight` → `20_stage` → `30_certs` → `40_bootstrap` → `50_config` → `60_deploy` → `70_post` → `80_verify`), so `--tags verify` or `--tags preflight` re-runs just that slice.

### The engineering patterns worth product/eng's attention

These are the parts that generalize beyond "an SE's install script" — they're solving problems the *vendor package itself* has:

- **Runtime discovery instead of a hardcoded table.** The vendor moves `ansible-playbook` and its Python interpreter between releases, and *differently* depending on OS major even within the same release prefix (traced across 5+ layouts across RHEL 8/9/10 and Ubuntu 22/24). A lookup table breaks every time the vendor changes packaging. This project instead: (1) checks the one path every release creates "for backward compatibility," (2) resolves symlinks and reads the shebang to get the real interpreter, (3) verifies that interpreter can `import docker` before letting the 20+ minute deploy start. Self-healing against vendor packaging drift, with zero code changes needed for each new layout encountered.
- **Checksum-gated staging.** `checksums.yml` records vendor-published SHA256 + size per tarball; size is checked first (instant, catches truncated downloads), SHA256 second. Prevents burning 20+ minutes of deploy on a corrupt 5GB download.
- **In-place config editing, never templating.** The vendor's `config.yml` carries ~20 keys beyond what an operator sets, including Jinja-derived paths. Overwriting it with a clean template (the obvious approach) silently strips those. This edits only the operator-facing lines and leaves the rest of the vendor file untouched.
- **Fail-fast preflight with actionable messages**, replacing "vendor deploy dies 20 minutes in with `Cannot allocate memory`": UID/GID 847 collision, RAM/vCPU floor, AVX (Mongo 5.0+ requires it — a Proxmox `kvm64` default masks it), `/tmp noexec`, disk space (advisory by default, hard-fail optional), iptables presence, cert passphrase, admin password character rules — all checked before anything on the host is modified.
- **Fixing bugs in the vendor's own scripts, precisely scoped.** Ubuntu 24.04's vendor installer never installs the `docker` Python module it needs, and carries a compatibility patch that silently patches the wrong directory and no-ops (Python 3.12's removed `configparser.readfp()`). Both are fixed here, gated on the exact condition that causes them (not "this is Ubuntu"), and **reapplied every run** — because the vendor's own re-run of `install_ansible.sh` uses `--ignore-installed` and reverts the patch.
- **Air-gap tiering.** RHEL 8's Python bootstrap tries, cheapest first: existing interpreter → configured repos (RHSM/Satellite/DVD, doesn't care which) → **the RPMs already inside the vendor tarball** → fail with the three remaining manual options spelled out. Verified working with zero repos configured on the host at all.
- **Health verification that waits for containers to settle** rather than snapshotting once (a container in `health: starting` at 1 minute can be healthy by 10 — this was observed live, not assumed), and separates "down by vendor design on this release" from "unexpectedly broken," so a real regression can't hide behind a known-noisy container list.
- **Upgrade-specific correctness fixes**: a snapshot-confirmation gate (won't run without `-e s1_snapshot_confirmed=true`), Postgres major-version detection to decide whether a Postgres 11→15 migration is even needed, and — notably — **mount_dir handling that overrides what the vendor's own upgrade doc says to do**, because following it literally relocates `/data/uploads` to `/uploads` and orphans `/data/s1` on any console installed the standard way. That was reproduced live on an `S-25.3.2 → O-25.4.4` hop.

### How it's validated

There's a real test matrix (RHEL 8.10/9.x/10.x, Ubuntu 22.04/24.04, now including O-26.3.1 on RHEL 9.9, RHEL 10.3 and Ubuntu 24.04.5, both clean-install and full-upgrade-chain routes), run against live hosts with a documented status legend (validated / in progress / vendor-supports-untested / roadmap / vendor-doesn't-support). This is functionally the project's CI record — there's no way to containerize an OS-level vendor installer that reboots hosts and manages Docker, so validation happens on real VMs, snapshotted between runs.

---

## 2. Structure of the project

```
install.yml                  clean-install playbook (2 plays: python bootstrap, then s1_mgmt role)
upgrade.yml                  one upgrade hop per run (2 plays: python bootstrap, then s1_upgrade role)
tasks/bootstrap_python.yml   shared: gets a usable Python onto RHEL 8 targets before anything else runs

roles/s1_mgmt/               the install role
  defaults/main.yml            ~30 tunables — package path, resource floors, AVX/storage gates, etc.
  tasks/
    00_facts.yml                derive version/prefix from filename, resolve install paths
    10_preflight.yml             OS/version/UID-GID/RAM/AVX/storage/iptables gates — fail fast
    20_stage.yml                 checksum-verify, unpack tarball, confirm version.json matches
    30_certs.yml                 stage BYO cert/key, refuse world-writable dirs, reject passphrases
    40_bootstrap.yml             dispatch to 41/42, then discover ansible-playbook + interpreter
    41_bootstrap_rhel.yml        offline_installation.sh install_docker_and_pip / install_ansible
    42_bootstrap_ubuntu.yml      Ubuntu's two standalone scripts (no dispatcher exists there)
    43_bootstrap_ubuntu_fixups.yml   patches for bugs in the vendor's Ubuntu scripts
    50_config.yml                 edit vendor config.yml in place (never overwrite)
    60_deploy.yml                  hand off to deploy_mgmt.yml with the resolved interpreter
    70_post.yml                    force_legacy_token, incoming-command-processor workaround
    80_verify.yml                  wait for container health to settle, HTTPS liveness check

roles/s1_upgrade/             the upgrade role — reuses s1_mgmt's facts/stage/bootstrap via
                               import_role rather than duplicating; only diverges where upgrade
                               semantics genuinely differ
  defaults/main.yml             upgrade order list, snapshot gate, mount_dir/postgres overrides
  tasks/
    00_facts.yml                 + reads installed version, validates it's a legal chain step,
                                  detects current Postgres major version
    10_preflight.yml              + snapshot-confirmation gate, existing data-dir probing
    40_config.yml                  mount_dir handling that preserves existing data paths
    50_presteps.yml                 metrics/SMTP/force_legacy_token settings.ini edits
    60_deploy.yml                    hands off to deploy_mgmt.yml with Postgres migration flags
    70_post.yml / 80_verify.yml      same shape as install

group_vars/all/
  main.yml.example              operator-facing non-secret config (copy to main.yml, gitignored)
  vault.yml.example             secrets template (copy to vault.yml, ansible-vault encrypted)

inventory.ini.example           copy to inventory.ini (gitignored)
ansible.cfg                     SSH keepalives/ControlPersist tuned for a 20-40 min single SSH command
checksums.yml                    known-good SHA256 + size per tarball, vendor vs. local provenance
requirements.yml                 the one external collection dependency (community.general)
certificates/                    your certificate.pem / certificate.key (+ its own README)
installers/                      gitignored — where operators stage the multi-GB vendor tarballs

README.md / INSTALL.md / UPGRADE.md   architecture+test-matrix / step-by-step install / step-by-step upgrade
LICENSE.md                        MIT — for this repo's own code only
```

**Two structural decisions that matter for reuse:** the upgrade role is not a copy-paste of the install role — it imports the install role's `00_facts`/`20_stage`/`40_bootstrap` task files directly and only forks logic where upgrade semantics are genuinely different (config, presteps, Postgres handling). And the vendor's own support matrix (which RHEL/Ubuntu majors a given release supports) is read from the package's own directory layout and `offline_installation.sh` case statement at runtime wherever possible, rather than duplicated into a static table that would drift.

---

## 3. Candidates worth pulling into shipping OPMC

Several of the fixes above aren't "this repo's automation," they're **corrections to bugs or gaps in the vendor package itself** — meaning they'd have value even outside an Ansible-wrapper context:

| Candidate | Why it's more than a wrapper convenience |
|---|---|
| **Runtime discovery** (resolve binary → shebang → verify `import docker`) | Directly fixes the class of bug that broke this project every time the vendor moved paths. Baking discovery into the installer itself removes the whole failure mode for *any* automation, not just this one. |
| **Ubuntu bootstrap fixups** (missing `docker` module install, broken `readfp` patch, missing `/usr/bin/python` symlink) | These are bugs in `install_docker_and_pip.sh`/`install_ansible.sh` themselves. Fixing them upstream means every Ubuntu customer benefits, not just ones using this repo. |
| **RHEL 10 reboot-required guard exiting 0** | A success exit code on a "reboot required, nothing installed" condition is a footgun for any automation layered on top, including the product's own tooling. |
| **Preflight gates** (UID/GID 847, AVX, RAM/vCPU floor, `/tmp noexec`, iptables) | Today the vendor deploy fails deep and unclearly (`Cannot allocate memory` after 20 minutes, Mongo crash-looping on missing AVX). A first-party preflight subcommand shipped with the installer would catch these for every customer, scripted or not. |
| **Upgrade `mount_dir` handling** | This is a correctness fix for the vendor's *own documented* upgrade procedure — following the doc as written relocates `/data/uploads` and orphans `/data/s1` on a standard install. Worth flagging to product/docs regardless of whether any of this repo ships. |
| **Checksum sidecar pattern** | Cheap integrity gate before a 5GB extract; the product could ship its own manifest alongside each release. |
| **Settle-before-judging container health check + expected-broken list** | Could become a first-party `sentinelmgmtctl status` command — the "is my install actually healthy" question currently has no vendor-supported answer short of `docker ps` and tribal knowledge of which containers are noisy by design. |
| **No-skip upgrade chain enforcement** | Currently enforced only by this repo reading an ordered list; enforcing it in the vendor's own upgrade playbook would protect every customer following the docs by hand. |

Happy to turn any row of that table into a standalone writeup (repro steps, exact vendor file/line, suggested fix) if it'd help make the case to whoever owns the installer packaging.
