# s1-opmc-ansible

A one-shot Ansible playbook that installs the **SentinelOne self-hosted
(on-premises) Singularity Platform Management Console** (TGZ deployment) on
Red Hat Enterprise Linux **8.10**, **9.7** and **10.2**. See the
[test matrix](#test-matrix) for exactly which release runs where — RHEL 10
takes O-26.1.1 and nothing earlier.

```bash
ansible-playbook install.yml --ask-vault-pass                       # clean install
ansible-playbook upgrade.yml -e s1_snapshot_confirmed=true          # one upgrade hop
```

See [INSTALL.md](INSTALL.md) for the full procedure, including where to place
the deployment package and your certificates, and [UPGRADE.md](UPGRADE.md) for
upgrading an existing console along the supported path
`S-25.3.2 → O-25.4.4 → O-25.4.6 → O-26.1.1`.

---

## What this actually does

The SentinelOne package **already ships its own Ansible playbook**
(`docker_ansible/deploy_mgmt.yml`) plus a pinned Ansible that the vendor
installs on the target via pip. This project does not reimplement that. It
automates the ~15 manual steps documented around it:

| Stage | File | What it does |
|---|---|---|
| Facts | `00_facts.yml` | Derives version + prefix from the filename and resolves paths (the Ansible runtime is discovered later, at bootstrap) |
| Preflight | `10_preflight.yml` | OS gate, UID/GID 847 free, no `s1` user, inputs present, disk space, iptables |
| Stage | `20_stage.yml` | Unpacks the TGZ into `~/<VERSION>/`, verifies `version.json` matches |
| Certs | `30_certs.yml` | Copies your key/PEM to the target, rejects passphrase-protected keys |
| Bootstrap | `40_bootstrap.yml` | Runs `offline_installation.sh`, ensures Docker is up, resolves `ansible-playbook` |
| Config | `50_config.yml` | Edits the vendor `config.yml` **in place** (never overwrites it) |
| Deploy | `60_deploy.yml` | Hands off to `deploy_mgmt.yml` with the correct interpreter |
| Post | `70_post.yml` | Verifies `mgmt_hostname`, sets `force_legacy_token` |
| Verify | `80_verify.yml` | Checks every container is `Up` and the console answers HTTPS |

Run a single stage with tags, e.g. `--tags preflight` or `--tags verify`.

## Two design decisions worth knowing

**It runs from a control node over SSH, not on the box.** The vendor installs a
pinned Ansible 2.15.x tied to a specific Python onto the target. If this
playbook ran *on* the target, the two would fight over `/usr/bin/python`
alternatives and `ansible-playbook` in `$PATH`. Kept separate, the vendor's
Ansible is just a command we shell out to.

**The vendor `config.yml` is edited in place, never templated over.** The
shipped file carries ~20 keys beyond the handful an operator sets, including
derived paths:

```yaml
postgres_dir: "{% if mount_dir %}{{ mount_dir }}/postgres{% else %}/postgres{% endif %}"
```

Overwriting it with just the operator-facing keys would strip those and break
the deploy. Values you leave empty are skipped entirely, so the vendor's blank
default survives — which is exactly what an air-gapped install needs.

## Ansible runtime — discovered, not tabulated

The vendor moves `ansible-playbook` between releases, and it differs **for the
same prefix on the same OS**:

| Package | RHEL | Real `ansible-playbook` | Mechanism |
|---|---|---|---|
| S-25.3.2 | 8 / 9 | `/root/.local/bin/` | `pip3 install --user` as root |
| O-25.4.4 | 8 / 9 | `/usr/local/bin/` | `pip --prefix=/usr/local` |
| O-25.4.6+ | 8 | `/usr/local/bin/` | `pip --prefix=/usr/local` |
| O-25.4.6+ | 9 | `/opt/sentinelone/installer/bin/` | venv, `python3.11` |
| O-26.1.1 | 10 | `/opt/sentinelone/installer/bin/` | venv, `python3.12` |

This repo previously kept that as a table keyed on prefix + OS major, which
cannot express the O-25.4.4 / O-25.4.6 split. It was wrong for O-25.4.4 on
RHEL 9 and aborted the upgrade at bootstrap.

**Note the vendor's own upgrade article is wrong here too**: it prescribes
`-e ansible_python_interpreter=/opt/sentinelone/installer/bin/python3.11` for
the O-25.4.4 hop (`docs/docs-000005155.txt`), but that package builds no venv,
so the path does not exist. Following the article literally fails.

So the runtime is now read off the host after `offline_installation.sh` runs:

1. **Binary** — `/root/.local/bin/ansible-playbook`. Every release examined
   creates it, as a real file or a symlink, because the vendor docs tell
   customers to invoke that path. The script comments call it out explicitly as
   "for backward compatibility".
2. **Interpreter** — the shebang of that binary, after resolving symlinks. It
   names the Python the vendor installed its Ansible into, which is also the
   one carrying the `docker` package: `/usr/bin/python3.11` for O-25.4.4,
   the venv's Python for O-25.4.6+.
3. **Checked** — the derived interpreter must `import docker` before the deploy
   is allowed to start. A mismatch otherwise surfaces 20+ minutes in.

Override with `s1_playbook_bin` / `s1_python_interpreter` to skip discovery. If
the documented path is missing, the playbook searches the host and reports the
candidates rather than failing blind.

## The install is fully offline

`offline_installation.sh` writes its own `/etc/yum.repos.d/sentinelone.repo`
pointing at `file://.../rhel8` (or `rhel9`) inside the unpacked package, and
runs every install with `--disablerepo=* --enablerepo=sentinelone`. **RHSM
registration, the DVD repo and internet access are all irrelevant** to the
bootstrap — it works identically on a registered or unregistered host.

The single exception: the vendor docs suggest `yum install python3.8*` for
S-prefix on RHEL 8. That package is *not* in the bundled repo (S ships
`python36`, O ships `python3.11`), so it would need RHSM or a DVD repo. The
docs explicitly accept Python 3.6 as the fallback, so this is off by default —
enable with `s1_install_python38: true` only if you have an external repo.

## Air-gapped operation

The vendor install never touches your repos — `offline_installation.sh` runs
everything `--disablerepo=* --enablerepo=sentinelone` against RPMs bundled in
the tarball. **One dependency reaches outside, and only on RHEL 8**: this
playbook needs Python 3.8+ on the target, and RHEL 8.10 ships only 3.6 while
ansible-core 2.18+ requires 3.8+. RHEL 9.x ships 3.9 and is unaffected.

[tasks/bootstrap_python.yml](tasks/bootstrap_python.yml) resolves it in tiers,
cheapest first:

1. A suitable interpreter is already present → nothing to do.
2. `dnf` from whatever repos the host has. Covers RHSM/CDN, **Satellite or
   Capsule**, a local `reposync` mirror, and a mounted DVD repo equally — dnf
   does not care that the repo is internal.
3. **The RPMs inside the tarball.** O-prefix packages bundle `python3.11` and
   `python39` under `docker_ansible/rhel/rhel8`. Needs no external access at
   all. *Verified on a host with zero repos configured.*
4. Fail, naming the remaining options.

**S-prefix packages bundle only `python36`**, so tier 3 cannot rescue them. For
a fully air-gapped **S-25.3.2** install with no repo of any kind, use
**ansible-core ≤ 2.16 on the control node** — the last release supporting
Python 3.6 managed nodes, requiring nothing on the target:

```bash
python3 -m venv ~/s1-ansible
~/s1-ansible/bin/pip install 'ansible-core>=2.16,<2.17' ansible
```

## Storage

`mount_dir` drives everything else in the vendor config:

```yaml
postgres_dir → {{ mount_dir }}/postgres
mongo_dir    → {{ mount_dir }}/mongo
upload_dir   → {{ mount_dir }}/uploads
docker_dir   → {{ mount_dir }}/dockers
log_dir      → {{ mount_dir }}/s1
```

With `s1_mount_dir: /data`, **only `/data` needs to exist** — the playbook
creates the rest. The bare `/postgres` and `/mongo` paths (and the
`Device or resource busy` errors in the vendor troubleshooting docs) only
occur when `mount_dir` is left empty.

A separate partition for `/data` is a **sizing guardrail, not an installer
requirement**. Disk checks are advisory by default; set
`s1_storage_fail_hard: true` to make them blocking for a production run, or
`s1_verify_storage: false` to skip them.

## Requirements

- **Control node**: Ansible 2.14+, the `community.general` collection
  (`ansible-galaxy collection install -r requirements.yml`). For a fully
  air-gapped RHEL 8 target with no repositories, use ansible-core ≤ 2.16 —
  see [Air-gapped operation](#air-gapped-operation).
- **Target**: RHEL 8.x, 9.x or 10.x, x86_64, SSH access with passwordless sudo.
  RHEL 10 requires O-26.1.1 or later — see the [test matrix](#test-matrix)
- **A CPU with AVX** — O-prefix packages ship MongoDB 5.0+, which requires it.
  Proxmox's default `kvm64` model masks AVX; use `host` or `x86-64-v3`.
- **Memory** — 20+ containers. The playbook defaults to a 16 GB floor and warns
  below 32 GB, 8 vCPU recommended. Check the vendor prerequisites article for
  official figures.
- UID/GID **847** free, and no pre-existing `s1` user or group
- `/tmp` not mounted `noexec`
- The deployment package pre-staged on the target (see [INSTALL.md](INSTALL.md))

Preflight verifies all of these before modifying anything:
`ansible-playbook install.yml --tags preflight`

## Tunables

Full defaults live in
[roles/s1_mgmt/defaults/main.yml](roles/s1_mgmt/defaults/main.yml) and
[roles/s1_upgrade/defaults/main.yml](roles/s1_upgrade/defaults/main.yml). The
ones you are most likely to need:

| Variable | Default | Purpose |
|---|---|---|
| `s1_package_path` | — | Path to the tarball **on the target** |
| `s1_install_base` | home of `ansible_user` | Where `~/<VERSION>/` is unpacked |
| `s1_mount_dir` | `/data` | Parent for all console data paths |
| `s1_force_redeploy` | `false` | Re-run the deploy over an existing install |
| `s1_require_checksum` | `true` | Refuse packages absent from `checksums.yml` |
| `s1_check_resources` | `true` | Enforce the RAM/vCPU floors |
| `s1_min_ram_gb` / `s1_rec_ram_gb` | `16` / `32` | Hard floor and recommendation |
| `s1_require_avx` | `true` | Block O-prefix installs on a CPU without AVX |
| `s1_verify_storage` / `s1_storage_fail_hard` | `true` / `false` | Disk checks; advisory by default |
| `s1_install_python38` | `false` | S-prefix on RHEL 8 only; needs an external repo |
| **Upgrade only** | | |
| `s1_snapshot_confirmed` | `false` | Must be `true` — the snapshot gate |
| `s1_skip_chain_check` | `false` | Bypass version-skip protection (see below) |
| `s1_force_empty_mount_dir` | `false` | Force the vendor doc's `mount_dir` form |
| `s1_postgres_dir` / `s1_mongo_dir` | auto-detected | Override database locations |
| `s1_dump_dir` | `/tmp/backup/postgres` | Scratch space for a Postgres 11→15 migration |
| `s1_force_legacy_token` | `null` (untouched) | Set `false` for EDG / hybrid-EDR |

**Versions cannot be skipped.** `upgrade.yml` reads `/s1/version.json`, computes
the one legal next hop, and refuses anything else — S-25.3.2 → O-26.1.1 is three
separate runs. `s1_skip_chain_check=true` bypasses *this* playbook's check but
not the product's migrations, which assume each prior release ran; use it only
for a path Support has approved.

## Repository layout

```
install.yml                  the one-shot install playbook
upgrade.yml                  one upgrade hop per run
tasks/bootstrap_python.yml   installs python3.11 on RHEL 8 before anything else
inventory.ini.example        copy to inventory.ini (gitignored)
ansible.cfg                  SSH keepalives for the long deploy
checksums.yml                known-good SHA256 sums for the tarballs
group_vars/all/main.yml.example   copy to main.yml (gitignored)
group_vars/all/vault.yml     secrets (gitignored; encrypt with ansible-vault)
certificates/                your certificate.pem / certificate.key
installers/                  deployment tarballs (gitignored)
roles/s1_mgmt/               the install role
roles/s1_upgrade/            the upgrade role (reuses staging + bootstrap)
```

## Test matrix

What has actually been run, and where the vendor packages can be run at all.
Two different things, so one legend covers both:

| | Meaning |
|---|---|
| ✅ | Validated here, end to end, on a live host |
| 🚧 | Under test right now |
| ⬜ | The vendor package supports it; this project has not tested it |
| 🔜 | Not supported yet by these playbooks; on the roadmap |
| ❌ | The vendor package does not ship an installer for it — `offline_installation.sh` exits with `Error: RHEL <version> is not supported` |

### Release × platform

Support is read from the package itself: the `case $RHEL_VERSION` arms in
`docker_ansible/rhel/offline_installation.sh` and the
`docker_ansible/{rhel/rhel*,ubuntu_*}` directories, checked in all four
tarballs on 2026-08-04.

| Release | RHEL 7 | RHEL 8 | RHEL 9 | RHEL 10 | Ubuntu 14–22 | Ubuntu 24.04 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| S-25.3.2 | ⬜ | ✅ 8.10 | ✅ 9.7 | ❌ | 🔜 | ❌ |
| O-25.4.4 | ⬜ | ✅ 8.10 | ✅ 9.7 | ❌ | 🔜 | 🔜 |
| O-25.4.6 | ⬜ | ✅ 8.10 | ✅ 9.7 | ❌ | 🔜 | 🔜 |
| O-26.1.1 | ⬜ | ✅ 8.10 | ✅ 9.7 | ✅ 10.2 | 🔜 | 🔜 |

### Campaign per host

| Host | OS | Clean install | Upgrade chain | Final state |
|---|---|---|---|---|
| s1console | RHEL 8.10 | ✅ S-25.3.2 | ✅ all three hops | O-26.1.1, 21 containers, none down |
| s1console02 | RHEL 9.7 | ✅ S-25.3.2 | ✅ all three hops | O-26.1.1, 21 containers, none down |
| rhel03 | RHEL 10.2 | ✅ O-26.1.1 | ❌ not possible | O-26.1.1, 21 containers, none down |

**RHEL 10 takes a clean O-26.1.1 install and nothing else.** RHEL 10 support
arrives in O-26.1.1 — S-25.3.2, O-25.4.4 and O-25.4.6 have no `10*` arm and
refuse to install. Since no release that could *precede* O-26.1.1 will run on
RHEL 10, there is no chain to upgrade along. Both preflights enforce this
(`s1_min_version_rhel10`), so the failure is a clear message rather than a
vendor error 20 minutes in.

### Ubuntu

Not supported by these playbooks yet, and the gap is real work rather than a
flipped flag:

- The vendor ships a complete parallel installer — `docker_ansible/ubuntu_24/`
  carries its own `install_docker_and_pip.sh`, `install_ansible.sh`, Docker
  engine `.deb`s and the Ansible/`docker` wheels. Ubuntu 24.04 has been in the
  package since **O-25.4.4**; 14/18/20/22 go back further.
- But there is **no `offline_installation.sh` outside `rhel/`**. That script is
  the dispatcher this project shells out to in `40_bootstrap.yml`, so Ubuntu
  needs a separate bootstrap path calling the two scripts directly.
- Preflight also asserts `ansible_distribution == 'RedHat'`, and the Python
  bootstrap is `dnf`-only.

**24.04 LTS is the intended starting point** when this is picked up — it is the
only Ubuntu the current release line still ships alongside RHEL 10.

## Status

Validated end-to-end on **RHEL 8.10 and RHEL 9.7**: a clean install of
S-25.3.2, then the complete upgrade chain, every hop on both platforms
reporting `failed=0` with the console serving HTTPS throughout.

```
S-25.3.2  →  O-25.4.4  →  O-25.4.6  →  O-26.1.1
```

**RHEL 10.2** is validated for a clean O-26.1.1 install — `failed=0`, 21
containers, console serving HTTPS. There is no chain to run there; see
[RHEL 10.2 notes](#rhel-102-notes).

Exercised against a live host:

- Package checksum verification against vendor-published sums (all four)
- The Python bootstrap, both from configured repos and **fully air-gapped**
  with zero repositories, installing from the tarball's own bundled RPMs
- In-place config editing, preserving every vendor key
- Runtime discovery across all three vendor layouts — `pip3 --user`,
  `--prefix=/usr/local` and the venv — including the mid-chain change on RHEL 9
- Upgrade chain validation, Postgres version detection, database-path probing
- The S→O prefix transition, including the MongoDB 4.4 → 5.0 jump that makes
  AVX mandatory

Final state on both platforms: O-26.1.1, 21 containers, none down.

### RHEL 9.7 notes

Two things differ from RHEL 8.10, both already handled:

- **No Python bootstrap runs.** RHEL 9 ships Python 3.9, so
  `tasks/bootstrap_python.yml` exits at its first probe. The air-gapped
  bundled-RPM tier is therefore exercised on RHEL 8 only.
- **The vendor moves `ansible-playbook` mid-chain.** O-25.4.4 installs to
  `/usr/local/bin`; O-25.4.6 switches RHEL 9 to a venv under
  `/opt/sentinelone/installer`. A hardcoded table cannot express that — see
  [Ansible runtime](#ansible-runtime--discovered-not-tabulated). This is
  invisible on RHEL 8, where every O release uses `/usr/local/bin`.

The RHEL 9.7 host had **7.5 GB RAM**, below the 16 GB floor, run deliberately
with `-e s1_min_ram_gb=7`. Install and all three hops completed with no
allocation failures, but it sat at ~6.9 GB used with ~1.2 GB of swap in play
throughout. That is a data point, not a recommendation — there was no headroom,
and nothing here justifies lowering the default floor.

### RHEL 10.2 notes

Clean install of O-26.1.1, `failed=0`, 21 containers up, HTTPS 200. What
differs from the earlier platforms:

- **Install only, by force.** RHEL 10 support arrives in O-26.1.1, so nothing
  that could precede it will install — see the [test matrix](#test-matrix).
  The upgrade role is therefore untested on RHEL 10, and will stay that way
  until a release ships that O-26.1.1 can upgrade *to*.
- **No Python bootstrap**, as on RHEL 9 — RHEL 10 ships `python3.12` as
  `/usr/bin/python3`, and `tasks/bootstrap_python.yml` exits at its first probe.
- **A fourth vendor runtime layout, discovered without a code change.** The
  venv is `python3.12`-based:

  ```
  binary      : /root/.local/bin/ansible-playbook (documented path)
  interpreter : /opt/sentinelone/installer/bin/python3.12 (from shebang)
  docker pkg  : importable
  ```

  This is the case that justifies [reading the runtime off the
  host](#ansible-runtime--discovered-not-tabulated): a prefix+OS-major table
  would have needed a new row for RHEL 10, and would have been wrong until
  someone added it.
- **The vendor's RHEL 10 reboot condition did not trigger.** `offline_installation.sh`
  can install missing kernel modules and demand a reboot before the deploy can
  continue; `40_bootstrap.yml` detects that and stops with instructions. The
  vendor documents it as mostly affecting AWS Marketplace images, and this
  host (a standard install) did not hit it — so that guard remains untested.

Like the RHEL 9.7 host, this VM had **7.5 GB RAM and 4 vCPUs**, below both
floors, run with `-e s1_min_ram_gb=7`. It finished clean but settled at
~6.9 GB used with ~1.5 GB of swap in play. Same caveat as before: a data point,
not a recommendation.

**Not yet exercised:** the Postgres 11→15 migration path (`upgrade_postgres` /
`dump_dir` — every console tested was already on 15), any upgrade hop on
RHEL 10, RHEL 7, and every Ubuntu release.

Behaviour here was derived from the vendor's own `offline_installation.sh`,
`deploy_mgmt.yml` and `group_vars/all/config.yml`, plus SentinelOne's
self-hosted install and upgrade articles. Those are behind a support login and
are deliberately not redistributed in this repository.
