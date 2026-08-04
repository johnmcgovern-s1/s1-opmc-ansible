# UPGRADE — SentinelOne Management Console

Upgrades an existing self-hosted console one version at a time.

```bash
ansible-playbook upgrade.yml -e s1_snapshot_confirmed=true --ask-vault-pass
```

Budget **40–60 minutes per hop**; the vendor deploy alone takes ~20.

---

## The upgrade path

Versions must not be skipped. From the vendor's path table:

```
S-25.3.2.96  →  O-25.4.4.22  →  O-25.4.6.34  →  O-26.1.1.009
```

**S-25.3.2 is the earliest supported source here.** It is also the final
S-prefix release — every version after it uses the O prefix, starting with
O-25.4.4. A console older than S-25.3.2 must be stepped up to S-25.3.2 using
the vendor procedure before this playbook can take over.

**One hop per run.** The playbook reads `/s1/version.json`, works out the legal
next hop, and refuses anything else:

```
Illegal upgrade hop — versions must not be skipped.
  installed      : S-25.3.2
  staged package : O-25.4.6
  expected next  : O-25.4.4
```

Take a fresh snapshot between hops. To go from S-25.3.2 to O-26.1.1 you run the
playbook three times, staging a different package each time.

Adding a new release is a one-line change — append it to `s1_upgrade_order` in
[roles/s1_upgrade/defaults/main.yml](roles/s1_upgrade/defaults/main.yml).

---

## Before you start

### 1. Take a snapshot — the playbook will not run without one

`s1_snapshot_confirmed` defaults to `false` and blocks the run. This is
deliberate: the documented recovery for a failed upgrade is "revert to your
snapshot".

On the Take Snapshot dialog, **disable "Snapshot the virtual machine's
memory"**. Agents stay connected while a snapshot is taken.

Confirm with `-e s1_snapshot_confirmed=true`.

### 2. Stage the package

Same as a clean install — copy the tarball to the target out of band, then
point the playbook at it:

```bash
scp installers/sentinel_mgmt_o_25_4_4_022.tar.gz <user>@console.example.com:/var/tmp/
```

```yaml
s1_package_path: /var/tmp/sentinel_mgmt_o_25_4_4_022.tar.gz
```

It is unpacked to `~/O-25.4.4/`, alongside the previous version's directory.
Old versions are left in place; pruning them is your responsibility.

