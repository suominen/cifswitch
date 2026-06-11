---
title: "CVE-2026-46243 — CIFSwitch tracking"
description: "Linux kernel CIFS cifs.spnego key-description origin LPE, via the rootful cifs.upcall helper — distro patch status tracker"
layout: "single"
date: 2026-05-27
lastmod: 2026-06-11
cover:
  image: "cifswitch-tracker.png"
  alt: "CVE-2026-46243 — CIFSwitch CIFS cifs.spnego key-origin LPE tracker"
  hiddenInSingle: true
---

*CIFSwitch is a Linux local privilege escalation: the kernel CIFS client
never validated that a `cifs.spnego` key description originated from
kernel code, so an unprivileged user can forge one and steer the rootful
`cifs.upcall` helper into loading an attacker-controlled NSS module.  The
kernel fix (commit [`3da1fdf4efbc`][fix-commit]) is in mainline; distro
adoption is being tracked below.  The Linux kernel CNA assigned
[`CVE-2026-46243`][nvd] on 2026-06-01.*

## Summary

| Field | Detail |
|---|---|
| CVE ID | [CVE-2026-46243][nvd] (assigned 2026-06-01 by the Linux kernel CNA) |
| Alias | CIFSwitch |
| Component | Kernel: `fs/smb/client/cifs_spnego.c` (pre-6.7 path: `fs/cifs/cifs_spnego.c`) · Userspace: `cifs.upcall` from cifs-utils ≥ 6.14 |
| Type | Local Privilege Escalation (LPE) — forged `cifs.spnego` key → rootful upcall → attacker NSS-module load |
| CWE | [CWE-269][cwe-269] Improper Privilege Management · [CWE-284][cwe-284] Improper Access Control (NVD: [CWE-20][cwe-20] Improper Input Validation) |
| CVSS | 7.1 (High) — `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N` (kernel CNA) · 7.8 (High) — `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` (CISA-ADP) |
| Discoverer | Asim Viladi Oglu Manizada |
| Public disclosure | 2026-05-27 — [heyitsas.im/posts/cifswitch][writeup] |
| Public PoC | [manizada/CIFSwitch][poc] |
| KEV listed | Not listed |
| EPSS | 0.00023 (7th percentile), scored 2026-06-11 |

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
| `f1d662a7d5e5` | Introduced | *[CIFS] Add upcall files for cifs to use spnego/kerberos* (2007-11-05) — added the `cifs.spnego` key type without an origin check; first shipped in v2.6.24.  Identified as the introducing commit by the kernel CNA's CVE record. |
| [`3da1fdf4efbc`][fix-commit] | Fix | Adds the `.vet_description = cifs_spnego_key_vet_description` hook, rejecting forged key descriptions; Linus mainline. |

The effective lifetime of the bug is therefore roughly 19 years
(2007–2026), spanning v2.6.24 (released 2008-01-24) through the fix.

## Upstream fixed versions

The fix is present in Linus mainline (merged post-v7.0) and was
backported to all tracked stable branches on 2026-06-01.  The first
fixed point release per branch is noted in the table.  Distro adoption
is in progress; no kernel advisory has referenced the fix yet.

| Branch | Status | Current | Notes |
|---|---|---|---|
| Linus mainline | :white_check_mark: Carries `3da1fdf4efbc` | — | merged post-v7.0; will appear in 7.1 on release |
| 7.0.x | :white_check_mark: Backported | 7.0.12 | first fixed: 7.0.11 |
| 6.18.x | :white_check_mark: Backported | 6.18.35 | first fixed: 6.18.34 |
| 6.12.x | :white_check_mark: Backported | 6.12.93 | LTS 2028-12; first fixed: 6.12.92 |
| 6.6.x  | :white_check_mark: Backported | 6.6.142 | LTS 2026-12; first fixed: 6.6.142 |
| 6.1.x  | :white_check_mark: Backported | 6.1.175 | LTS 2026-12; first fixed: 6.1.175 |
| 5.15.x | :white_check_mark: Backported | 5.15.209 | LTS 2026-12; first fixed: 5.15.209 |
| 5.10.x | :white_check_mark: Backported | 5.10.258 | LTS 2026-12; first fixed: 5.10.258 |

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

