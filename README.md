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
| `s1_cert_dir` | `/etc/sentinelone/certs` | Where the key/PEM are staged **on the target**. The vendor article says `/tmp`; don't — see below |
| **Upgrade only** | | |
| `s1_snapshot_confirmed` | `false` | Must be `true` — the snapshot gate |
| `s1_skip_chain_check` | `false` | Bypass version-skip protection (see below) |
| `s1_force_empty_mount_dir` | `false` | Force the vendor doc's `mount_dir` form |
| `s1_postgres_dir` / `s1_mongo_dir` | auto-detected | Override database locations |
| `s1_dump_dir` | `/tmp/backup/postgres` | Scratch space for a Postgres 11→15 migration |
| `s1_force_legacy_token` | `null` (untouched) | Set `false` for EDG / hybrid-EDR |

**Do not stage certificates in `/tmp`.** The vendor install article says to copy
them there, and the file being copied is an unencrypted private key — the same
article requires it to be passphrase-less. `/tmp` is world-writable, the article
specifies no file modes, and nothing removes the key afterwards, so it stays
readable for the life of the console.

`30_certs.yml` refuses a world-writable `s1_cert_dir` rather than staging a key
into one (`s1_allow_world_writable_cert_dir=true` to override). It also creates
the directory **only when absent** and never alters the permissions of one it
did not create — an earlier version enforced `0700 root:root` on whatever path
it was given, which turned `/tmp` from `1777` into root-only and broke every
non-root process depending on it.

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

What has actually been run, and where the vendor packages can be run at all —
two different things, so the status legend covers both:

| | Meaning |
|---|---|
| ✅ | Validated here, end to end, on a live host |
| 🚧 | Under test right now |
| ⬜ | The vendor package supports it; this project has not tested it |
| 🔜 | Not supported yet by these playbooks; on the roadmap |
| ❌ | The vendor package does not ship an installer for it — `offline_installation.sh` exits with `Error: RHEL <version> is not supported` |

A ✅ also records **how** that release was reached, because the two routes run
different code and are not interchangeable:

| | Route |
|---|---|
| **(I)** | Clean install — `install.yml` against a bare host |
| **(U)** | Reached by upgrading from the release above it — `upgrade.yml` |

### Release × platform

Support is read from the package itself: the `case $RHEL_VERSION` arms in
`docker_ansible/rhel/offline_installation.sh` and the
`docker_ansible/{rhel/rhel*,ubuntu_*}` directories, checked in all four
tarballs on 2026-08-04.

| Release | RHEL 7 | RHEL 8 | RHEL 9 | RHEL 10 | Ubuntu 14–20 | Ubuntu 22.04 | Ubuntu 24.04 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| S-25.3.2 | ⬜ | ✅ 8.10 **(I)** | ✅ 9.7 **(I)** | ❌ | 🔜 | ✅ 22.04 **(I)** | ❌ |
| O-25.4.4 | ⬜ | ✅ 8.10 **(I, U)** | ✅ 9.7 **(I, U)** | ❌ | 🔜 | ✅ 22.04 **(U)** | ✅ 24.04 **(I)** |
| O-25.4.6 | ⬜ | ✅ 8.10 **(I, U)** | ✅ 9.7 **(I, U)** | ❌ | 🔜 | ✅ 22.04 **(U)** | ✅ 24.04 **(I, U)** |
| O-26.1.1 | ⬜ | ✅ 8.10 **(I, U)** | ✅ 9.7 **(I, U)** | ✅ 10.2 **(I)** | 🔜 | ✅ 22.04 **(I, U)** | ✅ 24.04 **(I, U)** |

The **(I)** / **(U)** markers show that on **RHEL 8 and RHEL 9 every release has
been validated by both routes** — clean-installed directly onto a bare host, and
also reached by upgrading from S-25.3.2. That matters because installing the
current release directly is the likelier customer path, and it exercises
different code: the install and upgrade roles use separate config stages with
different `mount_dir` handling. RHEL 10 is install-only because no release that
could precede O-26.1.1 runs there.

### Campaign per host

