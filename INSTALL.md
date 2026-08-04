# INSTALL — SentinelOne Management Console

Clean installation on RHEL 8.10 / 9.7. Budget **40–60 minutes**; the vendor
deploy alone takes ~20.

---

## Where files go

Two different machines are involved. Getting this wrong is the most common
setup mistake, so it is spelled out first.

```
CONTROL NODE (your Mac / jump host)        TARGET SERVER (RHEL 8.10 / 9.7)
├── s1-opmc-ansible/                       ├── /var/tmp/
│   ├── certificates/         ← certs      │   └── <package>.tar.gz   ← the TGZ
│   │   ├── certificate.pem                ├── /home/<ssh-user>/
│   │   └── certificate.key                │   └── S-25.3.2/         ← unpacked here
│   ├── installers/           ← local copy └── /data/                ← console data
│   │   └── <package>.tar.gz     (optional)
│   └── checksums.yml         ← SHA256 sums
```

### 1. The deployment TGZ → the **target server**

The package is ~5 GB. Ansible's `copy` module is far too slow for that, so the
playbook **does not transfer it** — it expects the file to already be on the
target and fails preflight with a clear message if it is missing.

Copy it out of band:

```bash
scp installers/SentinelOne_MGMT_S_25_3_2_096.tar.gz <user>@console.example.com:/var/tmp/
```

Then point the playbook at that path in `group_vars/all/main.yml`:

```yaml
s1_package_path: /var/tmp/SentinelOne_MGMT_S_25_3_2_096.tar.gz
```

The local `installers/` directory is only a staging area for your own copies.
It is **gitignored** — never commit these files.

**Integrity is verified before unpacking.** The vendor's download service is
unreliable, and a truncated tarball otherwise fails deep inside the deploy —
after the host has already been modified. Staging checks the package against
[checksums.yml](checksums.yml) first:

1. **Size** is compared first. It is instant and catches the common truncated
   download without reading 5 GB.
2. **SHA256** is computed only if the size matches (~1 minute for a 5 GB file).

A mismatch aborts before anything is touched. Comparison is case-insensitive,
so vendor sums can be pasted exactly as published.

A package with no entry in `checksums.yml` is refused: its RPMs are installed
with gpgcheck disabled and its playbooks run as root, so an unverified package
is a supply-chain risk. Add an entry (see below), or — accepting that risk —
let it through with a warning:

```bash
ansible-playbook install.yml -e s1_require_checksum=false
```

To add an entry:

```bash
shasum -a 256 installers/<package>.tar.gz     # macOS   (sha256sum on Linux)
stat -f%z installers/<package>.tar.gz         # macOS   (stat -c%s on Linux)
```

```yaml
packages:
  <package>.tar.gz:
    version: "O-25.4.4"
    source: vendor          # 'vendor' = published by SentinelOne, 'local' = computed here
    size: 0                 # optional
    sha256: "..."
```

Prefer vendor-published sums where you can get them: a locally computed hash
proves the file has not changed since you hashed it, but cannot prove the copy
was good to begin with.

Re-running skips the hash when the version directory is already unpacked —
delete it to force a fresh verified unpack.

**Where it gets unpacked.** The playbook extracts into a version-named
directory in the home of the SSH user, giving `~/S-25.3.2/`. The tarballs
extract *flat* (no top-level directory), so this produces
`~/S-25.3.2/docker_ansible/...` exactly as the vendor docs assume. Old version
directories are left in place to make upgrades easy — **pruning them is your
responsibility**. Budget ~20 GB of free space in `/home` per version retained.

Override the parent directory if `/home` is too small:

```yaml
s1_install_base: /opt/sentinelone
```

### 2. Certificates → the **control node**

Place your **PEM-format** certificate and its private key in `certificates/`:

```
certificates/certificate.pem     ← the certificate (PEM)
certificates/certificate.key     ← the private key (PEM, NO passphrase)
```

