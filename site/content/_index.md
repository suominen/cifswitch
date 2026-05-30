---
title: "CIFSwitch — CIFS cifs.spnego key-origin LPE tracking"
description: "Linux kernel CIFS cifs.spnego key-description origin LPE, via the rootful cifs.upcall helper — distro patch status tracker"
layout: "single"
date: 2026-05-27
lastmod: 2026-05-30
cover:
  image: "cifswitch-tracker.png"
  alt: "CIFSwitch — CIFS cifs.spnego key-origin LPE tracker"
  hiddenInSingle: true
---

*CIFSwitch is a Linux local privilege escalation: the kernel CIFS client
never validated that a `cifs.spnego` key description originated from
kernel code, so an unprivileged user can forge one and steer the rootful
`cifs.upcall` helper into loading an attacker-controlled NSS module.  The
kernel fix (commit [`3da1fdf4efbc`][fix-commit]) is in mainline; distro
adoption is being tracked below.  No CVE has been assigned yet — this
tracker uses the placeholder `CVE-2026-XXXXX`.*

## Summary

| Field | Detail |
|---|---|
| CVE ID | Not yet assigned — placeholder `CVE-2026-XXXXX` |
| Alias | CIFSwitch |
| Component | Kernel: `fs/smb/client/cifs_spnego.c` (pre-6.7 path: `fs/cifs/cifs_spnego.c`) · Userspace: `cifs.upcall` from cifs-utils ≥ 6.14 |
| Type | Local Privilege Escalation (LPE) — forged `cifs.spnego` key → rootful upcall → attacker NSS-module load |
| CWE | [CWE-269][cwe-269] Improper Privilege Management · [CWE-284][cwe-284] Improper Access Control |
| CVSS | not yet scored |
| Discoverer | Asim Viladi Oglu Manizada |
| Public disclosure | 2026-05-27 — [heyitsas.im/posts/cifswitch][writeup] |
| Public PoC | [manizada/CIFSwitch][poc] |
| KEV listed | not yet |
| EPSS | n/a until a CVE is assigned |

## How the chain works

The kernel CIFS client registers a `cifs.spnego` key type, used to ask
userspace to perform SPNEGO/Kerberos negotiation for a mount.  The
request is serviced by `cifs.upcall` (shipped by **cifs-utils** and run
as **root** via `request-key`).  Before the fix, the key type carried no
`.vet_description` hook, so the kernel never checked that a `cifs.spnego`
key description it was asked to act on actually *originated from kernel
CIFS code*.

An unprivileged user can therefore request a `cifs.spnego` key with an
attacker-chosen description.  `request-key` runs the root `cifs.upcall`
helper, which parses fields out of that attacker-controlled description.
On **cifs-utils ≥ 6.14** the helper supports switching into the caller's
namespaces to resolve names; the attacker points it at a mount namespace
they control, containing a fake `/etc/nsswitch.conf` and a malicious
`libnss_*.so.2`.  The root helper then loads the attacker's NSS module —
arbitrary code execution as root.

The kernel fix, commit [`3da1fdf4efbc`][fix-commit], adds a
`.vet_description = cifs_spnego_key_vet_description` hook that rejects
descriptions not produced by the kernel.  That single change shuts the
door regardless of the installed cifs-utils version, so **the kernel
patch is the canonical fix** this tracker keys on.

> :information_source: Exploitation requires **all** of: a kernel without
> the `vet_description` fix; cifs-utils ≥ 6.14 (the namespace-switching
> `cifs.upcall`); unprivileged user namespaces enabled; and an LSM policy
> (SELinux/AppArmor) that does not block the upcall path.  This tracker
> records two version axes per release — **kernel** (the fix) and
> **cifs-utils** (the reachability gate) — with per-distro notes on the
> userns/LSM posture where it changes the verdict.

## Vulnerable commit range

| Commit | Role | Description |
|---|---|---|
| *(git blame: ~2007)* | Introduced | The `cifs.spnego` key type has lacked an origin check since SPNEGO upcall support was added (~2007).  No single introducing commit is cited by the disclosure. |
| [`3da1fdf4efbc`][fix-commit] | Fix | Adds the `.vet_description = cifs_spnego_key_vet_description` hook, rejecting forged key descriptions; Linus mainline. |

The effective lifetime of the bug is therefore roughly 19 years
(2007–2026).

## Upstream fixed versions

The fix is present in Linus mainline.  Per-branch stable backport status
is being verified against the stable tree and is marked unverified until
confirmed.

| Branch | Status | Current | Notes |
|---|---|---|---|
| Linus mainline | :white_check_mark: Carries `3da1fdf4efbc` | — | first released mainline tag pending verification |
| 6.18.x | :grey_question: Backport status unverified | — | |
| 6.12.x | :grey_question: Backport status unverified | — | LTS 2028-12 |
| 6.6.x  | :grey_question: Backport status unverified | — | LTS 2026-12 |
| 6.1.x  | :grey_question: Backport status unverified | — | LTS 2026-12 |
| 5.15.x | :grey_question: Backport status unverified | — | LTS 2026-12 |
| 5.10.x | :grey_question: Backport status unverified | — | LTS 2026-12 |

