# ZainiumOS

**Built to Endure. Programmed to Lead.**

[![Website](https://img.shields.io/badge/Website-zainiumdynamics.tech-0d1117?style=flat-square&logo=internetexplorer&logoColor=00d9ff&labelColor=0d1117&color=00d9ff)](https://zainiumdynamics.tech)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue?style=flat-square&logo=gnu&logoColor=white)](https://www.gnu.org/licenses/gpl-3.0)
[![Built with Rust](https://img.shields.io/badge/Built%20with-Rust-f74c00?style=flat-square&logo=rust&logoColor=white)](https://www.rust-lang.org)
[![Linux](https://img.shields.io/badge/Linux-Kernel-FCC624?style=flat-square&logo=linux&logoColor=black)](https://kernel.org)
[![LFS](https://img.shields.io/badge/Base-Linux%20From%20Scratch-2c3e50?style=flat-square&logo=linuxserver&logoColor=white)](https://www.linuxfromscratch.org)
[![Platform](https://img.shields.io/badge/Platform-x86__64-6e40c9?style=flat-square&logo=intel&logoColor=white)](https://zainiumdynamics.tech)
[![musl libc](https://img.shields.io/badge/libc-musl-darkgreen?style=flat-square&logoColor=white)](https://musl.libc.org)
[![Status](https://img.shields.io/badge/Status-Active%20Development-brightgreen?style=flat-square)](https://zainiumdynamics.tech)

ZainiumOS is a next-generation operating system built from scratch on a Linux From Scratch (LFS) foundation. It is not derived from any existing distribution. The filesystem layout, package management model, environment system, and core toolchain are all original work, designed and built by **Ali Zain**.

---

## What Is ZainiumOS

Most Linux distributions carry decades of structural compromise — shared directories, mutable system state, dependency graphs that span the entire filesystem, and package managers that can silently overwrite critical binaries. ZainiumOS was built to eliminate all of that.

Starting from LFS, every component of the system is intentional. The filesystem is divided into a permanently clean, immutable base layer and a fully isolated userland layer. These two layers never share the same physical location on disk. They are merged at runtime through a precisely controlled union and symlink model. The result is an operating system where the core can never be broken by anything that happens in userland.

---

## Filesystem Architecture

```
/ (Root)
├── dev/
├── home/
├── proc/
├── run/
├── sys/
│
├── zaisys/                            # Kernel and early-boot environment
│   └── kernel/                        # Kernel binary and quantra-ramfs
│
├── etc/          ──→  /overlayer/syshub/etc        # Symlink — no files live here directly
├── var/          ──→  /overlayer/zexlib/union/var  # Symlink — runtime data
├── tmp/          ──→  /overlayer/zexlib/union/tmp  # Symlink — temporary files
│
└── overlayer/
    │
    ├── syshub/                        # Immutable Base OS — read-only at runtime
    │   ├── bin/                       # Core system binaries (coreutils, cmake, etc.)
    │   ├── sbin/                      # Essential system administration binaries
    │   ├── lib/                       # Base system libraries
    │   ├── etc/                       # Core system configuration
    │   ├── drivers/                   # Hardware modules and system-level driver hooks
    │   │   ├── hardware/              # Firmware, GPU, sensor, peripheral modules
    │   │   ├── system/                # Zex operation drivers (install/detach/upgrade)
    │   │   └── modules/               # Kernel modules
    │   └── engine/                    # Init and service engine
    │       ├── services/              # Zainium System service definitions
    │       └── quantra                # PID 1 init binary
    │
    └── zexlib/                        # Writable Userland — all installed packages land here
        ├── registry/
        │   ├── packages.toml          # Master package state — versions, origins, dependency graph
        │   └── installed/             # Per-package receipt files
        │       ├── nginx.toml
        │       ├── rust.toml
        │       └── ...
        └── union/                     # Active merged layer — symlinked with syshub at runtime
            ├── bin/                   # Installed package binaries + syshub/bin symlinks
            ├── sbin/
            ├── lib/                   # Installed libraries + syshub/lib symlinks
            ├── etc/                   # Userland configuration
            ├── var/                   # Runtime and application data
            └── tmp/                   # Temporary files
```

The `/etc`, `/var`, and `/tmp` entries at the root are pure symlinks. They contain no files of their own. This keeps the root directory minimal and ensures that configuration and runtime data always resolve to their correct layer.

---

## The Two Layers

### syshub — The Immutable Base

`syshub` is the complete, self-sufficient operating system. It contains the desktop environment, coreutils, cmake, essential libraries, and all system-critical tooling. It is mounted read-only at runtime.

No package installed by the user ever enters `syshub`. It stays permanently clean. There is no dependency hell possible at the base layer because the base layer is never modified after installation. If the entire `zexlib` userland is deleted or corrupted, the system still boots fully into a working desktop using only `syshub`.

### zexlib — The Writable Userland

`zexlib` is where all user-installed software lives. Its `union/` directory is symlinked with `syshub` at runtime, so installed packages appear alongside base system tools transparently. The user sees one unified system. On disk, the two layers are physically separate.

The `union/bin/`, `union/lib/`, and other subdirectories each hold either files extracted by the package manager or symlinks pointing back into `syshub`. The base system files are never duplicated — only referenced. A package installed from Alpine or Chimera extracts into `union/` and is immediately available on the `$PATH` without touching `syshub`.

---

## Package Management

ZainiumOS uses **Zex** as its package manager. Zex operates entirely within `zexlib`. It sources packages from Alpine, Chimera, and the native `.zxpkg` format. The user never interacts with these backends directly — Zex presents a single unified interface.

### packages.toml — The Master State File

When a package is installed, Zex writes a record into `zexlib/registry/packages.toml`. This file is the single source of truth for the system's desired state. Each entry records the package name, version, origin mirror, and its full dependency graph inline. Dependencies are not resolved at install time by scanning the filesystem — they are declared in the graph at write time, so Zex already knows what is needed before a single file is extracted.

When the user runs an upgrade, Zex reads `packages.toml`, contacts the appropriate mirrors, and updates the entries and the files in `union/` directly. There is no separate upgrade database. The same file governs both state and update resolution.

### installed/ — Package Receipts

Every package that Zex installs also generates a receipt file at `zexlib/registry/installed/<name>.toml`. The receipt records every file that the package placed into the `union/` layer. When a package is removed, Zex reads only its receipt and deletes exactly those files. Nothing else is touched. No orphaned files, no leftover libraries, no stale configuration.

### Collision Blocking

Before extracting any package, Zex checks whether any of the incoming files conflict with files already present in `union/`. If a conflict is detected, the installation is aborted. Files from `syshub` cannot be overwritten under any circumstance. This guarantees that no package — regardless of origin — can silently replace a base system binary.

---

## Environment System

ZainiumOS manages the user environment through a dedicated Rust-based environment binary. This replaces the traditional `/etc/profile.d` shell script chain. It sets the `$PATH`, configures aliases, and manages `.bashrc` behavior from a single controlled source. The profile script and `.bashrc` equivalent are both handled by this environment layer rather than being scattered across the filesystem.

---

## Design Principles

ZainiumOS is built on a small set of non-negotiable constraints.

The base system is immutable. User-installed software has no mechanism to reach it. The package state is a single file, not a database scattered across `/var/lib`. Every install and every removal is fully reversible because every change is recorded in a receipt. The environment is managed programmatically, not through fragile shell script inheritance. And the entire userland can be wiped and rebuilt without touching the running OS.

---

## Creator

ZainiumOS is designed and built by **Ali Zain**.

---

## License

ZainiumOS is free software licensed under the **GNU General Public License v3.0 (GPL-3.0)**.

You are free to use, study, modify, and distribute this software under the terms of the GPL v3. Any derivative work must also be distributed under the same license. See the [LICENSE](LICENSE) file for the full license text, or visit [https://www.gnu.org/licenses/gpl-3.0.html](https://www.gnu.org/licenses/gpl-3.0.html).