The playbook copies both to `/etc/sentinelone/certs` on the target (a
root-only `0700` directory; `0600` on the key, `0644` on the certificate) and
writes those paths into `sentinel_pem_file` and `sentinel_key_file`. Override
the staging directory with `s1_cert_dir`.

Both files must be **PEM encoded** — base64 text between header lines, not DER
or PKCS#12:

```
-----BEGIN CERTIFICATE-----
MIIDdzCCAl+gAwIBAgIEAgAAuTANBgkqhkiG9w0BAQ...
-----END CERTIFICATE-----
```

**The key must not have a passphrase.** The console cannot prompt for one, and
preflight rejects an encrypted key rather than letting the deploy fail 20
minutes in. To strip a passphrase:

```bash
openssl rsa -in encrypted.key -out certificates/certificate.key
```

Converting from PKCS#12 (`.pfx`/`.p12`):

```bash
openssl pkcs12 -in cert.pfx -clcerts -nokeys -out certificates/certificate.pem
openssl pkcs12 -in cert.pfx -nocerts -nodes -out certificates/certificate.key
```

If the certificate needs an intermediate chain, concatenate it into the PEM —
server certificate first, then intermediates.

Use different filenames by setting `s1_key_src` / `s1_pem_src`. `*.pem`,
`*.key` and `*.crt` are gitignored, so real certificates will not be committed
by accident.

---

## Before you start

On the **target server**:

- RHEL 8.x or 9.x, x86_64
- SSH access with passwordless `sudo`
- **A CPU with AVX.** O-prefix packages ship MongoDB 5.0+, which requires it.
  Without AVX the mongo container crash-loops and the deploy fails ~7 minutes
  in at "Wait for mongo to be available".
  ```bash
  grep -qw avx /proc/cpuinfo && echo OK || echo "MISSING — fix the CPU model"
  ```
  On **Proxmox** the default `kvm64` CPU model masks AVX. Set
  Hardware → Processors → Type to `host` (or `x86-64-v3`). Note that reverting
  a VM snapshot also reverts the CPU setting, silently undoing this.
- **Memory.** The console runs 20+ containers. Check the vendor prerequisites
  article for the official figure; the playbook defaults to a 16 GB hard floor
  and warns below 32 GB, with 8 vCPU recommended. Under-provisioned hosts do
  not fail cleanly — the deploy runs for 20+ minutes, dies with
  `Cannot allocate memory`, and can leave the box unable to fork `sshd`.
- **UID and GID 847 must be free**, and no existing user or group named `s1`.
  These values are hard-coded in the installer and cannot be changed.
  ```bash
  getent passwd 847; getent group 847; getent passwd s1; getent group s1
  ```
  All four should return nothing.
- `/data` exists (or is creatable) with room for the console's databases
- **iptables must not be disabled** — Docker manages NAT through it
- `/tmp` must not be mounted `noexec` — it breaks the vendor's pip and Docker
  installs

Preflight checks every one of these before touching the host, so
`--tags preflight` is the fastest way to confirm them all at once.

You do **not** need a Red Hat subscription, an enabled DVD repo, or internet
access. The vendor installer ships its own yum repository inside the package
and installs everything with `--disablerepo=*`.

---

## Procedure

### 1. Install control-node dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Set the target

Both site-specific files are gitignored and shipped as `.example` templates, so
your hostnames and usernames are never committed:

```bash
cp inventory.ini.example inventory.ini
cp group_vars/all/main.yml.example group_vars/all/main.yml
```

`inventory.ini`:

```ini
[s1_mgmt]
console.example.com

[s1_mgmt:vars]
ansible_user=your-ssh-user
ansible_become=true
```

`ansible_user` must be the account you SSH in as — it determines which home
directory the package is unpacked into.

Do **not** set `ansible_python_interpreter`. The first play probes the target
and installs `python3.11` if needed — RHEL 8.10 ships only Python 3.6, which
modern ansible-core cannot manage — then pins whatever it found. On RHEL 9 this
is a no-op: it ships Python 3.9 and the probe succeeds immediately.

