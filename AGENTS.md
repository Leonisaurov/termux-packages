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

### Bind Permissions

Extended `-b` syntax with optional access mode:
- `-b /host:/guest:ro` — read-only (writes return `EROFS`)
- `-b /host:/guest:wo` — write-only (reads return `EACCES`)
- `-b /host:/guest:rw` — read-write (default, backward compatible)

Access mode enforced in `translate_path()` after canonicalization.
For `open`/`openat`, reads `O_WRONLY`/`O_RDWR` flags from tracee registers.

### Merge Bind (`-m` / `--mbind`)

Copies rootfs directory contents to host before creating a regular bind.
Files from rootfs overwrite any existing files at host.

```
proot -m /real/run:/run
proot --mbind /real/run:/run
```

- Copies `$ROOTFS/run/*` to `/real/run/` recursively (so `-m` is read as "mbind")
- Overwrites files that already exist at host (no EEXIST)
- Extra host files (not in rootfs) are preserved
- On proot exit, copied files are automatically cleaned up via talloc destructor
- If `$ROOTFS/run` doesn't exist or get_root() returns NULL → skip copy, bind proceeds

### Modified Source Files
- `proot-source/src/path/binding.h` — `BindingAccess`, `BindingType` enums, `Binding` struct extended, `check_binding_access()`
- `proot-source/src/path/binding.c` — mbind_prepare, copy_recursive, mbind_cleanup, check_binding_access, all new_binding() callers updated
- `proot-source/src/cli/proot.h` — `-p`/`--port`/`--protect-privileged-ports`/`-m`/`--mbind` options
- `proot-source/src/cli/proot.c` — handle_option_mbind, handle_option_b with :ro/:wo/:rw
- `proot-source/src/extension/extension.h` — `PortMapping`, `PortSwitchConfig` structs
- `proot-source/src/extension/port_switch/port_switch.c` — security hardening
- `proot-source/src/path/path.c` — access mode check in translate_path()
- `proot-source/src/extension/fake_id0/chroot.c` — updated new_binding() call

## How the Build Works

1. `TERMUX_PKG_SKIP_SRC_EXTRACT=true` skips download
2. `termux_step_pre_configure()` rsyncs `proot-source/` into build dir
3. `make` compiles with all features built-in (no patches)
4. Package step creates `.pkg.tar.xz`

## GitHub Actions Workflow

### `build-proot.yml`
- **Trigger**: Push to `packages/proot/**` or `proot-source/**`
- **Steps**: Clone → zram → restore cache → prepare → build → collect → release → save cache → artifact
- **Caching**: `~/.termux-build` mounted into Docker via `TERMUX_DOCKER_RUN_EXTRA_ARGS`
- **Cache key**: hash of `packages/proot/build.sh`, `packages/libtalloc/build.sh`, `packages/libandroid-shmem/build.sh`
- **Downloads**: `gh release download proot-latest -R Leonisaurov/termux-packages -p "*.pkg.tar.xz"`

### `docker_image.yml`
- Builds/pushes `ghcr.io/leonisaurov/package-builder:latest`

## Workflow Monitoring

Siempre que se lance un workflow, usar `gita notify` para esperar el resultado y leer los logs:

```bash
cd /data/data/com.termux/files/home/Develop/Clones/termux-packages
gita notify build-proot.yml 2>/dev/null | grep -E '(error|##\[error\]|mbind|Success)'
```

- Exit code 0 = éxito, 1 = falló, 2 = cancelado
- `gita notify` bloquea hasta que el workflow termina (o retorna inmediatamente si ya terminó)
- NO usar `timeout`

## Development

1. Edit files in `proot-source/src/`
2. Bump `TERMUX_PKG_REVISION` in `packages/proot/build.sh`
3. Push — workflow triggers automatically
4. Siempre verificar con `gita notify build-proot.yml`

### Commit Guidelines

`<type>(<scope>): <summary>`

Types: `fix`, `enhance`, `chore`, `ci`

## Important Paths

| Path | Purpose |
|------|---------|
| `build-package.sh` | Main build script |
| `scripts/run-docker.sh` | Docker wrapper |
| `packages/proot/` | Proot definition |
| `packages/libtalloc/` | Dependency |
| `packages/libandroid-shmem/` | Dependency |
| `packages/termux-keyring/` | GPG keys for -I dep install |
| `packages/termux-licenses/` | License files (GPL-2.0, etc.) |
| `packages/python/` | Python version (build variable) |
| `packages/libllvm/` | LLVM version (build variable) |
| `packages/termux-elf-cleaner/` | ELF cleaner version (build variable) |
| `packages/libc++/` | C++ stdlib (dep of libllvm/elf-cleaner) |
| `proot-source/src/` | Modified proot source (repo-tracked) |
| `.github/workflows/build-proot.yml` | CI workflow |
| `scripts/buildorder.py` | Dependency resolver (patched for proot-only) |
| `repo.json` | Package directories list |

## Known Issues

- **`buildorder.py`**: Patched to skip missing deps when building a specific package (not full build). This is needed because some remaining packages (libllvm, python) declare dependencies on packages we removed.
- **`/data` mount**: Not used in CI. `-m` flag in `run-docker.sh` mounts `/data` from host which causes permission issues on GHA runners. Cache is mounted via `TERMUX_DOCKER_RUN_EXTRA_ARGS`.
- **First CI run**: ~5 min (seeds cache). Subsequent runs: ~20-30s with cache hit.

## TODO

- **Merge bind (getdents64)**: Implement true overlay-style merge bind where files from
  both host and rootfs are visible simultaneously, not just copied. Requires intercepting
  `getdents64` to merge directory listings (similar to hidden_files extension).
