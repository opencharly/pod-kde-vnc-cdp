# AGENTS.md — pod-kde-vnc-cdp

The org rulebook at the umbrella (`opencharly/opencharly` `AGENTS.md`) applies in full.
This file adds only what is specific to this repo.

Canonical files:

- `charly.yml` — the `kde-vnc-cdp-session` candy (composes pod-kde-vnc's `kde-vnc-session` and
  pod-chrome-cdp's `chrome-cdp`, and declares the supervised `chrome` service on the Plasma
  socket), the `kde-vnc-cdp` image (base `podcdi.pod-cdi`), the `check-kde-vnc-cdp-pod`
  disposable bed, and the `kde-vnc-cdp-skill` routing node.
- `.agents/skills/kde-vnc-cdp/SKILL.md` — the skill body.
- `.github/workflows/tag-on-merge.yml` — installed by `bootstrap-repo-main`; never hand-edit it.
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-selkies:kde-vnc-cdp` — this repo's own skill.
- `/charly-selkies:kde-vnc`, `/charly-selkies:kde-nested`, `/charly-selkies:sway`,
  `/charly-selkies:wayvnc` — the desktop leaves this pod composes.
- `/charly-selkies:chrome`, `/charly-check:cdp` — the browser surface and its probe verb.
- `/charly-check:vnc`, `/charly-check:check` — the bed's second verb and the R10 sequence.
- `/charly-image:layer`, `/charly-image:image` — the candy and image authoring surface.
- `/charly-internals:git-workflow` — before any git/PR action.

## Rules specific to this repo

- **The GPU comes from the host through CDI.** The base is `podcdi.pod-cdi`; never compose
  `layer-nvidia` or bake a driver here (see opencharly/pod-cdi `AGENTS.md`).
- **The desktop is pod-kde-vnc's, the browser toolkit is pod-chrome-cdp's.** This repo composes
  both by a MERGED tag and adds one thing of its own: the supervised `chrome` service. Do not
  copy either candy's scripts or launcher here (R2); a missing behaviour belongs in its owner.
- **Chrome must run on `wayland-kde`, supervised, and maximized.** `wayland-0` is the compositor
  BEHIND the fullscreen Plasma window (a browser there is invisible), a compositor-driven launch
  races the supervisor, and `--start-maximized` alone is not proof on Wayland. The bed asserts all
  three (browser-side `MAXOK`, compositor-side `kdotool` geometry, and both again after a restart).
- **A VNC-frame assertion is made over the RECORDING, never a single capture.** Measured on this
  pod: after the browser's restart cycle presentation can stall while every metadata check passes —
  KWin reports the window's title/geometry/activation and CDP answers, yet a one-shot `vnc: screenshot`
  returns a complete, coherent, OLD frame (worst case measured: 3m20s stale). Assert with the VNC
  session recorder (`vnc: {method: session, action: start|stop}`) plus an `eventually:` OCR over the
  growing `frames.mjpeg`, and run the recorder AFTER the pointer/keyboard steps (a recorder running
  first was measured to swallow VNC typing). Never try to unstick the browser by un-maximizing or
  moving its window: that rewrites its normal bounds and moves the page out from under the bed's
  coordinates. Details: `.agents/skills/kde-vnc-cdp/SKILL.md`; the stall itself is
  opencharly/pod-kde-vnc-cdp#4.
- **Input in the bed goes through `vnc:`, never `wl: type` or `wl: click`.** KWin implements none
  of the wlroots input protocols; `plugin-wl` fail-fasts them, and `wtype`'s private keymap is
  re-mapped by the nested KWin so typed text arrives garbled.
- **The ambient session env is the pod's DEFAULT, and a check step still names its session.**
  `kde-vnc-session` sets `WAYLAND_DISPLAY=wayland-kde` and `KDE_SESSION_VERSION=6` for the image,
  and this pod's `chrome` service states its own `WAYLAND_DISPLAY` for the same reason a check step
  does: the invoking environment can carry a different one (measured — the runner forwards the
  operator's `WAYLAND_DISPLAY` and `DBUS_SESSION_BUS_ADDRESS` into the container). Never move the
  pair onto a service that must reach the compositor socket, and never assume the ambient value in
  a step that has to land in Plasma.
- **R10 gate:** `charly check run check-kde-vnc-cdp-pod` (the pod) and
  `charly check run check-cross-kde-cdp-pod` (the pod-to-pod CDP mechanism), both on a host with a
  GPU exposed through CDI — the second one only for its DRIVER member. A change to a composed leaf
  is verified here with `CHARLY_REPO_OVERRIDE=opencharly/<leaf>=<local tree>` before its own PR.
- **The cross-deployment bed is the org's only witness of pod→pod CDP rendering**
  (opencharly/distro-fedora#140; its Fedora predecessor was retired with #138). Do not delete it,
  and do not add Chrome to the subject image: the subject is `kde-web-subject` on the plain
  CachyOS base, and the driver reaches it by container name over the shared `charly` net.
