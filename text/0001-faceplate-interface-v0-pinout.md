# RFC 0001: Faceplate interface v0 pinout, defined by function

- **Status:** Accepted (2026-10-22, decided by Claude under the maintainers' delegation)
- **Date:** 2026-10-22
- **Affected repositories:** t1 (faceplate interface spec), board (pin plan, BOM), faceplate-touch, faceplate-cassette, faceplate-classic

## Summary

The faceplate interface spec in t1 said "Pinout: not defined yet", while the board docs carried the faceplate connector allocation with SoC names. This RFC moves the faceplate interface pinout into the t1 spec as interface v0, defined by function, and adds the three lines of the RGB status LED (FPT-025).

## Motivation

- The faceplate interface belongs to t1 (t1 ADR 0001); the board maps its SoC pins onto it, like the HAT interface (HCS-021).
- BOM v1.2 replaced the single PWM status LED line with an RGB LED driven by an LP5521 on the main board (FPT-025, FPT-016), which needs three lines on the connector.

## Proposal

- **t1:** the faceplate interface spec gains a "Flat-cable connector, interface v0" section: 51 contacts (Hirose FH35C-51S, 0.3 mm), 46 used and 5 reserved, by functional group, with electrical notes. No SoC names.
- **board:** the pin plan §4.1 maps the RK3566 onto that table and drops its own allocation; BOM line 71 points to the t1 spec.
- **faceplates:** each faceplate implements the groups it needs; unused groups stay unconnected on the faceplate side. The status LED is passive on the faceplate (three anodes in, cathodes to ground).
- **Compatibility:** v0 is the first defined pinout; there is nothing earlier to stay compatible with.

## Alternatives

- Keep a single PWM line and a single-colour LED: rejected by FPT-025 when an RGB part fits, which it does.
- Put the LED driver on the faceplate and control it over I2C: rejected because it adds an active part to every faceplate for one indicator, and the board driver serves every faceplate.

## Open questions

- Contact positions are fixed in pinout v1, together with the DSI pair placement and ground guards.
- FH35C contact voltage rating for the backlight string (≤ 16.5 V) is **UNVERIFIED**.
