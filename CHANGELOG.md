# Changelog

This file tracks changes in the `getnenai/guacamole-server` fork only.
For upstream changes, see `apache/guacamole-server`'s own release notes.

## 1.6.0-nen-0.2 — UNRELEASED

Adds the `keepalive-interval` RDP connection argument.

### Added

- **`keepalive-interval` RDP connection argument** (integer, milliseconds).
  When set to a positive value on an RDP connection, guacd emits an
  unconditional wire-level `nop` instruction on the client socket at
  approximately the configured interval from inside the main loop, so
  legitimately-idle desktops (no display changes, no input) do not
  produce silence on the protocol stream. Pairs with the 15-second
  receive timeouts in `wwt/guac` `Stream.SocketTimeout` (Cup's Go
  controller) and `guacamole-common-js` `Tunnel.receiveTimeout` (Nen's
  browser viewer), so read-only viewers of idle Windows desktops no
  longer see a ~15-second reconnect cycle.

  The main loop wakes at least every `GUAC_RDP_MESSAGE_CHECK_INTERVAL`
  (1000ms), so values below ~1000ms coarsen to the loop wakeup rate.
  Default is `0` (disabled); with the arg unset or zero this build is
  bit-for-bit equivalent to upstream Apache 1.6.0 (plus the prior
  `emit-input-drain` patch).

  Files touched: `src/protocols/rdp/settings.h`,
  `src/protocols/rdp/settings.c`, `src/protocols/rdp/rdp.c` — the
  same three files as the prior patch, per `FORK.md`'s divergence
  policy.

  See `FORK.md` and Linear NEN-1488 for rationale.

### Verification

> ⚠️ **Not yet verified.** This patch was authored as a strawman and
> has not been built, run against guacd, or observed on the wire. The
> patch follows the exact pattern of the prior `emit-input-drain` work
> (same call site, same `guac_protocol_send_nop` API, same gating
> shape), so the code review surface is small, but a real verification
> pass is required before merging.

Expected verification once built:

- Set `keepalive-interval=5000` on an RDP connection against an idle
  Windows desktop. Observe one inbound `nop` instruction every ~5s on
  the wire (tcpdump / wireshark / guacd debug log).
- With `keepalive-interval` unset (or 0), no inbound `nop` traffic;
  build is bit-for-bit equivalent to upstream + emit-input-drain.
- Upstream Apache Guacamole web client connects to the patched guacd
  with `keepalive-interval=5000` and silently drops the inbound nops
  without error.

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
