# AGENTS.md — termux-packages

## Project Overview

This is a fork of the official [termux-packages](https://github.com/termux/termux-packages) build system. It cross-compiles Linux packages for Android using the Android NDK, producing `.deb` or `.pkg.tar.xz` packages for 4 architectures: `aarch64`, `arm`, `i686`, `x86_64`.

**Current focus**: proot package with custom port mapping.

## Build System

### Core Architecture

- **Main script**: `build-package.sh` — orchestrates the entire build pipeline
- **Build steps**: `scripts/build/` — 44+ step implementations
- **Package definitions**: `packages/` containing `build.sh` files
- **Docker-based**: Builds run inside `ghcr.io/termux/package-builder:latest` via `scripts/run-docker.sh`

### Key Commands

```bash
# Build proot (our main package)
./scripts/run-docker.sh ./build-package.sh -I -a aarch64 --format pacman proot

# Build without Docker (if container already set up)
./build-package.sh -I -a aarch64 --format pacman proot

# Lint package recipe
./scripts/lint-packages.sh packages/proot/build.sh
```

## Proot Package

### Location
- `packages/proot/build.sh` — package definition (v5.1.107.86)
- `packages/proot/termux-chroot` — termux-chroot script template
- `proot-source/` — modified source directory (no patches, no downloads)

### Dependencies
- `libandroid-shmem`, `libtalloc`

### Port Mapping

Two modes:

1. **Explicit mapping** (`-p host:container` / `--port host:container`):
   - Maps container port to host port, max 64 mappings
   - Auto-finds free port if target is occupied

2. **Auto-redirect** (`--protect-privileged-ports`):
   - Redirects `bind()` for ports < 1024 by +2000
   - Finds next free port if occupied

### Modified Source Files
- `proot-source/src/extension/extension.h` — `PortMapping` and `PortSwitchConfig` structs
- `proot-source/src/cli/proot.h` — `-p`/`--port` and `--protect-privileged-ports` options
- `proot-source/src/cli/proot.c` — `handle_option_port_mapping()` and `handle_option_p()` with config
- `proot-source/src/extension/port_switch/port_switch.c` — helper functions and syscall interception

### Helper Functions (port_switch.c)
- `extract_port()` — read port from sockaddr in 1 call instead of 12 blocks
- `is_port_available()` — tests port with `bind()` + `SO_REUSEADDR`
- `find_available_port()` — searches up to 100 consecutive ports
- `get_sockaddr_len()` — returns correct size (16 for AF_INET, 28 for AF_INET6)
- `write_back_port()` — writes modified sockaddr, correct size via `get_sockaddr_len()`
- `is_localhost()` — optimized byte comparison (no inet_ntop/strcmp)

### Security Hardening (committed in b36b96d051)
- **Correct write-back size**: `get_sockaddr_len()` prevents 128-byte corruption
- **read_data error checking**: prevents using uninitialized data
- **find_available_port returns 0 on failure**: instead of occupied port
- **is_localhost optimized**: raw byte comparison, no string formatting
- **printf → VERBOSE**: consistent debug output via cli/note.h

## How the Build Works (No Patches)

The proot source is maintained directly in `proot-source/` in the repo. When `build-package.sh` runs:

1. `TERMUX_PKG_SKIP_SRC_EXTRACT=true` prevents downloading source
2. `termux_step_pre_configure()` runs `rsync -a --exclude=.git proot-source/` to copy source into the build directory
3. `make` compiles proot with the port mapping feature built-in
4. `make install` and `termux_step_post_make_install()` package the binary

To modify the port mapping: edit files in `proot-source/src/`, bump `TERMUX_PKG_REVISION` in `build.sh`, and push.

## GitHub Actions Workflows

### `build-proot.yml` (Primary)

- **Trigger**: Push to `packages/proot/**`, `proot-source/**` or manual dispatch
- **Architecture**: `aarch64` only
- **Format**: pacman (`.pkg.tar.xz`)
- **Steps**: Clone → zram → build proot → collect → GitHub Release → artifact
- **Key command**: `./scripts/run-docker.sh ./build-package.sh -I -a aarch64 --format pacman proot`
- **Downloads**: `gh release download proot-latest -R Leonisaurov/termux-packages -p "*.pkg.tar.xz"`

### `docker_image.yml`

- Builds and pushes Docker builder image to `ghcr.io/leonisaurov/package-builder:latest`
- Use GHA cache for rebuilds (~3-4 min after first ~15-20 min build)

### Critical: `free-space.sh` Corrupts Docker

**DO NOT use `scripts/free-space.sh` in proot workflow.** Line 51 deletes `/var/lib/containerd/io.containerd.content.v1.content/` which destroys Docker image layer cache. This causes `failed to extract layer ... blob not found` errors on subsequent builds.

Proot is small (~100KB source) and doesn't need the 36GB that `free-space.sh` frees.

## Downloading Builds

```bash
# With gh CLI
gh release download proot-latest -R Leonisaurov/termux-packages -p "*.pkg.tar.xz"

# With curl
curl -LO https://github.com/Leonisaurov/termux-packages/releases/download/proot-latest/proot-*.pkg.tar.xz
```

## Development Workflow

### Modifying Port Mapping

1. Edit source files in `proot-source/` directly
2. `cd proot-source && git diff` to review changes
3. Update `TERMUX_PKG_REVISION` in `packages/proot/build.sh`
4. Push — workflow triggers automatically (detects `proot-source/**` changes)

### Commit Guidelines

Format: `<commitType>(<scope>): <summary>`

Types: `addpkg`, `bump`, `fix`, `dwnpkg`, `disable`, `enhance`, `chore`, `rebuild`, `scripts`, `ci`

Examples:
- `fix(proot): fix port mapping write-back size corruption`
- `ci(build-proot): remove free-space.sh that corrupts Docker layers`

## Important Paths

| Path | Purpose |
|------|---------|
| `build-package.sh` | Main build script |
| `scripts/run-docker.sh` | Docker container management |
| `scripts/free-space.sh` | **DO NOT USE** — corrupts Docker layers |
| `packages/proot/` | Proot package definition |
| `proot-source/` | Modified proot source (git-initialized, in `.gitignore`) |
| `.github/workflows/build-proot.yml` | Proot CI workflow |

## Common Build Issues

- **Docker blob corruption**: Caused by `free-space.sh` deleting containerd content store. Remove `free-space.sh` from workflow.
- **Write-back memory corruption**: Use `get_sockaddr_len()` not `sizeof(struct sockaddr_storage)` when writing back to tracee.
- **Port conflicts**: Use higher ports (8080+) — Android reserves low ports.

## Testing

```bash
# Build proot
./scripts/run-docker.sh ./build-package.sh -I -a aarch64 --format pacman proot

# Lint
./scripts/lint-packages.sh packages/proot/build.sh
```
