# Moonray fork for Gaffer integration

This is a soft fork of OpenMoonRay maintained for the Gaffer port.
**Do not push patches to upstream `dreamworksanimation/openmoonray`
without explicit user approval.**

## Goal

Build Moonray against Gaffer 1.6's dep stack so binaries are ABI-compatible
in the same process:

- Python 3.11.14 (upstream Moonray expects 3.9)
- OpenUSD 26.05 (upstream Moonray uses likely 24.x or 25.x)
- Boost 1.85.0 (upstream Moonray likely 1.78–1.82)
- oneTBB 2021.13.0
- OpenImageIO 3.0.6.1 (upstream Moonray likely 2.4–2.5)
- OpenEXR 3.3.6, Imath 3.1.12, Embree 4.4.0

## Strategy

Soft fork. Keep upstream Moonray clean. Patches live in
`dependencies/moonray-patches/*.patch`, applied at build time. Rebase
against new Moonray releases periodically (every 2–3 months).

## Plan

Full port plan lives in the companion Gaffer fork:

- Local: `~/work/gaffer/contrib/moonray/PLAN.md`
- GitHub: https://github.com/jaechoidev/gaffer/blob/claude/moonray-gaffer-planning-deg3f/contrib/moonray/PLAN.md

This session owns **Phase 0 only** — porting Moonray's build to match
Gaffer's dep stack and producing a working install at `/opt/moonray-gaffer`.

## Output

When Phase 0 is complete, hand off to the Gaffer session:

1. A working `MOONRAY_ROOT` install at `/opt/moonray-gaffer` containing
   libs, headers, and DSOs that the Gaffer SConstruct can link against.
2. The patch set in `dependencies/moonray-patches/*.patch`.
3. A short status report on what was patched, what's outstanding, and
   any known limitations.

## Branching

- All work happens on `feature/gaffer-deps-port`.
- Do NOT push to `dreamworksanimation/openmoonray`. Pushes only to this
  fork.

## Companion session

Gaffer-side work happens in a separate Claude session in `~/work/gaffer`.
Coordinate via the shared filesystem (`/work/` inside the build container)
and PLAN.md updates.
