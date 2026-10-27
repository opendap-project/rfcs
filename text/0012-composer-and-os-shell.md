# RFC 0012: The composer and the OS Shell

- **Status:** Accepted (2026-10-24).
- **Date:** 2026-10-24
- **Affected repositories:** system, blackbird-os, blackbird-ui, virtual-devices, toolkit-host, toolkit-image, toolkit-verify, rfcs.

## Summary

Blackbird OS's display server and interface runtime is the **composer**: it composes the screen from surfaces and layers, turns input into the input domain's events and dispatches them, and carries the primitives every interface is built from. The **OS Shell** is the system's own interface built on the composer: the places and how one moves between them, the status bar and its panel, the volume and power views, consent, toasts, the system keyboard, the lock. The device binary `odap-shell` becomes `odap-composer`; "shell" names the OS Shell only. What exists of the OS Shell today is drafts, hosted in the composer's process, that serve to shape the primitives.

## Motivation

The work so far used "shell" for the compositor and its primitives (`odap-shell`, the shell and UI runtime spec), and also for the system interface that will sit on it. The maintainers set the vocabulary: the compositor and its domain of primitives is the composer; the OS Shell is a later layer, not designed yet, which will define its own instances and singletons (`@status-bar`, `@keyboard`) on the composer's primitives, as each activity will through the SDK (system input events §6.4). One word for two layers hid that boundary.

## Proposal

### What each one is

| The composer (`odap-composer`) | The OS Shell |
|---|---|
| The display: DRM/KMS, the renderer, every display the hardware record lists | The spatial model: the places and how they relate |
| Input: libinput to raw and domain events, controls by role, gestures and their regions, the arena, dispatch with its directives, focus and navigation without touch | What the system's gestures mean (back, the companion screen, more, quick settings) |
| Wayland and its protocols (xdg-shell, dmabuf, presentation, text input and input methods) | The system keyboard and its variants, and where it is placed |
| Surfaces and layer bands, trusted layers drawn above every client, insets of the content area | The status bar and status panel, the volume and power views, consent, pairing, toasts with Undo |
| The render loop and its gates; the mechanism that moves buffers between places | Display power policy: idle, lock, what wakes the display |
| The hand-over with the boot UI; the debug UI mode | The system's identity on screen |

The OS Shell is built on the composer's primitives and registers its parts in the composer's domain language; it does not reach around them. Until it is designed, the status bar, the power menu, toasts, the display-off rules and the keyboard's stand-in live in the composer's process as drafts, marked as such.

### Renames

| Before | After |
|---|---|
| crate and binary `odap-shell` (system) | `odap-composer` |
| `odap-shell.service`, its PAM service, its drop-ins (blackbird-os, virtual-devices' BSP) | `odap-composer.service`, PAM `odap-composer` |
| `/run/opendap/shell/` (the runtime directory: `ui-ready`) | `/run/opendap/composer/` |
| system spec "Shell and UI runtime" (`shell-and-ui-runtime.md`) | "Composer and UI runtime" (`composer-and-ui-runtime.md`), with the OS Shell's parts marked as drafts |
| system ADR 0012 "Shell process model and system layers" | Keeps its number and file; its text names the composer, with a note that links here |

"Shell" stays where it means a Unix shell (a login shell, a shell script, OS-035's "no boot shells"), and in `xdg-shell`.

### In each repository

- **system:** the crate, its binary and its strings; the specs (composer and UI runtime, input events, hardware support, user-facing conventions), the ADRs' text, requirements; the UI primitives activity; `odap-boot`'s references; the workspace's deploy restart list.
- **blackbird-os:** the unit, its PAM name, the units ordered against it, the post-install script, the image README. The image is rebuilt.
- **blackbird-ui:** the specs' references (ui-model, design system).
- **virtual-devices:** `odap-vdev ui activity` and `ui debug` (the unit's name), the BSP's drop-in and orderings, the layers' comments, the spec.
- **toolkit-host, toolkit-image, toolkit-verify:** the development workflow spec, `odapkit cross`'s example, `odap-inject`'s documentation.
- **rfcs:** RFC 0008 and 0009 keep their text as records; this RFC supersedes their mention of `odap-shell`.

## Alternatives

- **Keep `odap-shell` and only reword the documents.** Rejected: the binary's name is the one most people meet first (logs, units, `systemctl`), and it would keep teaching the wrong layer.
- **`odap-compositor`.** Rejected: the composer is more than composition of buffers (input, dispatch, focus, the primitives' domain); "composer" is the maintainers' word.

## Open questions

- Whether the OS Shell runs in the composer's process or as a privileged client of it: decided when the OS Shell is designed.
