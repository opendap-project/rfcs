# RFC 0020: One input model for the device and the host

- **Status:** Accepted (2026-10-27).
- **Date:** 2026-10-27
- **Affected repositories:** system (the crate's home, its users on the device), cli (the host's model re-exports it), virtual-devices (the simulator window, through cli)
- **Related:** [RFC 0004](0004-opendap-protocol.md) (the OpenDAP Protocol), [RFC 0017](0017-naming-and-topology.md) (names; the crate's name at the `blackbird` cut), [RFC 0019](0019-versioning-and-releases.md) (schema versions)

## Summary

A control's role and the input events a host sends a device are defined four times today: as an enum of seven roles in the device description (system `opendap-device`), an enum of four in the input pipeline (system `opendap-input`), a free string on the host (cli `opendap-model`), and the protocol's events again on the host. This RFC puts them in one crate, `opendap-bus-model` in system (`blackbird-bus-model` at the `blackbird` cut, RFC 0017 §3), which both sides use. The crate holds `Role`, one closed and versioned vocabulary; `InputEvent`, the events of `opendap.input/v1`; and the schema identifiers both sides check. The wire format does not change.

## Motivation

- **Four definitions drift.** The device knows `back`, `select` and `menu`; the input pipeline does not. The host accepts any string and learns that a role does not exist only from the device's refusal. The same JSON is described twice, in two repositories.
- **A role is a vocabulary, not a name.** Software adapts to the hardware it runs on (SYS-040) through what each device's description declares: which roles it has, on which edge, with which code. The meaning of a role (what `volume-up` does) is shared by the device, the host, the input pipeline and the simulator, so it is defined once. Linux's `KEY_*` codes work the same way: a fixed set that a keyboard uses part of.
- **Names collide.** `opendap-input`'s navigation model also calls its node kinds `Role`.

## Proposal

### 1. The crate

`opendap-bus-model`, a library with no dependencies beyond `serde` (MPL-2.0, as a library others build on). It contains:

- **`Role`**, `Copy`: `action`, `volume-up`, `volume-down`, `main`, `back`, `select`, `menu` (kebab-case on the wire). `as_str`, `FromStr` (an unknown name is an error that lists the known ones) and `ALL`.
- **`InputEvent`**: `key {role, hold_ms}`, `key-down {role}`, `key-up {role}`, `keyboard {key, down}` and `tap {x, y}`, tagged by `kind`, as `opendap.input/v1` already carries them.
- **Schema identifiers**: `opendap.input/v1` and `opendap.device-description/v1`.

### 2. The vocabulary is versioned

A device declares the roles it has (its description) and rejects others. A new kind of control adds a role to the crate. That is an addition, so the schema keeps `opendap.input/v1` (RFC 0019 §4, Schemas), and the crate gets a minor version. A host or device that does not know the role yet refuses it with the list of the roles it knows, instead of sending or accepting a name it cannot act on.

### 3. Each repository

- **system:** `opendap-device` re-exports `Role` in place of its own enum. `opendap-input`'s semantic layer maps every role: a role without a meaning there yet gives no semantic action. Its navigation `Role` becomes `NavRole`. The composer is unchanged apart from these names.
- **cli:** `opendap-model` re-exports `Role` and `InputEvent` (a git dependency on system, by revision) in place of `Role(String)` and its own events. The commands parse a role with `FromStr`.
- **virtual-devices:** the simulator window takes the types through cli, as now; its pins move to the revisions that have them.

## Alternatives

- **Keep the host's role a free string.** The host stays open to roles it has not heard of, but nothing can act on such a role: the device rejects it, the input pipeline has no meaning for it, and the simulator draws no key for it. The openness is only apparent, and it costs a fourth definition.
- **A role that is either known or `Other(name)`.** The host could carry unknown names, but `Role` stops being `Copy` in the roughly hundred places the device's pipeline matches on it, for a case no device has.
- **The crate in cli.** The device would then depend on the host's repository. The roles are the device's vocabulary, so the device's repository holds them.

## Open questions

None.
