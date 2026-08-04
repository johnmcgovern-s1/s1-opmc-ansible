# s1-opmc-ansible

A one-shot Ansible playbook that installs the **SentinelOne self-hosted
(on-premises) Singularity Platform Management Console** (TGZ deployment) on
Red Hat Enterprise Linux **8.10** and **9.7**.

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
| O-25.4.6+ | 9 / 10 | `/opt/sentinelone/installer/bin/` | venv |

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
- **Target**: RHEL 8.x or 9.x, x86_64, SSH access with passwordless sudo
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

## Status

Validated end-to-end on **RHEL 8.10 and RHEL 9.7**: a clean install of
S-25.3.2, then the complete upgrade chain, every hop on both platforms
reporting `failed=0` with the console serving HTTPS throughout.

```
S-25.3.2  →  O-25.4.4  →  O-25.4.6  →  O-26.1.1
```

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

**Not yet exercised:** the Postgres 11→15 migration path (`upgrade_postgres` /
`dump_dir` — every console tested was already on 15), and RHEL 10 /
`python3.12`.

Behaviour here was derived from the vendor's own `offline_installation.sh`,
`deploy_mgmt.yml` and `group_vars/all/config.yml`, plus SentinelOne's
self-hosted install and upgrade articles. Those are behind a support login and
are deliberately not redistributed in this repository.
