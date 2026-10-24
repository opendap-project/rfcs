# RFC 0009: odapkit, the maintainers' command, and its boundary with opendap

- **Status:** Accepted (2026-10-23). The name and the boundary were set by the maintainers; the rest was decided by Claude under their delegation. Written by Claude.
- **Date:** 2026-10-23
- **Affected repositories:** toolkit-host, toolkit-image, cli, virtual-devices, blackbird-os, system, toolkit-verify, claude-plugins.

## Summary

The Developer Toolkit gets one command, `odapkit`. Like `git` and `cargo`, it dispatches `odapkit <tool>` to an independent `odapkit-<tool>` executable, which stays in its own repository. The toolkit's host tools are renamed after it. The `opendap` CLI and `odapkit` cooperate without depending on each other the wrong way: `opendap` never needs `odapkit`, and `odapkit` may use `opendap`.

## Motivation

The maintainers set out the following:

- **One command for the toolkit.** The `odap-*` host tools are separate scripts, but they should work as subcommands of one base toolkit command, such as `odapkit venv --help`. The name must be told apart from `opendap` and its aliases.
- **Cooperation without competition.** `opendap` and `odapkit` work at different levels of development, and should cooperate and integrate seamlessly.
- **`opendap` stands alone.** It must exist for users of that CLI who are not internal maintainers with access to `odapkit`. So `opendap` neither calls `odapkit` nor stops working without it.
- **`odapkit` is self-sufficient in its niche.** It uses `opendap` when it needs to, rather than reinventing what `opendap` should do.

There was also a naming problem. `odap-*` names two different things: the maintainers' host tools (`odap-image`, `odap-cross`…) and the device's own binaries (`odap-shell`, `odap-boot`, `odap-reboot`).

## Design

### The command

- **Dispatch.** `odapkit TOOL [ARGS…]` runs `odapkit-TOOL ARGS…`. It looks for `odapkit-TOOL` in the `bin/` folders of the workspace's toolkit repositories (through `repos.tsv`), then on `PATH`. Any repository can add a tool by shipping an `odapkit-<tool>` executable.
- **Discovery.** `odapkit` or `odapkit help` lists every tool it finds, with the first line of each tool's `--help`. `odapkit help TOOL` is `odapkit-TOOL --help`.
- **Home.** The dispatcher lives in toolkit-host (`bin/odapkit`), and `setup.sh` links it into `~/.local/bin`.
- **Conventions,** shared by every tool:
  - `--help` and a one-line summary;
  - exit status 0 for success, 1 for failure, 2 for a usage error;
  - `--json` where a tool has machine-readable output.

### Names

| Before | After |
|---|---|
| `odap-image`, `odap-sysroot`, `odap-cross`, `odap-deploy`, `odap-dev-key`, `odap-security-check` (toolkit-image) | `odapkit-image`, `odapkit-sysroot`, `odapkit-cross`, `odapkit-deploy`, `odapkit-dev-key`, `odapkit-security-check`: `odapkit image build qemu`… |
| `odap-vmnet` (toolkit-host) | `odapkit-vmnet`: `odapkit vmnet install` |

- **What stays `odap-*`:** the device's binaries and units (`odap-shell`, `odap-boot`, `odap-reboot`, `odap-power.socket`, `odap-inject`), the crates, and environment variables (`ODAP_*`). `odap` remains the technical identifier for the device and its protocol.
- **`odap-vdev` keeps its name.** It runs the emulator, which belongs to the user side: package developers reach it through `opendap vm`, the way Android's emulator ships with the SDK rather than with the platform's build tools. It is not a toolkit tool.

### The boundary

| | May use | Never uses |
|---|---|---|
| `opendap` (user side) | Devices over their transports; the emulator (virtual-devices `odap-vdev`) | `odapkit`, or any toolkit repository: not as a call, not as a required path, not as the only fix in a hint |
| `odapkit` (maintainers) | Its own tools; `opendap` for device operations it would otherwise duplicate (exec, screen, input, reboot), when installed | — |
| virtual-devices (emulator) | Its own runner; an image and a key from wherever the user has them | `odapkit` |

What this changes today:

- **The image.** `odap-vdev` looks in this order:
  1. `ODAP_IMAGE`;
  2. a blackbird-os checkout's build output, when there is one: a maintainer's latest build;
  3. the user's images, `~/.local/share/opendap/images/blackbird-os-qemu.raw`, where released images will be put.
- **The build in progress.** `opendap` no longer looks for an `odap-image` process. An image being rebuilt carries a `.building` marker next to it, which `odapkit image build` writes and `odap-vdev status` reports.
- **Hints.** `opendap`'s messages name what a user can do: put an image in place, set a variable. A maintainer's tool is never the only way out.
- **SSH.** The virtual device's key is the user's own key, `~/.local/share/opendap/dev-ssh/`. `odapkit dev-key` creates it for maintainers' images. Device pairing (`opendap device pair`) will be how users get one onto released images. Until then, such a key comes from a maintainers' image.

## Consequences

- Every toolkit tool is reachable from one command, and listed by it.
- The two kinds of `odap-*` names are told apart by their prefix.
- A user with only `opendap` and an image runs the emulator.
- Released images and device pairing are what is still missing for users outside the workspace; this RFC does not add them.