The rows below track a focused set of distributions with their current
per-distro kernel and cifs-utils package versions and any shipped fixes.
Other systems the disclosure writeup (2026-05-27) reported vulnerable —
Ubuntu, AlmaLinux, Oracle Linux, openSUSE / SLES, Fedora, Arch — are not
tracked here and appear only as references where relevant.

| Distribution | Release | Kernel | cifs-utils | Fixed since | Status |
|---|---|---|---|---|---|
| Debian | sid (unstable) | 7.0.12-1 | 7.4 | 2026-06-10 | :white_check_mark: Fixed |
| Debian | forky (testing) | 7.0.10-1 | 7.4 | 2026-05-28 | :white_check_mark: Fixed |
| Debian | 13 (trixie) | 6.12.90-2 | 7.4 | 2026-05-28 | :white_check_mark: Fixed |
| Debian | 12 (bookworm) | 6.1.174-1 | 7.0 | 2026-05-28 | :white_check_mark: Fixed |
| Debian | 11 (bullseye, LTS) | 5.10.257-1 | 6.11 | 2026-05-29 | :white_check_mark: Fixed |
| Proxmox VE | 9 | 7.0.6-2-pve | 7.4 | — | :x: Vulnerable — no fixed kernel yet |
| Proxmox VE | 8 | 6.8.12-28-pve | 7.0 | — | :x: Vulnerable — no fixed kernel yet |
| NixOS | Unstable | 7.0.11 | 7.5 | 2026-06-08 | :white_check_mark: Fixed |
| NixOS | Unstable (small) | 7.0.11 | 7.5 | 2026-06-02 | :white_check_mark: Fixed |
| NixOS | 25.11 | 7.0.11 | 7.4 | 2026-06-06 | :white_check_mark: Fixed |
| NixOS | 25.11 (small) | 7.0.11 | 7.4 | 2026-06-03 | :white_check_mark: Fixed |
| Rocky Linux | 10 | 6.12.0-211.16.1.el10_2 | 7.5 | — | :x: Vulnerable — see Rocky notes |
| Rocky Linux | 9 | 5.14.0-687.12.1.el9_8 | 7.5 | — | :x: Vulnerable — see Rocky notes |
| Rocky Linux | 8 | 4.18.0-553.126.1.el8_10 | 7.0 | — | :x: Vulnerable — see Rocky notes |
| Amazon Linux | 2023 | 6.1.172-216.329.amzn2023 | 7.5 | — | :x: Vulnerable |
| Amazon Linux | 2 | 4.14.355-282.729.amzn2 | 6.2 | — | :x: Vulnerable — cifs-utils 6.2 &lt; 6.14; primary exploit path absent, reduced exposure |
{.distros}

### NixOS

NixOS enables unprivileged user namespaces by default, but `cifs.upcall`
(cifs-utils) is only present and wired as the `request-key` handler on
hosts actually configured for CIFS/SMB mounts — so the full chain applies
only to those hosts.

### Rocky Linux / RHEL family

On the EL family `cifs` is a loadable module and SELinux is enforcing by
default, which may constrain `cifs.upcall`'s ability to load an arbitrary
NSS module — confirm against the actual policy before treating a release
as not exploitable.  The shipped `cifs-utils` version is the other gate:
older EL releases may predate the 6.14 namespace-switch upcall.  Watch for
RLSAs (and the matching RHSA / ALSA references) citing CVE-2026-46243 to
carry the fixed kernel.

## Detection

**Is the kernel fixed?**  The fix adds the `.vet_description` hook in
`fs/smb/client/cifs_spnego.c` (pre-6.7: `fs/cifs/cifs_spnego.c`).  In
practice, compare the running kernel against the *Upstream fixed
versions* table and your distro's row above rather than reading source.

**Is `cifs` present / loadable?**  Whether it is loaded right now:

```bash
lsmod | grep '^cifs '
```

Whether the module is available to load at all:

```bash
modinfo cifs >/dev/null 2>&1 && echo "cifs available" || echo "cifs not present"
```

