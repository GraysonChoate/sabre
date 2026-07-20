# SABRE — Landing Page

Source of truth for the SABRE marketing site and the in-progress **hero video swap**.

> **Repo is private** and uses **Git LFS** for the large video assets. After
> cloning you must run `git lfs install` and `git lfs pull` or those `.mp4`
> files will be small text pointers, not real media. The small first-paint intro
> is intentionally stored in normal Git so hosting platforms can always serve it.

---

## The hero sequence (what we're preserving)

The experience uses a self-contained intro followed by the scroll-controlled hero.
The logic lives in the inline `<script type="text/x-dc">` at the bottom of
`index.html`.

| Phase | Role | File | Duration | Mechanism |
|-------|------|------|----------|-----------|
| **1** | Page-load **planetary zoom** | `media/sabre-intro.mp4` | 4.3s | Parser-discovered autoplay video with an immediate poster and **scroll locked**. It includes the settled handoff frame, so no second network request is needed during the transition. |
| **2** | **Parallax retail scene** | `hero-scroll.mp4` | ~8.0s | **True scroll-scrubbing** — the script eases `video.currentTime` toward a scroll-derived target (`currentTime = current + delta * ease`) inside a 420vh pinned section. Idles when you stop; moves forward/backward as you scroll. |

Other videos in the page (`media/what-we-do-*.mp4`, plus a Section-6 background
clip that is base64-inlined inside the `__bundler/manifest` in `index.html`) are
unrelated to the hero sequence.

## The task — DONE (Option A: split into the existing two slots, no JS changes)

Replaced the hero with `source-assets/new-hero-combined-4k-hevc.mp4` — a single
13.04s 4K/24fps/10-bit HEVC clip containing both phases (zoom → retail).

**Split point: 3.500s** (frame 84 of 24fps) — the exact frame where the aerial
swoop settles onto the locked storefront composition. The source was cut there
and both halves transcoded to web-safe **H.264 / 1080p / 8-bit**:

| Slot | File | Source range | Encode | Size |
|------|------|--------------|--------|------|
| Phase 1 loader | `media/sabre-loader-1080.mp4` | 0 → 3.5s | H.264 High, CRF 19, +faststart | ~5.1 MB |
| Phase 2 scrub | `hero-scroll.mp4` | 3.5 → 13.04s | H.264 High, CRF 20, **all-intra** (`keyint=1`) + faststart | ~32.8 MB |

- HEVC→H.264 because HEVC does **not** scrub reliably in Chrome/Firefox.
- The scrub is **all-intra** (every frame a keyframe) so `currentTime` scrubbing
  lands frame-accurately with no decode lag.
- Loader's last frame == scrub's first frame (verified), so the freeze-frame
  cross-fade is seamless.
- Sequence preserved by construction — **no HTML/JS changes**.

Previous originals remain in Git history (the commit before the swap) for rollback.

### Verification (`verification/`)

Rendered from the source clip (ffmpeg, ground truth) to show the full experience:

- `sequence_storyboard.png` — 9 labeled frames across the journey: LOADER
  (Earth → warp → swoop → arrival) → HANDOFF @3.5s → SCRUB (night → dawn →
  network-map lines drawing in).
- `scrub_preview.gif` — animated phase-2 scrub (3.5→13s), i.e. what scrolling reveals.
- `full_sequence.gif` — the whole 0→13s clip.

Live in-browser note: the hero scrub is **gated to `innerWidth >= 768`** (below
that the hero is intentionally static). To sanity-check locally, serve the repo
and open it in a normal-width desktop browser:
`python3 -m http.server 8099` then scroll the hero.

## Files

```
index.html                              # site (structure + loader/scrub logic + inlined manifest)
template-tail.html                      # reference: readable copy of the markup + DCLogic script
hero-scroll.mp4                         # CURRENT phase-2 scrub video (to be replaced)
media/sabre-loader-1080.mp4             # CURRENT phase-1 loader used by index.html (to be replaced)
media/sabre-loader.mp4                  # loader, full-res variant
page-loader.mp4                         # older 4s loader variant
media/what-we-do-*.mp4                  # section background videos (unrelated to hero)
media/sabre-white-logo-transparent.png
source-assets/new-hero-combined-4k-hevc.mp4   # NEW source clip (both phases, HEVC 4K, 13.0s)
```