When verifying a kernel tree directly, the file is
`fs/smb/client/cifs_spnego.c` on 6.7 and later, and
`fs/cifs/cifs_spnego.c` on earlier kernels (the `fs/cifs` → `fs/smb/client`
move landed in 6.7).

## Distribution status

The deciding facts per release are whether the **kernel** carries the
`vet_description` fix and whether **cifs-utils** is ≥ 6.14 (the
namespace-switching `cifs.upcall`), tempered by whether unprivileged user
namespaces and the LSM policy actually allow the upcall path.  *Fixed
since* records the date the kernel fix first lands in that release.

The rows below track a focused set of distributions; current per-distro
kernel and cifs-utils package versions, and any shipped fixes, are being
verified — unconfirmed cells are marked :grey_question:.  Other systems
the disclosure writeup (2026-05-27) reported vulnerable — Ubuntu,
AlmaLinux, Oracle Linux, openSUSE / SLES, Fedora, Arch — are not tracked
here and appear only as references where relevant.

| Distribution | Release | Kernel | cifs-utils | Fixed since | Status |
|---|---|---|---|---|---|
| Debian | sid (unstable) | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| Debian | forky (testing) | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| Debian | 13 (trixie) | unpatched | :grey_question: | — | :x: Vulnerable — no fixed kernel yet |
| Debian | 12 (bookworm) | unpatched | :grey_question: | — | :x: Vulnerable — no fixed kernel yet |
| Debian | 11 (bullseye, LTS) | unpatched | :grey_question: | — | :x: Vulnerable — no fixed kernel yet |
| Proxmox VE | 9 | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| Proxmox VE | 8 | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| NixOS | `nixos-unstable` | :grey_question: | :grey_question: | — | :grey_question: Unverified — see notes |
| NixOS | `nixos-unstable-small` | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| NixOS | `nixos-25.11` | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| NixOS | `nixos-25.11-small` | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| Rocky Linux | 10 | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| Rocky Linux | 9 | unpatched | :grey_question: | — | :x: Vulnerable — SELinux may constrain the upcall; see notes |
| Rocky Linux | 8 | :grey_question: | :grey_question: | — | :grey_question: Unverified |
| Amazon Linux | 2023 | unpatched | :grey_question: | — | :x: Vulnerable |
| Amazon Linux | 2 | :grey_question: | :grey_question: | — | :grey_question: Unverified |

### NixOS

Each NixOS release line — unstable and 25.11 — has a default channel and
a faster, server-oriented `-small` subset that usually leads it by a
point release; all four are tracked.  NixOS enables unprivileged user
namespaces by default, but `cifs.upcall` (cifs-utils) is only present and
wired as the `request-key` handler on hosts actually configured for
CIFS/SMB mounts.  A row's kernel pin and cifs-utils version are determined
per channel from the nixpkgs tree; `tracker.security.nixos.org` is a
secondary signal only.

### Rocky Linux / RHEL family

The table tracks **Rocky Linux** (10 / 9 / 8).  **RHEL** and **AlmaLinux**
are not tracked as separate rows — Rocky rebuilds RHEL, and AlmaLinux
usually ships the same fix ahead of Rocky in its `testing` repo, so both
serve as references / leading indicators for the Rocky kernel (`6.12.x`
on 10, `5.14.x` on 9, `4.18.x` on 8).

On the EL family `cifs` is a loadable module and SELinux is enforcing by
default, which may constrain `cifs.upcall`'s ability to load an arbitrary
NSS module — confirm against the actual policy before treating a release
as not exploitable.  The shipped `cifs-utils` version is the other gate:
older EL releases may predate the 6.14 namespace-switch upcall.  Once a
CVE is assigned, RLSAs (and the matching RHSA / ALSA references) carry the
fixed kernel.

## Detection

**Is the kernel fixed?**  The fix adds the `.vet_description` hook in
`fs/smb/client/cifs_spnego.c` (pre-6.7: `fs/cifs/cifs_spnego.c`).  In
practice, compare the running kernel against the *Upstream fixed
versions* table and your distro's row above rather than reading source.

**Is `cifs` present / loadable?**

```bash
lsmod | grep '^cifs '
modinfo cifs >/dev/null 2>&1 && echo "cifs available" || echo "cifs not present"
```

**Which cifs-utils is installed?**  (≥ 6.14 is the reachability gate.)

```bash
mount.cifs -V                 # e.g. "mount.cifs version: 7.0"
dpkg -l cifs-utils 2>/dev/null || rpm -q cifs-utils 2>/dev/null
```

**Are unprivileged user namespaces allowed?**