**Is `cifs` a loadable module or built in?**  This decides whether the
module-blocking mitigation can work.  Inspect the kernel config:

```bash
grep -E 'CONFIG_CIFS\b' /boot/config-$(uname -r)
```

Interpret the result:

- `CONFIG_CIFS=m` → loadable module — the modprobe block and `rmmod`
  work.  Every mainstream distro (Debian, Proxmox, NixOS, Rocky, Amazon)
  ships it this way.
- `CONFIG_CIFS=y` → built in — the module cannot be unloaded and the
  modprobe block will not help; disable unprivileged user namespaces
  instead until a patched kernel is installed.
- `# CONFIG_CIFS is not set` → not built — the CIFS client is absent, so
  this chain is not reachable.

Fallback if `/boot/config-*` is unreadable and `CONFIG_IKCONFIG_PROC=y`:

```bash
zgrep -E 'CONFIG_CIFS\b' /proc/config.gz
```

**Which cifs-utils is installed?**  (≥ 6.14 is the reachability gate.)
The helper reports its own version, e.g. `mount.cifs version: 7.0`:

```bash
mount.cifs -V
```

Or query the package manager:

```bash
dpkg -l cifs-utils 2>/dev/null || rpm -q cifs-utils 2>/dev/null
```

**Are unprivileged user namespaces allowed?**  On Debian/Ubuntu, `1`
means allowed:

```bash
sysctl kernel.unprivileged_userns_clone 2>/dev/null
```

On Ubuntu 24.04+, `1` means the AppArmor userns restriction is active:

```bash
sysctl kernel.apparmor_restrict_unprivileged_userns 2>/dev/null
```

Across all distros, `0` here means user namespaces are disabled:

```bash
cat /proc/sys/user/max_user_namespaces
```

**Is `cifs.upcall` wired as the key handler?**

```bash
grep -rs cifs.spnego /etc/request-key.conf /etc/request-key.d/ 2>/dev/null
```

## Public PoC

The upstream PoC is in [manizada/CIFSwitch][poc].  Do **not** run it on a
system you are not authorised to test.

## Mitigation

The real fix is the kernel patch.  Until a fixed kernel is installed, two
interim measures each break the chain on their own — disabling
unprivileged user namespaces (removes the attacker's ability to build the
fake namespace) and blocking the `cifs` module (removes the upcall
surface).

### Disable unprivileged user namespaces

On Debian/Ubuntu:

```bash
sudo sysctl -w kernel.unprivileged_userns_clone=0
```

Or generically, on any distro:

```bash
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

### Block the cifs module (when it is a loadable module)

If CIFS/SMB client mounts are not needed, blocking the `cifs` module
removes the upcall surface entirely.  This works only when `cifs` is a
loadable module (`CONFIG_CIFS=m`, the universal distro default — see
*Detection*); a built-in `cifs` cannot be unloaded or blocked this way.
A bare `blacklist cifs` only suppresses alias-based autoload — the module
still loads on an explicit `mount -t cifs`, so use an `install` override
to block it outright:

```bash
echo 'install cifs /bin/false' | sudo tee /etc/modprobe.d/cifswitch.conf
```

Then unload it if it is currently loaded (unmount any CIFS shares first):

```bash
sudo rmmod cifs
```

Verify it is gone and stays blocked:

```bash
lsmod | grep '^cifs ' && echo "STILL LOADED" || echo "Not loaded"
```

Not installing (or removing) cifs-utils drops the root `cifs.upcall`
helper as well.  **Downgrading cifs-utils below 6.14 is not a reliable
mitigation** — the disclosure notes some older backported versions are
also affected, and it sacrifices functionality without closing the
kernel-side hole.

### NixOS

NixOS manages `/etc/modprobe.d` declaratively — set the option below
rather than editing those files (they are regenerated on every rebuild),
then run `nixos-rebuild switch`.  It is an ordinary NixOS option: it
belongs in `configuration.nix`, or — with a flake — in any module
imported by the host's `nixosConfigurations.<host>` entry.

Block the `cifs` module — this text is appended to
`/etc/modprobe.d/nixos.conf`:

```nix
boot.extraModprobeConfig = ''
  install cifs /bin/false
