# RFC 0006: Development workflow: build on the host, run on the device

- **Status:** Accepted (2026-10-23).
- **Date:** 2026-10-23
- **Affected repositories:**
  - toolkit-host (the [development workflow spec](https://github.com/opendap-project/toolkit-host/blob/main/docs/specs/development-workflow.md), `Brewfile`, `setup.sh`);
  - blackbird-os (the `sysroot` profile, `build/dap-sysroot.sh`, the shell's session in the `qemu` profile);
  - system (the Rust workspace, `rust-toolchain.toml`, `scripts/cross.sh`, `scripts/deploy.sh`, the first crates);
  - blackbird-ui (`tokens/generate.py` and its `export/`, `fonts/`).

## Summary

OpenDAP software is written, built and tested on the developer host.

- **Build VM.** The Linux build VM runs mkosi and nothing else. It builds the images and a cross-compilation sysroot from the same Debian packages.
- **Runtimes.** The virtual T1 and the T1 only run software. A change reaches them as a development system extension, without rebuilding the image.
- **Which loop.** Each change uses the fastest loop that can show it:
  - host tests;
  - the product UI in a window on the host;
  - the virtual T1;
  - the T1.

## Motivation

- The first bring-up plan installed a Rust toolchain inside the build VM. The maintainers asked instead for development on the host and VMs as the runtime, and for the whole process to be defined.
- The relevant requirements:
  - TK-001 and TK-010: host tools, UNIX only;
  - OS-009, OS-016, OS-018 and OS-019: run, control and automate Blackbird OS on the host;
  - OS-010: images built from code and packages.
- The stack the decisions name has to be used from the first screen:
  - the own compositor on Smithay;
  - the `dap` user's session;
  - Slint behind the renderer interface;
  - hardware described by data.

## Proposal

The full process is in the [development workflow spec](https://github.com/opendap-project/toolkit-host/blob/main/docs/specs/development-workflow.md). In short:

| Repository | Change |
|---|---|
| toolkit-host | The spec. `zig` and `lima` in the `Brewfile`; `cargo-zigbuild` in `setup.sh` |
| blackbird-os | `mkosi.profiles/sysroot`: the image's packages plus their `-dev` packages, as a tar. `build/dap-sysroot.sh` installs it in a case-sensitive APFS volume, because Linux headers differ only by case. The `qemu` profile gains the shell's session: `dap-shell.service` as the `dap` user through PAM and logind, the display stack's libraries, `systemd-sysext`, the virtual T1's device description |
| system | Workspace with `rust-toolchain.toml` (1.99.1). `scripts/cross.sh`: cargo-zigbuild for `aarch64-unknown-linux-gnu.2.41`, the image's glibc, against the sysroot. `scripts/deploy.sh`: the development extension `opendap-dev`, `systemd-sysext refresh`, restart. Crates `opendap-device` (domain: device description, roles), `dap-shell` (Smithay), `dap-product` (Slint) |
| blackbird-ui | `tokens/generate.py` writes the Slint globals and CSS custom properties from the token file (design system §18.1), into `export/`. IBM Plex is vendored in `fonts/` under OFL-1.1 |

**Interfaces.**

- `opendap.device-description/v1` gets its first schema in code: identity, display and controls. The other sections of hardware support spec §3 follow as their consumers appear.
- The extension name `opendap-dev`, with `ID=_any`, is reserved for development. Images never carry it.

## Alternatives

- **Compile inside the build VM or a container that matches the image.**
  - Not chosen: it puts a second toolchain and a second build cache in a machine that should stay disposable, and it breaks the host's editor, tests and UI preview loop.
  - The sysroot gives the same exactness: the binaries link against the image's own packages.
- **A third-party kiosk compositor (cage) for the first screen.** Not chosen: ADR 0002 and ADR 0012 make `dap-shell` on Smithay the frame, and the image carries only OpenDAP's own ecosystem (OS-024).
- **Rebuild the image for every change.** Not chosen: it takes minutes instead of seconds, and system extensions are already layer L3 of ADR 0011.
- **cross-rs.** Not chosen: its images are x86_64 and run emulated on Apple silicon, and its sysroots are Ubuntu's.

## Open questions

- How the remaining host tools are pinned (zig, cargo-zigbuild, QEMU). mise is a candidate.
- How developer mode is entered and shown on a T1 (ADR 0002). Deploying to a T1 over USB waits for it.
- When the first `.deb` packages are built (cargo-deb) and published to the OpenDAP repositories.
