# AGENTS.md — termux-packages (proot-only fork)

## Project Overview

Proot-only fork of termux-packages. Cross-compiles proot for Android aarch64 using NDK r29 via Docker.

The source lives directly in `proot-source/` — no patches, no downloads.

## Build System

- **Main script**: `build-package.sh` — orchestrates the entire build pipeline
- **Docker**: `scripts/run-docker.sh` runs builds inside `ghcr.io/termux/package-builder:latest`
- **Cache**: GitHub Actions caches `~/.termux-build` for incremental builds (~20s when nothing changes)

### Key Commands

```bash
./scripts/run-docker.sh ./build-package.sh -I -a aarch64 --format pacman proot
```

## Proot Package

### Location
- `packages/proot/build.sh` — package definition
- `packages/proot/termux-chroot` — termux-chroot script template
- `proot-source/src/` — modified source (1.3 MB, tracked in repo)

### Dependencies
- `libandroid-shmem`, `libtalloc` (built automatically via `-I` flag)

### Port Mapping

1. **Explicit mapping** (`-p host:container` / `--port host:container`):
   - Max 64 mappings, auto-finds free port if target occupied

2. **Auto-redirect** (`--protect-privileged-ports`):
   - Redirects `bind()` for ports < 1024 by +2000, finds next free port

### Modified Source Files
- `proot-source/src/extension/extension.h` — `PortMapping`, `PortSwitchConfig` structs
- `proot-source/src/cli/proot.h` — `-p`/`--port`/`--protect-privileged-ports` options
- `proot-source/src/cli/proot.c` — option handlers with config
- `proot-source/src/extension/port_switch/port_switch.c` — helper functions, syscall interception, security hardening

## How the Build Works

1. `TERMUX_PKG_SKIP_SRC_EXTRACT=true` skips download
2. `termux_step_pre_configure()` rsyncs `proot-source/` into build dir
3. `make` compiles with port mapping feature built-in (no patches)
4. Package step creates `.pkg.tar.xz`

## GitHub Actions Workflow

### `build-proot.yml`
- **Trigger**: Push to `packages/proot/**` or `proot-source/**`
- **Steps**: Clone → zram → restore cache → build → collect → release → save cache → artifact
- **Caching**: `~/.termux-build` keyed on build.sh hashes. Restore-key falls back to any prior cache.
- **Downloads**: `gh release download proot-latest -R Leonisaurov/termux-packages -p "*.pkg.tar.xz"`

### `docker_image.yml`
- Builds/pushes `ghcr.io/leonisaurov/package-builder:latest`

## Development

1. Edit files in `proot-source/src/`
2. Bump `TERMUX_PKG_REVISION` in `packages/proot/build.sh`
3. Push — workflow triggers automatically

### Commit Guidelines

`<type>(<scope>): <summary>`

Types: `fix`, `enhance`, `chore`, `ci`, `bump`

## Important Paths

| Path | Purpose |
|------|---------|
| `build-package.sh` | Main build script |
| `scripts/run-docker.sh` | Docker wrapper |
| `packages/proot/` | Proot definition |
| `packages/libtalloc/` | Dependency |
| `packages/libandroid-shmem/` | Dependency |
| `proot-source/src/` | Modified proot source (repo-tracked) |
| `.github/workflows/build-proot.yml` | CI workflow |
