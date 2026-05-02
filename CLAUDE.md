# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

OT-SA-Tools is a FreeBSD-based build toolchain forked from OPNsense/tools. It produces bootable images (ISO, memstick, VM disk, nano flash, ARM) by orchestrating builds across four sibling repositories: `src` (FreeBSD source), `ports` (third-party packages), `core` (OT-SA core), and `plugins`.

All builds run as root on FreeBSD 14.3-RELEASE (amd64). This toolchain is not designed to run on Linux or macOS.

## Build Commands

```sh
make update                     # clone/pull all sibling repos
make base                       # build userland
make kernel                     # build kernel
make ports                      # build all ports
make plugins                    # build plugins
make core                       # package OT-SA core
make dvd                        # produce DVD ISO image
make vm                         # produce VM disk image
```

### Lint and Test

```sh
make lint                       # sh -n syntax check on ALL scripts (build + composite)
make lint-steps                 # syntax check build/*.sh only
make lint-composite             # syntax check composite/*.sh only
make test                       # regression tests (requires core to be built first)
make audit                      # vulnerability check on built packages
```

**Lint is a prerequisite for every build step** — the Makefile wires `lint-steps` as a dependency for all STEPS targets.

## Build Dependency Graph

```
base ──→ kernel
base ──→ ports ──→ plugins ──→ core ──→ packages / test
base ──→ distfiles
kernel + core ──→ dvd / nano / serial / vga / vm / arm
```

Stages are incremental — each picks up where it left off. Reissuing `ports` clears `plugins` and `core` progress.

## How the Makefile Dispatches

The Makefile defines two sets of targets:

- **STEPS** (30+ targets like `base`, `kernel`, `ports`): each invokes `build/<step>.sh` with a long list of `-flag value` arguments parsed by `build/common.sh` via `getopts`.
- **SCRIPTS** (composite targets like `nightly`, `distribution`, `hotfix`): each invokes `composite/<script>.sh` directly.

Targets with hyphens (e.g., `make ports-curl`, `make clean-base,kernel`) are split: the prefix selects the step, the suffix becomes positional arguments passed to the script.

## Key Source Files

- **`build/common.sh`** — the most important file. Contains option parsing, all `git_*` helpers, `setup_*` functions (chroot, base, kernel, packages, stage), signing/verification, and package management. Every build step sources this.
- **`composite/util.sh`** — helper for composite scripts: `load_core_version()` and `load_make_vars()`.
- **`config/<ABI>/build.conf`** — per-ABI config defining OS, language, and SSL versions. Auto-selected by matching `OS` field against the running FreeBSD version.
- **`config/<ABI>/repositories.yaml`** — optional YAML config to override git repository URLs and mirrors (not tracked in git).
- **`device/<NAME>.conf`** — device profiles that export architecture vars and define `arm_install_uboot()` hooks.

## Shell Script Conventions

- All scripts use `set -e` (exit on error).
- Status messages use `>>> ` prefix to stderr. Errors use `echo "..." >&2` with `exit 1`.
- Exported variables follow `PRODUCT_*` (product metadata) or `REPO_*` (git state) prefixes.
- Functions use lowercase with underscores: `git_clone()`, `setup_base()`, `generate_signature()`.
- Optional/fallible operations use `|| true` to suppress errors under `set -e`.
- Scripts are POSIX `sh`, not bash — no bashisms.

## Config and Secrets

Files tracked in `.gitignore` (never commit these):
- `config/*/build.conf.local` — local build overrides
- `config/*/plugins.conf.local` — local plugin overrides
- `config/*/repositories.yaml` — custom repository URL config (may contain private URLs)
- `config/*/repo.key` and `config/*/repo.pub` — signing keys

### build.conf vs build.conf.local

| | `build.conf` | `build.conf.local` |
|---|---|---|
| Committed to git | ✅ Yes | ❌ No (gitignored) |
| Required | ✅ Yes (hard include) | ❌ No (soft include) |
| Loaded order | Second | First |
| Syntax | `?=` (set if undef) | `=` (force set) |
| Purpose | Project defaults | Per-developer overrides |

**Important:** Always use `=` (not `?=`) in `build.conf.local`. Since the .local file is loaded BEFORE build.conf, using `?=` won't override anything because build.conf also uses `?=` and will skip already-set variables. Templates available at `config/<ABI>/build.conf.local.example`.

## Repository Configuration

To override default repository URLs (https://github.com/opnsense/*) and mirror URLs:

1. Copy the example template:
   ```sh
   cp config/25.7/repositories.yaml.example config/25.7/repositories.yaml
   ```

2. Edit `repositories.yaml` to customize:
   - `git_base` — base URL for repositories (default: https://github.com/opnsense)
   - `repositories` — individual repo overrides (set to `null` to use git_base/<repo_name>)
   - `mirrors` — list of mirror URLs for prefetch/clone operations

3. The build system will automatically use YAML config if present. Override precedence (high to low):
   - Command-line: `make -O "custom_url"`
   - `build.conf.local`: `GITBASE=custom_url`
   - `repositories.yaml`: `git_base` and `repositories` sections
   - Makefile default: `https://github.com/opnsense`

## Device Hook System

Image creation calls hook functions from two sources (config hooks first, then device hooks):
1. `config/<ABI>/extras.conf` — hooks like `dvd_hook()`, `vm_hook()`, etc.
2. `device/<NAME>.conf` — same hook names, device-specific.

The hook receives the target filesystem root as `${1}`.