This is the interpreter Ansible uses to *manage* the target. It is unrelated to
the one passed to the vendor's own deploy, which is discovered after bootstrap
(see [README](README.md#ansible-runtime--discovered-not-tabulated)).

### 3. Configure

Edit `group_vars/all/main.yml`:

```yaml
s1_package_path: /var/tmp/SentinelOne_MGMT_S_25_3_2_096.tar.gz
s1_mount_dir: /data
s1_accessible_url: https://console.example.com
s1_admin_fullname: CHANGEME
s1_admin_email: changeme@example.com
```

For an **air-gapped** install leave these empty — the playbook then skips them
and the vendor's blank defaults are kept:

```yaml
s1_cloud_address: ""
s1_cloud_token: ""
s1_mgmt_id: ""
s1_account_name: ""
s1_salesforce_id: ""
```

### 4. Set the admin password

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
$EDITOR group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml
```

It must be `group_vars/all/vault.yml`. A file at `group_vars/vault.yml` would
map to a host group named `vault`, which does not exist, so it would never be
loaded — and the missing password would only surface at preflight.

Password rules: at least 10 characters, 3+ of uppercase / lowercase / digits /
symbols, no whitespace, and no `"` or `'`. The vendor playbook also rejects
`sentinel`, `sentinel1` and `sentinel1!`.

### 5. Preflight only

Check everything without touching the host:

```bash
ansible-playbook install.yml --tags preflight --ask-vault-pass
```

### 6. Install

```bash
ansible-playbook install.yml --ask-vault-pass
```

Take a VM snapshot first if you can — a failed deploy is far easier to roll
back than to repair.

### 7. Confirm

The final task prints container status and the console's HTTP response. Then
log in and check **Help > About** for the version (SHIFT+F5 forces a hard
refresh).

---

## Re-running

Safe to re-run. The deploy is gated on `/s1/conf/settings.ini`: if the console
is already installed it is skipped with a message. To deliberately redeploy
over an existing install:

```bash
ansible-playbook install.yml -e s1_force_redeploy=true --ask-vault-pass
```

---

## Troubleshooting

**`failed=N` in the vendor PLAY RECAP** — the full log is written to
`~/<VERSION>/deploy-<timestamp>.log`. For a support ticket, attach the `.err`
and `.log` files from `/var/log/s1/migrations`.

**`ERROR! [DEPRECATED]: ansible.builtin.include has been removed`** — a system
Ansible 2.16+ is being used instead of the vendor's pinned 2.15.x.
`deploy_mgmt.yml` still uses the removed `include:` directive. Check the
`binary :` line in the "Show the resolved Ansible runtime" output — it should
point at the vendor's install, not a system one. Pin it with `s1_playbook_bin`
if discovery picked the wrong thing.

**`cannot import the docker Python package`** — bootstrap resolved an
interpreter that the vendor's `offline_installation.sh` did not install `docker`
into. The message names the interpreter it tried; set `s1_python_interpreter` to
the one that has it. This check exists because the alternative is failing 20+
minutes into the deploy.

**`Error starting container`** — `systemctl restart docker`.

**`rmtree failed: Device or resource busy: '/postgres'`** — only happens when
`mount_dir` is empty. Confirm `s1_mount_dir` is set, then `umount /postgres`.

**Podman conflicts** — the playbook removes `podman`, `podman-manpages` and
`buildah` before installing docker-ce. If Docker still misbehaves,
`systemctl restart docker`.

**`KERNEL UPDATE INSTALLED - REBOOT REQUIRED`** — reboot and re-run. The
playbook detects this and stops with that instruction.

**Ansible reports Python 3.6 on RHEL 8 with an S-prefix package** — expected.
The bundled repo ships `python36`, not `python38`, and the vendor docs accept
3.6 as the fallback. Only set `s1_install_python38: true` if you have RHSM or a
DVD repo enabled and specifically need 3.8.
