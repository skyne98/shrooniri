# Plan: shrooniri as a window into the mycelium network

Status: design, not implemented.

## Goal

A shrooniri session is a *window into the network*: windows running on any
machine in the mycelium tailnet appear in your local compositor, at LAN
latency. Exactly one client **owns** a window at a time — it may resize it and
its input goes to it. Everyone else watches. Ownership is transferable.

## What this does not need

A compositor fork. The remote windows are ordinary Wayland clients:

```
source machine                      viewer machine
  app ─▶ shrooniri ─▶ ext-image-copy ─┐
                                      │  frames (dmabuf local / PyroWave or HEVC remote)
                                      ▼
                                    proxy client ─▶ niri window
                                      ▲                  │
                                      └── input events ◀──┘  (focused surface)
```

1. **Capture (source).** `ext-image-copy-capture` on the toplevel. Already in
   shrooniri (upstream main + PR #3800). Frames stay in dmabufs on the GPU.
2. **Transport.** Same machine or same host: dmabuf fd passing, no encode.
   Remote: PyroWave (compute shader) or NVENC HEVC, then QUIC over the tailnet
   direct path — never DERP.
3. **Proxy client (viewer).** A small Rust app opening a normal niri window,
   decoding into a dmabuf and drawing it. niri resizes this surface like any
   other, which is what makes resize look local.
4. **Input.** The proxy is the focused surface, so niri hands it pointer and
   keyboard for free; it forwards them to the owner machine, which injects via
   `wlr-virtual-pointer` / virtual-keyboard. No multi-seat work.
5. **Resize.** Proxy surface resize → new size sent to the source → applied with
   niri IPC (`set-window-width`, `expand-column`).
6. **Ownership.** mycelium `stream` (Raft, linearizable) holds the ownership
   token; `map` (CRDT) holds presence and metadata. Acquiring the pen is rare
   — consensus fsync cost is irrelevant there. **Input events never go through
   Raft**; after acquiring ownership they go peer-to-peer.

## Hard parts, in rank order

1. **Popups and menus.** `ext-image-copy` captures one toplevel; a remote GTK
   context menu or tooltip is a separate surface and will not appear. Options:
   capture window + composited popups into one image (compositor feature), or
   proxy each popup as its own surface.
2. **Encode fan-out.** "Everyone watches everything" means one encode per
   viewer, or a relay that fans out one encode. PyroWave avoids NVENC's
   session limit (RTX 3090: 3 concurrent), but burns SMs per stream.
3. **Latency budget.** LAN raw/PyroWave is fine; WAN needs a jitter buffer and a
   bandwidth estimate. Budget: capture 2 ms, encode 3–6 ms, network 1–30 ms,
   decode 2 ms, present 1 frame.
4. **Escape hatch.** Compositor shortcuts (Super+D, Super+1) never reach the
   remote app. Needs an explicit "all keys go remote" toggle plus a local
   escape chord.
5. **Security.** Forwarding input is remote code execution with extra steps.
   Identity from mycelium's TLS client certs; ownership must be authenticated,
   and a stale owner must not be able to inject after handover.

## Codec ladder

Pick like a video site, not like a menu of strings. Host hardware matters:
RTX 3090 is Ampere — **no AV1 encode**, so the ladder ends at HEVC.

| Step | Codec | Encoder | Use |
| --- | --- | --- | --- |
| 0 | PyroWave 4:4:4 | Vulkan compute shaders | desktop and text, full chroma |
| 1 | PyroWave 4:2:0 | Vulkan compute shaders | motion, lower bandwidth |
| 2 | HEVC Main10 4:2:0 | NVENC (1 of 3 sessions) | fallback, other clients |
| 3 | H.264 High | NVENC | compatibility floor |

Selection: rolling bandwidth estimate plus content class (idle desktop →
4:4:4, scrolling or video → 4:2:0), and never downgrade during a burst of
input. The two encoder families are different resources — PyroWave burns SMs,
NVENC burns session slots — so "what else is streaming from this host" is part
of the decision, not just bandwidth.

Switching quality:
- **At session start**: already possible (Moonlight client codec + bitrate,
  Pyroshine `stream.video.encoder` per app). Expose it in `aurora`'s UI.
- **Mid-stream**: a codec change tears down and re-initialises the encode
  pipeline. *Fast restart* (0.3–1 s black frame) is what Moonlight does today.
  *Hot handoff* — keep a second encoder warm, cut over on an IDR boundary,
  client preloads the decoder — needs a `renegotiate` control message in the
  Sunshine protocol and decoder work in aurora.

## Milestones

- **M0 (one weekend)** one window, one machine, LAN, raw frames, one viewer,
  input forwarded, ownership fixed to one client. Proves capture → present →
  input round trip.
- **M1** remote encode: PyroWave 4:4:4, bandwidth estimate, quality ladder at
  session start.
- **M2** mycelium ownership: token in `stream`, presence in `map`, handover,
  stale-owner rejection.
- **M3** hot codec handoff and multi-viewer relay.
- **M4** popup proxying.

## Open questions

- Capture popups: compositor-side composition, or one proxy surface per popup?
- Should the proxy window live in a dedicated column, or float?
- Do we expose remote windows through the compositor's IPC as if they were
  local toplesels, so the bar shows them like any other window?
- Where does the input event log live — mycelium `stream` for audit, but only
  after the fact, never on the hot path.
