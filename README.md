# pod-kde-vnc-cdp — a GPU-accelerated KDE desktop over VNC, with the browser under CDP

A pod that serves a **fullscreen KDE Plasma desktop** on VNC port 5900, rendered with
**hardware OpenGL** on the host's GPU, and exposes the **Chrome** that runs inside it to
automation over the **Chrome DevTools Protocol** (9222) and to MCP clients (9224).

```
sway (headless)  ── wayland-0 ──►  wayvnc  ── :5900 ──►  your VNC client
  └─ KWin (nested, fullscreen, --socket wayland-kde)
        └─ plasmashell + KDE apps
        └─ Chrome (supervised, maximized)  ── cdp-proxy :9222 ──► cdp: checks, browser-open
                                          └─ chrome-devtools-mcp :9224 ──► MCP clients
```

- **Base:** [opencharly/pod-cdi](https://github.com/opencharly/pod-cdi). The GPU driver comes from
  the host through CDI; nothing is baked into the image.
- **Desktop:** [opencharly/pod-kde-vnc](https://github.com/opencharly/pod-kde-vnc) — headless sway,
  wayvnc, and the shared nested Plasma session.
- **Browser toolkit:** [opencharly/pod-chrome-cdp](https://github.com/opencharly/pod-chrome-cdp) —
  `chrome-wrapper`, `cdp-proxy`, `browser-open`, `chrome-restart`, and `chrome-devtools-mcp`.

## Use it

```bash
charly box build kde-vnc-cdp
charly deploy add my-desktop kde-vnc-cdp
charly config my-desktop
charly start my-desktop
# VNC on the published 5900; CDP on the published 9222; MCP on 9224
```

Set `RESOLUTION=WxH` (default `1920x1080`) to size the desktop; the maximized Chrome follows it.

Inside the pod, `$BROWSER` is `browser-open`, so a tool that opens a URL puts it in the running
browser, where the VNC user sees it.

## Proof

There are two beds. `charly check run check-kde-vnc-cdp-pod` (needs a GPU through CDI) builds and
deploys the pod, then checks that:

- the six services run and Chrome stays up under the supervisor;
- KWin is fullscreen and composites on the GPU; Chrome is a KWin client, maximized — asserted from
  the browser AND from the compositor, and again after a restart;
- every `cdp:` method answers through the 9222 proxy, and the MCP catalog answers on 9224;
- the VNC framebuffer SHOWS the page (OCR of the streamed frame) and VNC pointer + keyboard input
  DRIVE it (read back through CDP);
- the inherited keyboard path still reaches a Konsole.

`charly check run check-cross-kde-cdp-pod` proves the **pod-to-pod** path: the KDE pod's Chrome
renders the `kde-web-subject` fixture pod (a plain CachyOS base) over the shared `charly` network,
and the VNC frame shows that cross-pod page — the successor to the retired Fedora
`check-cross-pod-cdp` bed (opencharly/distro-fedora#140).
