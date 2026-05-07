# Changelog

This file tracks changes in the `getnenai/guacamole-server` fork only.
For upstream changes, see `apache/guacamole-server`'s own release notes.

## 1.6.0-nen-0.1 — 2026-05

Initial Nen patch on top of upstream Apache Guacamole 1.6.0.

### Added

- **`emit-input-drain` RDP connection argument.** When set to `true` on
  an RDP connection, guacd emits a wire-level `nop` instruction
  immediately after every `guac_rdp_handle_input_events()` iteration in
  its main loop. Cup's RDP controller (Nen) uses the inbound nop as a
  precise per-keystroke barrier signal, eliminating the screen-redraw
  dependency of the upstream `sync`-based barrier on idle Windows
  desktops. Default is `false`; with the arg disabled this build is
  bit-for-bit equivalent to upstream Apache 1.6.0.

  Files touched: `src/protocols/rdp/settings.h`,
  `src/protocols/rdp/settings.c`, `src/protocols/rdp/rdp.c`.

  See `FORK.md` for the full rationale and the linked Linear issue
  (NEN-1341).

### Verification

- Manual: built the image, launched against a Nen Windows desktop with
  `emit-input-drain=true`, observed inbound `nop` traffic on the wire
  (one `nop` per main-loop iteration). With the arg disabled, no `nop`
  traffic. Per-character `type` latency dropped from ~3 s/char to
  ~10 ms/char on idle Windows.
- Manual: ran upstream Apache Guacamole web client against the patched
  guacd; web client logs no errors and silently drops the inbound
  `nop` instructions, as expected.