The package is checksum-verified before unpacking, using the shared staging
logic — size first, then SHA256 — against [checksums.yml](checksums.yml). Add
the sum for each new package before you upgrade with it; see
[INSTALL.md](INSTALL.md#1-the-deployment-tgz--the-target-server) for the
format. A bad download caught here costs a minute; caught mid-deploy it costs
a snapshot restore.

### 3. Check `/tmp` is not `noexec`

`noexec` on `/tmp` breaks the vendor's pip and Docker installs. Preflight fails
with the fix if it finds it.

### 4. Confirm the CPU has AVX before any S → O hop

**This is the upgrade-specific trap.** S-prefix releases ship MongoDB 4.4,
which runs happily without AVX. O-prefix releases ship MongoDB 5.0+, which
requires it. So a console that has been running fine for months on S-25.3.2
will crash-loop the moment it is upgraded to O-25.4.4 — and the vendor playbook
only discovers it ~7 minutes in, at "Wait for mongo to be available", leaving a
half-migrated console.

```bash
grep -qw avx /proc/cpuinfo && echo OK || echo "MISSING — fix before upgrading"
```

Preflight blocks the hop if AVX is missing and the target is O-prefix. On
Proxmox, set Hardware → Processors → Type to `host` (or `x86-64-v3`) — and
re-check after any snapshot rollback, since reverting restores the old VM
config and silently undoes it.

---

## What the playbook does differently from an install

### `mount_dir` handling — and why it is conditional

The deploy must point at where the data already lives. The playbook probes
`/data/postgres` then `/postgres`, prints the Postgres container's own mounts as
a cross-check, and then picks one of two forms:

**A — the layout is derivable** (`/data/postgres` + `/data/mongo`):

```yaml
mount_dir: /data
```

`postgres_dir`, `mongo_dir`, `upload_dir`, `docker_dir` and `log_dir` all derive
from it, exactly as they did at install. Nothing moves. This is the normal case
for a console built by `install.yml`.

**B — the layout is not derivable** (custom paths):

```yaml
mount_dir:
postgres_dir: /var/postgres
mongo_dir: /etc/mongo
```

This is the form the vendor upgrade article documents.

**Why not always use form B?** Because five paths derive from `mount_dir`, not
two. Pinning `postgres_dir` and `mongo_dir` still leaves the rest to fall
through to their `else` branches:

| Variable | `mount_dir: /data` | `mount_dir:` empty |
|---|---|---|
| `upload_dir` | `/data/uploads` | **`/uploads`** |
| `log_dir` | `/data/s1` | **empty** |

Confirmed live on an S-25.3.2 → O-25.4.4 hop: following the article literally
moved `/data/uploads` to `/uploads` and orphaned `/data/s1`. The upgrade
reported success. On a production console `upload_dir` holds uploaded agent
installer packages, so they would appear to vanish with no error.

Force the vendor's documented form with `-e s1_force_empty_mount_dir=true`, or
override detection entirely:

```bash
-e s1_postgres_dir=/var/postgres -e s1_mongo_dir=/etc/mongo
```

If you are not sure where the databases live, **ask SentinelOne Support before
running** — pointing an upgrade at the wrong path risks data loss.

### Postgres 11 → 15 is handled automatically

The playbook inspects the running `postgres` container image tag and picks the
flags the vendor documents:

| Detected | Action | Flags passed |
|---|---|---|
| 15 or higher | none | *(none)* |
| `11-bullseye`, hop to O-25.4.4 | defer | `upgrade_postgres=false`, `dump_dir` |
| `11-bullseye`, later hops | upgrade | `upgrade_postgres=true`, `dump_dir` |
| plain `11` | upgrade | `upgrade_postgres=true`, `dump_dir` |

**Coming from S-25.3.2 you are already on Postgres 15.4**, so this is the
no-flags case and no dump space is needed.

When a dump *is* required it needs free space equal to 100% of current Postgres
usage. Preflight measures both and fails with the numbers if it will not fit.
Change the location with `-e s1_dump_dir=/mnt/big/pgdump`.

Override detection if it misreads the tag: `-e s1_postgres_version_override=11`.

### `force_legacy_token` is left alone

S and O releases default it to `True`, and `install.yml` sets it explicitly.
**Upgrades do not touch it**, because it must be `False` for EDG or hybrid-EDR
deployments and an earlier upgrade may already have overwritten it.

Set it deliberately if you need to:

```bash
-e s1_force_legacy_token=false    # EDG / hybrid-EDR
-e s1_force_legacy_token=true     # standard
```

Specifically use `False` for: air-gap using the EDG, hybrid EDR self-hosted
with EDG data in Singularity AI SIEM, and hybrid EDR self-hosted with the
Reputation Service and Deep Visibility.

### Expected-broken containers are stopped

The vendor documents these as not working by design after an upgrade. The
playbook stops them and excludes them from the health check, so a clean upgrade
does not report false failures:

- `incoming-command-processor` — gets stuck restarting
- `tags-synchronizer`, `mgmt-entities-uploader`, `mgmt-entities-cdc`,
  `ranger-fingerprint-consumer` — report unhealthy by design

Disable with `-e s1_stop_known_unhealthy=false`.

---

## Procedure

```bash
# 1. Snapshot the VM (memory snapshot disabled)

# 2. Stage the next package on the target
scp installers/<package>.tar.gz <user>@console.example.com:/var/tmp/

# 3. Point at it
$EDITOR group_vars/all/main.yml     # s1_package_path

# 4. Preflight — validates the hop, paths and dump space, changes nothing
ansible-playbook upgrade.yml --tags preflight -e s1_snapshot_confirmed=true

# 5. Upgrade
ansible-playbook upgrade.yml -e s1_snapshot_confirmed=true --ask-vault-pass

# 6. Repeat from step 1 for the next hop
```

The final task prints the next hop in the chain, or tells you when you have
reached the latest supported version.

Afterwards, log in and check **Help > About** (SHIFT+F5 forces a hard refresh).

---

## Stage reference

| Stage | Tag | What it does |
|---|---|---|
| Facts | `always` | Reads `/s1/version.json`, validates the hop, detects Postgres state |
| Preflight | `preflight` | Snapshot gate, OS gate, `/tmp` noexec, locates databases, dump space |
| Stage | `stage` | Unpacks to `~/<VERSION>/` (shared with install) |
| Bootstrap | `bootstrap` | Runs the new package's `offline_installation.sh` (shared with install) |
| Config | `config` | Empty `mount_dir`, explicit `postgres_dir` / `mongo_dir` |
| Presteps | `presteps` | Removes `/tmp/sentinel_mgmt_deploy`, sets `[metrics]`, optional SMTP allowlist |
| Deploy | `deploy` | Runs `deploy_mgmt.yml` with the Postgres flags |
| Post | `post` | Asserts the version file advanced, stops expected-broken containers |
| Verify | `verify` | Container health, HTTPS check, next hop |

---

## A landmine worth knowing

The vendor's own `validate_upgrade_version.yml` fails with
`UPGRADE_VERSION_MISMATCH` when the release prefix changes:

```yaml
when: (current_tag_type == 's_release' and new_tag_type != 's_release') or ...
```

That is exactly the S-25.3.2 → O-25.4.4 hop the documentation prescribes. It
only stays dormant because `deploy_mgmt.yml` guards the include with
`when: docker_tag is defined` (among others), and `docker_tag` is undefined by
default. Note the check does not use `docker_tag`'s *value* — the prefix
comparison reads the package's own `versions.api` — so passing it at all is
enough to trigger the failure. Written up internally as PKG-003.

**So: never pass `-e docker_tag=...` on an S→O upgrade.** This playbook does
not. If you ever need to, add `-e disable_version_check=true` alongside it.

---

## Troubleshooting

**`failed=N` in the PLAY RECAP** — revert to your snapshot and open a Support
ticket, attaching the `.err` and `.log` files from `/var/log/s1/migrations`.
The playbook also writes a full log to `~/<VERSION>/upgrade-<timestamp>.log`.

**`FAILED - RETRYING: Set backward compatibility`** — comment out the
`Set mongo backward compatibility` block in the staged `deploy_mgmt.yml` and
re-run.

**`NotNullViolation` on `bulk_tasks.description`**

```bash
sudo docker exec -it postgres psql sentinellabs
```
```sql
delete from bulk_tasks bt where not exists (select 1 from single_tasks st where bt.id=st.parent_task_id);
```

**`rmtree failed: Device or resource busy: '/postgres'`** — `sudo umount /postgres/`
(or `/mongo/`), then re-run.

**Active Directory conflict** (`FAILED! => {"changed": false, ... "status": 1}`)

```sql
delete from active_directory_role_mappings;
```

**`Error starting container`** — `systemctl restart docker`.

**Not enough space on `/`** — redirect the build directory:
`-e remote_build_path=/mongo/tmp_mgmt_deploy`.

**Deep Visibility auth fails after upgrade** — open a Support ticket for your
Management ID, then:

```bash
sudo sentinelmgmtctl change_settings --section cloud --key mgmt_id --value <mgmt_id>
```

**Private IPs rejected as SMTP servers** (S-25.1.4+) — set
`-e s1_smtp_whitelisted_host_ips="10.1.1.1,10.1.9.1"`.

---

## Upgrading the OS as well

Do it in this order:

1. Upgrade the OS to a version supported by **both** the old and new platform
   releases
2. Upgrade the platform
3. Optionally upgrade the OS again, to a version supported by the new release

Preflight fails early if the OS major version has no entry in the runtime
matrix for the target's release prefix.

---

## Status

Built from SentinelOne's "Upgrading to the latest self-hosted (on-premises)
Singularity Platform version" article and the vendor logic in
`roles/common/check_upgraded_version.yml`,
`roles/common/validate_upgrade_version.yml` and `roles/db/pg_upgrade/`.

**The full chain has been run on RHEL 8.10 and RHEL 9.7**, each hop reporting
`failed=0` with the console serving HTTPS throughout:

| Hop | RHEL 8.10 | RHEL 9.7 | Notes |
|---|---|---|---|
| S-25.3.2 → O-25.4.4 | `ok=320 changed=136 failed=0` | `ok=75 changed=10 failed=0` | S→O prefix transition; MongoDB 4.4 → 5.0 |
| O-25.4.4 → O-25.4.6 | `ok=308 changed=127 failed=0` | `ok=76 changed=10 failed=0` | 26 → 21 containers (deprecated ones removed) |
| O-25.4.6 → O-26.1.1 | `ok=308 changed=126 failed=0` | `ok=76 changed=10 failed=0` | Final state: 21 containers, none down |

(The RHEL 8.10 figures are the vendor playbook's recap; the RHEL 9.7 figures are
this playbook's. Both reported `failed=0` at each layer.)

Postgres tracked 15.4 → 15.16 → 15.18 on both platforms **without**
`upgrade_postgres` ever being set — minor bumps ride along with normal
container replacement.

On RHEL 9.7, `mount_dir` detection was confirmed to hold: after all three hops
`/data/uploads` and `/data/s1` were still in place, with `docker inspect api`
showing `/data/uploads->/uploads`, i.e. the source path unchanged. No `/uploads`
directory was created on the host.

Chain validation was also tested directly against a legal hop, a skipped
version, an unknown source and an already-latest console.

### RHEL 9.7 specifics

The vendor changes where `ansible-playbook` lives **partway through the chain**
on RHEL 9: O-25.4.4 uses `/usr/local/bin`, O-25.4.6 switches to a venv under
`/opt/sentinelone/installer`. The runtime is therefore discovered from the host
after bootstrap rather than read from a table — see the README. Note the vendor
upgrade article's own O-25.4.4 command prescribes the venv interpreter path,
which that release never creates; recorded internally as DOC-004.

**Not yet exercised:** the Postgres 11 → 15 migration (`upgrade_postgres` /
`dump_dir`), since every console tested was already on 15; and RHEL 10 with
`python3.12`.
