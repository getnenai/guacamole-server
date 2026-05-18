# Changelog

This file tracks changes in the `getnenai/guacamole-server` fork only.
For upstream changes, see `apache/guacamole-server`'s own release notes.

## 1.6.0-nen-0.2 — UNRELEASED

Adds the `keepalive-interval` RDP connection argument.

### Added

- **`keepalive-interval` RDP connection argument** (integer, milliseconds).
  When positive, guacd emits a distinct `nen-keepalive` instruction on
  the client socket at ~the configured interval from the main loop **and
  explicitly flushes the client socket**. Two essentials:
  - **Distinct opcode, not `nop`:** the `emit-input-drain` patch also
    sends a bare `nop`, and Cup's `bring` client treats any inbound `nop`
    as the NEN-768 input-drain barrier signal. A keepalive `nop` landing
    mid-keystroke would prematurely release that barrier and silently
    drop/reorder input, so the keepalive uses a separate opcode that
    conformant clients ignore — decoupling the two signals by
    construction.
  - **Flush is essential, not cosmetic:** `client->socket` only flushes
    on a frame/draw boundary, so on a genuinely idle desktop an unflushed
    instruction never leaves the guacd child and idle viewers are dropped
    as "not responding".
  Pairs with the 15-second receive timeouts in `wwt/guac`
  `Stream.SocketTimeout` (Cup's Go controller) and `guacamole-common-js`
  `Tunnel.receiveTimeout` (Nen's browser viewer), so read-only viewers of
  idle Windows desktops no longer see a ~15-second reconnect cycle.

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

**Compile verified.** Built end-to-end against the cup repo's
`docker/Dockerfile.guacd` (the production build chain — Alpine 3.18 +
FreeRDP from source with `WITH_KRB5=ON` + autobuild.sh against the
guacamole-server tree). Build completed cleanly under
`-Werror -Wall -pedantic`, so no warnings on the patched files.
guacd binary, libguac-client libraries, and DEPENDENCIES manifest all
produced successfully.

**Runtime verified (2026-05-18, cup-sandbox-dev).** Built as
`cup-guacd:nen1488-fix-a9241994` and rolled to the controller pool
(`cup-sandbox-dev-controller:17`) with the controller passing
`keepalive-interval=5000`. On an idle read-only viewer
(`dsk_33ecb184345ccd6b54e6ca95`, connID `$76d23e1b`): the controller's
per-viewer guacd socket received a `nop` **every ~5s, unbroken, for
~6 minutes of idle**, as small lone-nop chunks (n≈36–54);
**zero** guacd `User is not responding`; **no** tunnel reconnect churn.
Without the flush (strawman, same session): nops reached idle viewers
only on sporadic screen-draw boundaries and guacd dropped the viewer
every ~20–35s, producing the reconnect loop. With `keepalive-interval`
unset (0) this block is a single integer compare — bit-for-bit
equivalent to `1.6.0-nen-0.1`. Full record: Linear NEN-1488.

**Note:** the 2026-05-18 verification used the earlier `nop` keepalive
and exercised only the idle path. The keepalive opcode was subsequently
changed to the distinct `nen-keepalive` (to stop aliasing the
`emit-input-drain` `nop`); idle behaviour is unchanged (any inbound
instruction resets the timers) and decoupling is proven deterministically
by the cup-side `bring` unit test. Concurrent-typing re-verification on
sandbox-dev is tracked as a release gate before this tag is rolled.

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
