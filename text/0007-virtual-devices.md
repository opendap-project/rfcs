# RFC 0007: Virtual devices in their own repository

- **Status:** Accepted (2026-10-23).
- **Date:** 2026-10-23
- **Affected repositories:**
  - virtual-devices (new);
  - blackbird-os;
  - system;
  - toolkit-cli and toolkit-host;
  - workspace.

## Summary

Virtual OpenDAP devices move out of blackbird-os into **virtual-devices**, separate from the operating system and from the toolkit (VD-004). A virtual device is a **model over hardware layers**: SoC, board, faceplate and HATs, as in the system hardware support spec (VD-005).

- Each layer brings the QEMU devices that stand in for its hardware, and its part of the device description.
- The virtual board hands the description to the guest through fw_cfg.
- The Blackbird OS `qemu` image therefore carries no model, and one image runs every model.

## Motivation

The virtual T1's runner, its hardware and its device description were built into blackbird-os:

- `build/dap-qemu.sh` held a fixed list of devices;
- `dap-qmp.py` sat next to it;
- the description was baked into the `qemu` profile.

That tied the OS to one virtual model and mixed developer tooling into the OS repository. The maintainers asked for:

- virtual devices separate from the toolkit and the OS;
- organisation by hardware layers, with the model on top;
- emulating everything possible on the host (VD-006);
- a window that opens at the right size (VD-007).

## Proposal

| Repository | Change |
|---|---|
| virtual-devices | `layers/`, `models/`, `build/vdev.py` (composer), `build/dap-qemu.sh` and `build/dap-qmp.py` (moved from blackbird-os), `export/bsp/` (the virtual board's support package), requirements VD-001 to VD-007 (OS-018, OS-033 and OS-034 moved), the virtual devices spec |
| blackbird-os | Runner and description removed; the `qemu` profile installs `../virtual-devices/export/bsp` and no longer sets the panel mode. OS-035: a quiet product boot, with consoles only in the debug boot entry |
| system | `dap-shell` reads `/run/opendap/device/description.toml` (from the hardware) before a support package's `/usr/share/…`; `scripts/deploy.sh` targets virtual-devices |
| toolkit-cli | `opendap-transport-vm` runs the device from virtual-devices, with blackbird-os's image and key |
| toolkit-host | Development workflow updated |
| workspace | `repos.tsv` lists `public/virtual-devices` |

**Interfaces:**

- the fw_cfg entry `opt/org.opendap/device-description`, an `opendap.device-description/v1` document;
- the runner's commands and environment, described in its header;
- `export/bsp/`, installed as is into an image.

## Alternatives

- **Keep virtual devices in blackbird-os.** Not chosen: the maintainers want them separate, and the OS should not know models.
- **Put them in the toolkit.** Not chosen, for the same reason. The toolkit drives virtual devices through their interface.
- **A description baked into each image.** Not chosen: it needs one image per model, against the hardware support spec's "described by data".

## Open questions

- The restructuring that follows this setup:
  - which repository owns what, by domain model, concerns and dependencies;
  - the `opendap` CLI as the user-side surface next to Desktop, apart from the Developer Toolkit;
  - names: the host tools' `dap-` prefix, and the device side's namespace;
  - turning the runner and its scripts into proper toolkit tools.