'';
```

Disabling unprivileged user namespaces is the alternative lever:

```nix
boot.kernel.sysctl."user.max_user_namespaces" = 0;
```

Neither option unloads a module that is *already* loaded — reboot, or run
`rmmod cifs`, to clear a live one.

### Built-in cifs (`CONFIG_CIFS=y`)

If `cifs` is compiled in rather than modular, neither `rmmod` nor the
modprobe block help.  No mainstream distribution builds the CIFS client
in; if a custom kernel does, disable unprivileged user namespaces (above)
until a patched kernel is installed.

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

*Last verified 2026-06-11.*

### Upstream

- The kernel fix is Linus mainline commit `3da1fdf4efbc`, adding the
  `.vet_description = cifs_spnego_key_vet_description` hook in
  `fs/smb/client/cifs_spnego.c`.  The fix was merged into Linus mainline
  after v7.0 branched; it will first appear as a standalone release in v7.1.
- **CVE assigned**: the Linux kernel CNA assigned `CVE-2026-46243`
  (*smb: client: reject userspace cifs.spnego descriptions*) on
  2026-06-01.  Its record confirms the introducing commit `f1d662a7d5e5`
  (v2.6.24) and per-branch first-fixed versions matching the *Upstream
  fixed versions* table.
- **All stable branches now carry the backport**: it landed in all
  tracked branches on 2026-06-01.  First fixed point releases: 7.0.11,
  6.18.34, 6.12.92, 6.6.142, 6.1.175, 5.15.209, 5.10.258 (all 2026-06-01,
  confirmed per branch).  Current point releases per kernel.org
  finger_banner: 7.0.12, 6.18.35, 6.12.93, 6.6.142, 6.1.175, 5.15.209,
  5.10.258.

### Distributions

- Tracked rows: Debian sid / forky / 13 / 12 / 11, Proxmox VE 9/8,
  NixOS (nixos-unstable[-small], nixos-25.11[-small]), Rocky Linux
  10/9/8, Amazon Linux 2023 and 2.  The disclosure
  writeup (2026-05-27) also reported Ubuntu 22.04/20.04/18.04, AlmaLinux
  9.7, Oracle Linux 9/8, CentOS Stream 9, SLES 15 SP7, openSUSE Leap
  15.6, Linux Mint 22.3/21.3, and Kali 2021.4+ as vulnerable; those are
  used as references only, not tracked as rows.
- **Debian** (via the Debian security tracker; main-archive versions
  confirmed via madison): all five tracked releases now carry the kernel
  fix.  sid: 7.0.12-1 (already fixed; main archive).  forky: 7.0.10-1
  fixed — Debian cherry-picked `3da1fdf4efbc` into 7.0.10, predating
  the upstream first-fixed 7.0.11.  trixie: 6.12.90-2 via DSA 6305-1.
  bookworm: 6.1.174-1 via DSA 6306-1.  bullseye: 5.10.257-1 via
  DLA-4606-1.  All three DSA/DLA publications predate CVE assignment
  (2026-06-01); the packages were retroactively linked to CVE-2026-46243
  by the security tracker.  cifs-utils unchanged: sid/forky/trixie 7.4;
  bookworm 7.0; bullseye 6.11 (< 6.14 — primary exploit path was absent).
- **NixOS** (via the nixpkgs channel branches): all four tracked channels
  have now advanced to `linux_7_0` at 7.0.11 — the first fixed release;
  all four rows `:white_check_mark: Fixed`.  nixos-unstable advanced to
  7.0.11 (from 7.0.10) since the previous check (via the channel
  pointer).  cifs-utils 7.5 on unstable channels, 7.4 on the 25.11
  channels.
- **Proxmox VE** (via the `pve-no-subscription` `Packages` index): VE 9
  default kernel `proxmox-kernel-7.0` (newest image 7.0.6-2-pve), VE 8
  default `proxmox-kernel-6.8` (newest 6.8.12-28-pve); both unpatched.
  Proxmox ships its own kernel but Debian userland, so cifs-utils is the
  Debian base version (trixie 7.4, bookworm 7.0 — both ≥ 6.14); both rows
  flipped from `:grey_question:` to `:x: Vulnerable`.
- **Rocky Linux** (via the Rocky BaseOS repodata / errata RSS): 10 ⇒
  kernel 6.12.0-211.16.1.el10_2
  / cifs-utils 7.5 (no update); 9 ⇒ 5.14.0-687.12.1.el9_8 / 7.5
  (RLSA-2026:21556, 2026-05-30); 8 ⇒ 4.18.0-553.126.1.el8_10 / 7.0
  (RLSA-2026:21706, 2026-05-31).  All kernels unpatched for CIFSwitch
  (no RLSA cites CVE-2026-46243; no `vet_description` backport in the EL
  kernels yet).  The new RLSAs include CVE-2026-31709 (SMB/CIFS client,
  CWE-1288 out-of-bounds read — a different CIFS vulnerability, not the
  `vet_description` fix).
  SELinux-enforcing default may still constrain the upcall (see Rocky
  notes).
- **Amazon Linux** (via the Amazon Linux core repodata): 2023 ⇒ kernel
  6.1.172-216.329.amzn2023 / cifs-utils 7.5
  (default 6.1 stream; `kernel6.12`/`kernel6.18` streams not tracked
  separately); 2 ⇒ core kernel 4.14.355-282.729.amzn2 / cifs-utils 6.2.
  Both kernels unpatched.  AL2's cifs-utils 6.2 is < 6.14 — primary
  exploit path absent, reduced exposure; its row flipped from
  `:grey_question:` to `:x:`.
- **Upstream stable backports landed 2026-06-01** — all tracked stable
  branches now carry the fix (7.0.11, 6.18.34, 6.12.92, 6.6.142,
  6.1.175, 5.15.209, 5.10.258); no distro kernel advisory has referenced
  `3da1fdf4efbc` yet (checked Rocky errata RSS).
- **CVE-keyed feeds**: NVD record status *Analyzed* (initial analysis
  completed 2026-06-09; CWE-20); NIST has not yet published its own CVSS
  score.  Kernel CNA submitted CVSS 3.1 7.1 High
  `AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N`; CISA-ADP independently scored
  7.8 High `AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` (A:H vs CNA's A:N).
  EPSS scores it 0.00023 (7th percentile, scored 2026-06-11); CISA KEV
  does not list it.  NVD records CWE-20 (Improper Input Validation); the
  Summary keeps the privilege-management framing (CWE-269 / CWE-284)
  alongside it.

## References

| Source | URL |
|---|---|
| Disclosure writeup | <https://heyitsas.im/posts/cifswitch/> |
| Public PoC | <https://github.com/manizada/CIFSwitch> |
| Kernel fix commit | <https://github.com/torvalds/linux/commit/3da1fdf4efbc490041eb4f836bf596201203f8f2> |
| cifs-utils upstream | <https://git.samba.org/?p=cifs-utils.git;a=summary> |
| NVD | <https://nvd.nist.gov/vuln/detail/CVE-2026-46243> |
| MITRE CVE | <https://www.cve.org/CVERecord?id=CVE-2026-46243> |
| Kernel CNA record | <https://lore.kernel.org/linux-cve-announce/?q=CVE-2026-46243> |
| Red Hat | <https://access.redhat.com/security/cve/CVE-2026-46243> |
| FIRST EPSS | <https://api.first.org/data/v1/epss?cve=CVE-2026-46243> |
| CISA KEV | <https://www.cisa.gov/known-exploited-vulnerabilities-catalog> |

[writeup]: https://heyitsas.im/posts/cifswitch/
[poc]: https://github.com/manizada/CIFSwitch
[fix-commit]: https://github.com/torvalds/linux/commit/3da1fdf4efbc490041eb4f836bf596201203f8f2
[cwe-269]: https://cwe.mitre.org/data/definitions/269.html
[cwe-284]: https://cwe.mitre.org/data/definitions/284.html
[cwe-20]: https://cwe.mitre.org/data/definitions/20.html
[nvd]: https://nvd.nist.gov/vuln/detail/CVE-2026-46243
