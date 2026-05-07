# Nen fork of apache/guacamole-server

This is `getnenai/guacamole-server`, a downstream fork of
[`apache/guacamole-server`](https://github.com/apache/guacamole-server).

## Why we forked

Cup's RDP controller types into Windows desktops by sending one keydown
plus one keyup per character through guacd. Between keydown and keyup,
the controller needs to know guacd has dispatched the keydown to FreeRDP
— otherwise the two events can land in the same input-drain cycle and
Windows treats it as a zero-duration keypress, silently dropping
characters under load (see Nen's NEN-768 / Cup's `pkg/controller/desktop/rdp.go`).

Upstream guacd's only post-input signal is the `sync` instruction, which
fires only after a display batch is rendered. On idle Windows desktops
the screen does not change, no batch is rendered, no sync is sent —
and the controller's per-keystroke barrier sits at its 600 ms floor.
Result: ~3 s per character, ~21 s for a 7-char string (Nen's NEN-1341).

This fork adds a single, opt-in connection arg — `emit-input-drain` —
that causes the RDP plugin to emit a wire-level `nop` instruction
immediately after each `guac_rdp_handle_input_events()` call. Cup's
controller observes the `nop` as the input-drain barrier signal,
collapsing per-character latency from ~3 s to ~10 ms.

When the arg is unset (default `0`), this build is bit-for-bit
equivalent to upstream: zero new instructions on the wire, identical
RDP behavior, identical performance characteristics.

## Divergence policy

- **Default branch:** `nen/main`, branched off upstream tag `1.6.0`.
- **Patch set is intentionally minimal.** Three files touched:
  `src/protocols/rdp/settings.h`, `src/protocols/rdp/settings.c`,
  `src/protocols/rdp/rdp.c`. Reviewers should be able to re-derive the
  whole fork from a glance.
- **Rebase on each upstream release.** When upstream cuts 1.6.1 / 1.7.0,
  we rebase `nen/main` onto the new tag and re-tag as
  `<upstream>-nen-0.<n>`.
- **Conflicts must touch the same three files.** If a rebase produces
  conflicts elsewhere, that is a signal to revisit the patch design.
- **Do not accept unrelated patches.** Upstream-applicable changes go
  to `apache/guacamole-server` directly.

## Tags

- `1.6.0` — clean upstream base.
- `1.6.0-nen-0.1` — initial Nen patch (this release): adds
  `emit-input-drain` connection arg.

## Building

The build process is identical to upstream — see
`doc/guacamole-manual/guacamole-server-installation.md`. The Cup repo's
`docker/Dockerfile.guacd` clones this fork at the pinned tag and runs
`autoreconf -fi && ./configure && make install` against an Ubuntu 24.04
base.