| Host | OS | Clean install | Upgrade chain | Final state |
|---|---|---|---|---|
| s1console | RHEL 8.10 | ✅ S-25.3.2 | ✅ all three hops | O-26.1.1, 21 containers, none down |
| s1console02 | RHEL 9.7 | ✅ S-25.3.2 | ✅ all three hops | O-26.1.1, 21 containers, none down |
| rhel03 | RHEL 10.2 | ✅ O-26.1.1 | ❌ not possible | O-26.1.1, 21 containers, none down |
| rhel01 | RHEL 8.10 | ✅ all four releases | — not run | each verified independently, none down |
| rhel02 | RHEL 9.7 | ✅ all four releases | ✅ all three hops | O-26.1.1, 21 containers, none down |
| ubuntu2404 | Ubuntu 24.04.4 | ✅ all three that ship `ubuntu_24` | ✅ both hops | O-26.1.1, 21 containers, none down |
| ubuntu2204 | Ubuntu 22.04.5 | ✅ S-25.3.2, O-26.1.1 | ✅ all three hops | O-26.1.1, 21 containers, none down |

`rhel01` and `rhel02` ran **install-only campaigns**: every release was installed
onto a host reverted to the same clean snapshot beforehand, so each result stands
on its own rather than inheriting state from the release before it. Container
counts fall identically on both platforms as the vendor retires components — 26
for S-25.3.2, 25 for O-25.4.4, then 21 for both O-25.4.6 and O-26.1.1.

`rhel02` then ran the **upgrade chain separately**, from a fresh S-25.3.2 install
through all three hops, snapshotting between each. So RHEL 9.7 has both routes
validated independently on the same host at the same specification.

**RHEL 10 takes a clean O-26.1.1 install and nothing else.** RHEL 10 support
arrives in O-26.1.1 — S-25.3.2, O-25.4.4 and O-25.4.6 have no `10*` arm and
refuse to install. Since no release that could *precede* O-26.1.1 will run on
RHEL 10, there is no chain to upgrade along. Both preflights enforce this
(`s1_min_version_rhel10`), so the failure is a clear message rather than a
vendor error 20 minutes in.

### Ubuntu

**22.04 and 24.04 LTS are supported**, from bare hosts with no overrides —
`failed=0`, console serving HTTPS, no unexpectedly unhealthy containers.

- **22.04** carries the **full chain**: S-25.3.2 installs, then all three hops
  through to O-26.1.1, plus a direct O-26.1.1 install. Same coverage as RHEL 8
  and 9.
- **24.04** covers every release shipping `ubuntu_24` — O-25.4.4, O-25.4.6 and
  O-26.1.1 — by install and by both hops.

The difference is the packages themselves: **`ubuntu_22` ships in every release
including S-25.3.2, while `ubuntu_24` first appears in O-25.4.4.** So 22.04 is
the only Ubuntu where the S→O prefix transition can be exercised at all.

`ubuntu_14/18/20` are not wired up and preflight refuses them (`ubuntu_14`'s
`install_everything.sh` reaches the internet unconditionally, so it has no
offline path at all).

The two releases install by **entirely different mechanisms**, which is why the
bootstrap discovers rather than assumes:

| | `ubuntu_22` | `ubuntu_24` |
|---|---|---|
| Ansible | `dpkg -i ansible/*` — distro package, **2.10.8** | `pip --break-system-packages` — **2.15.13** |
| `ansible-playbook` | `/usr/bin` | `/usr/local/bin` |
| `docker` Python module | installed by the script | **never installed** — this project supplies it |
| `readfp` patch | absent, and unnecessary on Python 3.10 | present and **silently broken** |
| `python-is-python3` | absent | bundled, installed incidentally |

`s1_ubuntu_tooling` derives `ubuntu_<major>` from the OS rather than mapping it,
and `s1_ubuntu_playbook_candidates` tries both binary locations in order. Adding
a future Ubuntu should need neither change.

**The bare-`python` problem affects every Ubuntu, and used to decide which
releases worked.** `deploy_mgmt.yml` runs `python -c "import uuid;..."`, and no
Ubuntu since 20.04 ships `/usr/bin/python`. That line is byte-identical across
every release examined — the vendor never fixed it. What differed was whether a
release's `pip/` bundle happened to include `python-is-python3`: `ubuntu_24`'s
does from O-25.4.6, `ubuntu_22`'s never does. So the platform worked or failed
on the incidental contents of a bundle.

`s1_ubuntu_provide_python` creates the same symlink that package would, only
when absent. That removed the dependency entirely, and with it the reason
O-25.4.4 was refused on 24.04 — it now installs cleanly there, 25 containers,
none unexpectedly unhealthy. `s1_min_version_ubuntu24` is therefore back to
**25.4.4**, meaning what it says: the first release that ships the tooling.

