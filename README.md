# CIFSwitch — CIFS cifs.spnego key-origin LPE tracking site

Patch-status tracker for **CIFSwitch**, a Linux kernel local privilege
escalation in the CIFS client: the kernel never validated that a
`cifs.spnego` key description originated from kernel code, so an
unprivileged user can forge one and steer the rootful `cifs.upcall`
helper (from cifs-utils ≥ 6.14) into switching into an attacker-controlled
namespace and loading an attacker-supplied NSS module — arbitrary code as
root.  Discovered by Asim Viladi Oglu Manizada and
[disclosed on 2026-05-27](https://heyitsas.im/posts/cifswitch/).  Public
PoC: <https://github.com/manizada/CIFSwitch>.

The canonical fix is Linux kernel mainline commit
[`3da1fdf4efbc`](https://github.com/torvalds/linux/commit/3da1fdf4efbc490041eb4f836bf596201203f8f2),
which adds a `.vet_description` hook that rejects forged key descriptions.

No CVE has been assigned yet; the tracker uses the placeholder
`CVE-2026-XXXXX`.

The rendered site is published at **<https://kimmo.cloud/cifswitch/>**.
Deployment plan and current setup state live in [`WEBSITE.md`](WEBSITE.md).

## Source of truth

The tracker is a single Hugo page: [`site/content/_index.md`](site/content/_index.md).
Edit that file; everything else is build infrastructure.

## Local development

Requires Hugo extended (≥ 0.146.0) and Go (for Hugo Modules to fetch the
PaperMod theme).

### With Nix (recommended)

```sh
nix develop          # dev shell: hugo, go, git, resvg
cd site
hugo server          # local preview at http://localhost:1313/cifswitch/
```

If you use [direnv](https://direnv.net/), `direnv allow` once and the
dev shell auto-activates whenever you `cd` into the repo.

### Without Nix

Install Hugo extended ≥ 0.146.0 and Go ≥ 1.24 yourself, then:

```sh
cd site
hugo server          # http://localhost:1313/cifswitch/
```

## Build and publish

```sh
make build       # local build into site/public/
make dist        # build, then rsync to haig:.www/sites/kimmo.cloud/htdocs/cifswitch/
make banner      # re-rasterise the social banner SVG → PNG (needs resvg + Roboto)
```

`make dist` runs `make build` first.  `make banner` is only needed after
editing `site/assets/cifswitch-tracker.svg`; the rendered PNG is committed.

## Repo layout

```
.
├── flake.nix              # Nix dev environment (hugo, go, git, resvg + RPM tools)
├── .envrc                 # direnv hook → `use flake`
├── .gitignore
├── Makefile               # `make build`, `make dist`, `make banner`
├── LICENSE                # CC BY 4.0
├── README.md              # this file
├── CLAUDE.md              # project instructions for Claude Code
├── WEBSITE.md             # publication plan / decisions log
├── scripts/               # auto-update agent: prompt + driver
├── systemd/               # user-level timer + service units
└── site/                  # Hugo project
    ├── hugo.toml
    ├── content/
    │   └── _index.md      # the tracker (single page)
    ├── assets/css/extended/custom.css  # PaperMod CSS overrides
    ├── assets/cifswitch-tracker.svg    # social-banner source (→ make banner)
    ├── static/cifswitch-tracker.png    # rendered OpenGraph banner (committed)
    ├── layouts/partials/  # PaperMod overrides (post_meta, extend_footer)
    ├── go.mod, go.sum     # Hugo Modules — pulls PaperMod theme
    └── …                  # standard Hugo skeleton
```

## License

[CC BY 4.0](LICENSE) — share and adapt with attribution.
