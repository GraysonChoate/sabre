# SABRE — Landing Page

Source of truth for the SABRE marketing site and the in-progress **hero video swap**.

> **Repo is private** and uses **Git LFS** for all video assets and `index.html`.
> After cloning you must run `git lfs install` and `git lfs pull` or the `.mp4`
> files (and `index.html`) will be small text pointers, not real media.

---

## The hero sequence (what we're preserving)

The intro is built from **two separate video files** playing two distinct roles.
The logic lives in the inline `<script type="text/x-dc">` at the bottom of
`index.html`.

| Phase | Role | File | Duration | Mechanism |
|-------|------|------|----------|-----------|
| **1** | Page-load **planetary zoom** | `media/sabre-loader-1080.mp4` | ~4.0s | Autoplays fullscreen in an overlay with **scroll locked**. On end it cross-fades to the hero; its last frame is matched to hero frame 0 so it reads as one continuous shot. |
| **2** | **Parallax retail scene** | `hero-scroll.mp4` | ~8.0s | **True scroll-scrubbing** — the script eases `video.currentTime` toward a scroll-derived target (`currentTime = current + delta * ease`) inside a 420vh pinned section. Idles when you stop; moves forward/backward as you scroll. |

Other videos in the page (`media/what-we-do-*.mp4`, plus a Section-6 background
clip that is base64-inlined inside the `__bundler/manifest` in `index.html`) are
unrelated to the hero sequence.

## The task

Replace the hero with `source-assets/new-hero-combined-4k-hevc.mp4` — a **single
13.0s 4K HEVC clip that contains both phases** (zoom → retail) back-to-back.

### Plan (Option A — split into the existing two slots; no JS changes)

1. Find the cut frame where the planetary zoom settles into the retail space
   (old split was ~4.0s / ~8.0s).
2. Split the source clip at that point.
3. **Transcode both pieces to H.264 MP4** (the source is HEVC/H.265, which does
   **not** scrub reliably in Chrome/Firefox), likely downscaled 4K→1080p.
4. Re-encode the scroll segment with a **dense keyframe interval** so
   `currentTime` scrubbing stays smooth.
5. Drop the results in as `media/sabre-loader-1080.mp4` (phase 1) and
   `hero-scroll.mp4` (phase 2). Sequence is preserved by construction.

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