```bash
sysctl kernel.unprivileged_userns_clone 2>/dev/null              # Debian/Ubuntu: 1 = allowed
sysctl kernel.apparmor_restrict_unprivileged_userns 2>/dev/null  # Ubuntu 24.04+: 1 = restricted
cat /proc/sys/user/max_user_namespaces                           # 0 = disabled
```

**Is `cifs.upcall` wired as the key handler?**

```bash
grep -rs cifs.spnego /etc/request-key.conf /etc/request-key.d/ 2>/dev/null
```

## Public PoC

The upstream PoC is in [manizada/CIFSwitch][poc].  Do **not** run it on a
system you are not authorised to test.

## Mitigation

The real fix is the kernel patch.  Until a fixed kernel is installed, the
most reliable interim mitigation is to remove the attacker's ability to
build the fake namespace by **disabling unprivileged user namespaces**:

```bash
# Debian/Ubuntu
sudo sysctl -w kernel.unprivileged_userns_clone=0
```
```bash
# Generic (all distros)
sudo sysctl -w user.max_user_namespaces=0
```

Persist it via a drop-in in `/etc/sysctl.d/`.  This breaks workloads that
legitimately rely on unprivileged user namespaces (rootless containers,
some sandboxes, `bwrap`-based apps).

On Ubuntu 24.04+ the AppArmor restriction is equivalent and on by
default:

```bash
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=1
```

If CIFS/SMB client mounts are not needed, removing or blacklisting the
`cifs` module (and not installing cifs-utils) removes the upcall surface
entirely.  **Downgrading cifs-utils below 6.14 is not a reliable
mitigation** — the disclosure notes some older backported versions are
also affected, and it sacrifices functionality without closing the
kernel-side hole.

## Risk notes

- **Multi-user and shared hosts, CI/CD runners:** any unprivileged local
  user can attempt the escalation where the preconditions hold.  Self-
  hosted CI runners executing untrusted code are directly in scope.
- **Containers:** on shared-kernel hosts that permit unprivileged user
  namespaces, this is a host-root primitive from inside a container.
- **CIFS clients/servers:** hosts that mount CIFS/SMB shares have
  cifs-utils installed and `cifs.upcall` wired — exactly the reachable
  configuration.
- **Hardened by default:** a vulnerable kernel is not exploitable via
  this chain where unprivileged user namespaces are disabled or the LSM
  policy blocks the upcall (e.g. Ubuntu 24.04 AppArmor default).  Treat
  those as mitigated, not fixed — the kernel hole remains until patched.

## Verification log

*Last verified 2026-05-30.*

### Upstream

- The kernel fix is Linus mainline commit `3da1fdf4efbc`, adding the
  `.vet_description = cifs_spnego_key_vet_description` hook in
  `fs/smb/client/cifs_spnego.c`.  The first released mainline tag and the
  per-branch stable backports have **not yet** been verified against the
  local stable clone (`~/src/linux/stable`) — every stable-branch row
  remains `:grey_question:` pending that check (look for backports of
  `3da1fdf4efbc` on `fs/smb/client/cifs_spnego.c`, or the legacy
  `fs/cifs/cifs_spnego.c` path on pre-6.7 branches).

### Distributions

- Tracked rows: Debian sid / forky / 13 / 12 / 11, Proxmox VE 9/8,
  NixOS (nixos-unstable[-small], nixos-25.11[-small]), Rocky Linux
  10/9/8, Amazon Linux 2023 and 2.  The disclosure
  writeup (2026-05-27) also reported Ubuntu 22.04/20.04/18.04, AlmaLinux
  9.7, Oracle Linux 9/8, CentOS Stream 9, SLES 15 SP7, openSUSE Leap
  15.6, Linux Mint 22.3/21.3, and Kali 2021.4+ as vulnerable; those are
  used as references only, not tracked as rows.
- Current per-distro kernel and cifs-utils package versions, and any
  shipped fixes, are pending verification against distro package metadata
  and advisories.  NixOS rows are pending the local nixpkgs-clone lookup.
- No CVE-keyed feeds (NVD, EPSS, Red Hat JSON, CISA KEV) resolve yet —
  no CVE has been assigned.

## References

| Source | URL |
|---|---|
| Disclosure writeup | <https://heyitsas.im/posts/cifswitch/> |
| Public PoC | <https://github.com/manizada/CIFSwitch> |
| Kernel fix commit | <https://github.com/torvalds/linux/commit/3da1fdf4efbc490041eb4f836bf596201203f8f2> |
| cifs-utils upstream | <https://git.samba.org/?p=cifs-utils.git;a=summary> |

[writeup]: https://heyitsas.im/posts/cifswitch/
[poc]: https://github.com/manizada/CIFSwitch
[fix-commit]: https://github.com/torvalds/linux/commit/3da1fdf4efbc490041eb4f836bf596201203f8f2
[cwe-269]: https://cwe.mitre.org/data/definitions/269.html
[cwe-284]: https://cwe.mitre.org/data/definitions/284.html