Ubuntu needs its own bootstrap because the vendor ships two entirely separate
mechanisms. There is **no `offline_installation.sh` outside `rhel/`** — that
script is the dispatcher the RHEL path shells out to — and `ubuntu_24` has no
`install_everything.sh` wrapper either. So `42_bootstrap_ubuntu.yml` calls
`install_docker_and_pip.sh` and `install_ansible.sh` directly, passing the
offline/online argument **explicitly to both**: they interpret it inversely, and
any value other than exactly `true`/`false` installs Ansible from the bundled
wheels while Docker reaches for the internet.

**Two things the vendor scripts leave undone**, both required for the deploy to
run at all, both applied by `43_bootstrap_ubuntu_fixups.yml`:

- `ubuntu_24/python-docker/` ships the `docker` wheels and **neither script
  installs them**. The deploy drives Docker through Ansible's docker modules, so
  without this it cannot start.
- `install_docker_and_pip.sh` carries a patch rewriting the removed
  `configparser.readfp()` for Python 3.12, but it resolves `import ansible` from
  its own directory — where the bundled wheel folder is also called `ansible`.
  It therefore patches the wheel directory, finds nothing, and exits 0. The
  incompatibility survives unpatched on a platform that ships Python 3.12.

Both fixes run **after** the vendor scripts on every invocation, not once:
`install_ansible.sh` uses `pip --ignore-installed`, so a re-run reinstalls
Ansible and reverts the patch.

Everything else generalised without change. Runtime discovery needed only a new
starting path — Ubuntu does not create the `/root/.local/bin/ansible-playbook`
that every RHEL release makes deliberately, so the search begins at
`/usr/local/bin` and reads the same shebang. `tasks/bootstrap_python.yml`
self-skips, since 24.04 already ships Python 3.12.

## Roadmap

Ordered by value, not by effort.

### 1. Ubuntu 24.04 LTS — install and upgrade

**Done for 24.04, by both routes.** All three releases shipping `ubuntu_24` —
O-25.4.4, O-25.4.6 and O-26.1.1 — install cleanly from a fresh snapshot, and the
full chain runs clean, snapshotting between hops:

```
O-25.4.4 (clean install)  →  O-25.4.6  →  O-26.1.1
```

S-25.3.2 ships no `ubuntu_24` directory at all, so 24.04 starts where the
tooling does. That gives Ubuntu 24.04 the same coverage RHEL 8 and 9 have:
every release reachable by install and by upgrade.

Worth noting from the hops: the container set drops 25 → 21 on
`O-25.4.4 → O-25.4.6`, and does so **cleanly** — no stale image, nothing
stranded. The `S→O` transition behaves differently: it leaves a retired
container running the *previous* release's image, which the following hop then
clears. Reproduced identically on RHEL and on Ubuntu 22.04, so it belongs to the
prefix change rather than to retirement hops in general, and it is bounded to
one release rather than accumulating.

Preflight enforces the floor via `s1_min_version_ubuntu24`, in both roles. One
hop is thinner than hoped, but it is the only exercise the upgrade role gets on
a non-RHEL platform, and it covers stages the install path never touches —
`40_config.yml` with its different `mount_dir` handling, `50_presteps.yml`, the
Postgres detection, and the expected-broken container handling.

Two things the campaign settled that a single install could not. The `ubuntu_24`
bootstrap does **not** drift between releases, unlike RHEL where the vendor
moved `ansible-playbook` between releases and between OS majors: both scripts
are byte-identical across O-25.4.4 and O-25.4.6, and every release needed the
same two fix-ups. And the upgrade leaves no stale container behind, unlike the
RHEL `S→O` hop — consistent with that divergence being specific to O-25.4.4.
What varies on Ubuntu is the *bundle contents* and the deploy playbook, not the
installer.

**Done.** 24.04 is complete by both routes, and 22.04 carries the full
four-release chain — the same coverage RHEL 8 and 9 have. The bootstrap selects
tooling and binary path per release rather than assuming one layout, so
`ubuntu_18` and `ubuntu_20` should follow without structural change if they are
ever wanted.

### 2. Straight-to-version installs

**Done.** All four releases have been clean-installed on both RHEL 8.10 and
RHEL 9.7 — eight installs, each onto a host reverted to a fresh snapshot first,
all `failed=0` with the console serving HTTPS. This was worth doing because a
straight install is the more common real-world path: a new customer installs
whatever is current rather than installing a year-old release and chaining
forward.

The paths differ enough that they needed testing rather than assuming:

- `50_config.yml` edits a pristine vendor `config.yml`, and it has only ever
  done that for an S-prefix package on those platforms. The upgrade role uses
  `40_config.yml`, a different file with different `mount_dir` handling.
- Install sets `mount_dir` and lets the vendor derive the data paths; upgrade
  deliberately leaves it empty and sets `postgres_dir` / `mongo_dir` explicitly.
- The S-prefix and O-prefix branches in preflight and bootstrap (AVX, the
  `python38` opt-in, MongoDB 4.4 vs 5.0) are selected differently.

All of those paths are now exercised on both platforms, including straight
installs of the two intermediate releases, which had never been installed
directly anywhere. Nothing in this item remains outstanding; it is kept here as
a record of what the gap was and why it mattered.

### 3. The RHEL 10 kernel-reboot guard

`40_bootstrap.yml` carries a guard that has never been triggered on a live host
— `source review` evidence, not `observed`. Two independent RHEL 10.2 installs
have now run without it firing, on standard (non-Marketplace) images, which is
consistent with where the vendor says it applies rather than evidence the guard
is wrong.

On RHEL 10, `offline_installation.sh` checks for kernel modules before
installing `docker-ce` (RHEL 10's `iptables-nft` has a conditional dependency on
`kernel-modules-extra` that cannot resolve from an offline repo). If they are
missing it installs four bundled RPMs from
`docker_ansible/rhel/rhel10/kernel_modules/`, prints
`KERNEL UPDATE INSTALLED - REBOOT REQUIRED.` — and then **exits 0**.

That exit code is the whole problem. A success-looking return would carry the
play on to the vendor deploy against a host with no Docker, so the guard matches
on stdout instead:

```yaml
when: "'REBOOT REQUIRED' in (s1_bootstrap_docker.stdout | default(''))"
```

**Testing it needs a cloud instance.** The vendor documents this as mostly
affecting AWS Marketplace RHEL 10 images; standard installs on Nutanix or bare
metal ship the modules and never trigger it, which is why `rhel03` did not.
Verify both halves: the play stops at bootstrap with the reboot message, and a
re-run after `sudo reboot` completes. Being a substring match, it would also
fail quietly if the vendor ever rewords the banner — worth re-checking against
each new release.

### Not planned

RHEL 7 (past end of maintenance — preflight warns), Ubuntu 14 through 22, and
the Postgres 11→15 migration path, which needs a console old enough to still be
on Postgres 11.

## Status

Validated end-to-end on **RHEL 8.10 and RHEL 9.7**: a clean install of
S-25.3.2, then the complete upgrade chain, every hop on both platforms
reporting `failed=0` with the console serving HTTPS throughout.

```
S-25.3.2  →  O-25.4.4  →  O-25.4.6  →  O-26.1.1
```

**RHEL 10.2** is validated for a clean O-26.1.1 install — `failed=0`, 21
containers, console serving HTTPS, at unmodified defaults. There is no chain to
run there; see [RHEL 10.2 notes](#rhel-102-notes).

**RHEL 8.10 and RHEL 9.7** additionally have every release validated as a
*straight* clean install, not only as an upgrade target: S-25.3.2, O-25.4.4,
O-25.4.6 and O-26.1.1 were each installed onto a host reverted to a fresh
snapshot beforehand — eight installs in total, all reporting `failed=0` with the
console serving HTTPS.

**Ubuntu 22.04 and 24.04 LTS** are validated. 22.04 carries the full
four-release chain — S-25.3.2 installed, then all three hops to O-26.1.1 — the
same coverage RHEL 8 and 9 have. 24.04 covers every release shipping its tooling
by install and by both hops. `failed=0` throughout, console serving HTTPS,
nothing unexpectedly unhealthy, from bare hosts with no overrides. The vendor's
own `deploy_mgmt.yml` runs on both unmodified; what this project adds is a
separate bootstrap path and fixes for work the vendor's Ubuntu scripts omit —
which differ by release, since the two trees install by different mechanisms.
See [Ubuntu](#ubuntu).

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

Everything recorded above for RHEL 9.7 — the four clean installs and the full
upgrade chain — was run at the playbook's unmodified defaults, with no
overrides of any kind.

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

Validated at the playbook's unmodified defaults, with no overrides — 16 GB and
8 vCPU, finishing with ~6.5 GB still available and swap essentially untouched.

**Not yet exercised:** the Postgres 11→15 migration path (`upgrade_postgres` /
`dump_dir` — every console tested was already on 15), any upgrade hop on
RHEL 10, RHEL 7, and Ubuntu releases before 24.04. The [Roadmap](#roadmap) says
which of those are being picked up and in what order.

Behaviour here was derived from the vendor's own `offline_installation.sh`,
`deploy_mgmt.yml` and `group_vars/all/config.yml`, plus SentinelOne's
self-hosted install and upgrade articles. Those are behind a support login and
are deliberately not redistributed in this repository.
