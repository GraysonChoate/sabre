# Web Video Production Playbook

Use this document as a reusable standard for websites that contain cinematic intros,
scroll-controlled footage, ambient loops, interactive CTA animations, or other video-led
experiences. It is framework-agnostic and may be copied into a repository as-is.

## Agent Directive

Treat every editing export as a master asset, not a web deliverable. Do not declare a
video-heavy page complete until desktop and mobile delivery files exist, first paint is
protected by a poster or branded loading state, cold-cache behavior is tested, and all
media passes the verification checklist in this document.

## Project Variables

Complete this table before implementation.

| Variable | Project value |
| --- | --- |
| Production URL | `[URL]` |
| Deployment platform | `[VERCEL / NETLIFY / CLOUDFLARE / OTHER]` |
| Desktop breakpoint | `[EXAMPLE: 768px]` |
| Primary mobile devices | `[IOS SAFARI / ANDROID CHROME / BOTH]` |
| Intro duration | `[SECONDS]` |
| Intro handoff frame | `[SECONDS OR FRAME NUMBER]` |
| Hero scroll range | `[START TIME]` to `[END TIME]` |
| Source frame rate | `[24 / 25 / 30 / 60]` |
| Mobile framing | `[LANDSCAPE CROP / PORTRAIT CROP / SHARED]` |
| Media host | `[STATIC HOST / CDN / VIDEO PLATFORM]` |
| Large-file strategy | `[GIT LFS / OBJECT STORAGE / VIDEO CDN]` |

## Video Roles

Classify every asset before encoding because each role has different requirements.

| Role | Playback behavior | Loading behavior | Keyframe requirement |
| --- | --- | --- | --- |
| First-paint intro | Plays once before the page | Immediate | Normal GOP |
| Scroll-scrubbed hero | `currentTime` follows scroll | Preload before reveal | Keyframe every 0.20-0.30s |
| Ambient loop | Loops while visible | Lazy-load near viewport | Normal GOP |
| Hover or CTA activation | Plays on interaction | Load shortly before viewport | Normal GOP |
| Decorative background | Loops while visible | Lazy-load | Normal GOP |

Do not use animated GIF for cinematic footage. Use MP4, WebM, or a modern image format
for simple loops. GIF files are usually larger, lower quality, and less efficient.

## Required Deliverables

Every important animation should have:

- An untouched editing master stored outside the public website bundle.
- A desktop web MP4.
- A mobile web MP4 with deliberate mobile framing.
- A lightweight poster matching the first visible frame.
- H.264 video in an MP4 container for the broadest compatibility.
- `yuv420p` pixel format for reliable browser and mobile decoding.
- Fast-start metadata so playback can begin before the download completes.
- No audio stream when the animation is silent.

## Recommended Size Budgets

These are delivery targets, not excuses to accept visible compression artifacts.

| Asset | Preferred mobile size | Preferred desktop size |
| --- | ---: | ---: |
| First-paint intro under 6 seconds | 2-4 MB | Under 8 MB |
| Scroll hero around 10 seconds | Under 8 MB | Under 20 MB |
| Ambient or section loop | Under 5 MB | Under 8 MB |
| Poster image | Under 250 KB | Under 400 KB |

If acceptable visual quality cannot fit within these budgets, use a video CDN or adaptive
streaming rather than damaging the animation.

## Naming Convention

```text
media/
  intro-desktop.mp4
  intro-mobile.mp4
  intro-poster.jpg
  hero-scroll-desktop.mp4
  hero-scroll-mobile.mp4
  hero-poster.jpg
  section-name-loop-desktop.mp4
  section-name-loop-mobile.mp4
  section-name-poster.jpg
```

Avoid spaces, duplicate extensions, opaque export IDs, and names such as `final-final-v7`.

## Standard FFmpeg Export

The following recipe creates a high-quality 1280x720 mobile delivery file from a 16:9
master. Change the dimensions only after deciding how the composition should crop.

```bash
ffmpeg -y -i INPUT_MASTER.mp4 \
  -vf "scale=1280:720:flags=lanczos,fps=24" \
  -c:v libx264 -preset slow -crf 22 \
  -profile:v high -level 4.0 -pix_fmt yuv420p \
  -movflags +faststart -an \
  OUTPUT-mobile.mp4
```

For a desktop version, use 1920x1080 and begin quality testing around `-crf 19` to
`-crf 21`. Lower CRF means higher quality and a larger file.

### Preserve the Entire Frame

Use this when letterboxing is acceptable and nothing may be cropped.

```bash
-vf "scale=1280:720:force_original_aspect_ratio=decrease:flags=lanczos,pad=1280:720:(ow-iw)/2:(oh-ih)/2:black,fps=24"
```

### Full-Bleed Crop

Use this when the video must completely fill a 16:9 canvas.

```bash
-vf "scale=1280:720:force_original_aspect_ratio=increase:flags=lanczos,crop=1280:720,fps=24"
```

Create a dedicated portrait render, such as 720x1280, when a landscape center crop loses
the subject or key visual information. Do not assume CSS `object-fit: cover` is sufficient.

## Scroll-Scrubbed Export

Scroll-controlled video requires frequent keyframes. At 24 fps, a six-frame GOP creates a
seek point every 0.25 seconds.

```bash
ffmpeg -y -i INPUT_MASTER.mp4 \
  -vf "scale=1280:720:flags=lanczos,fps=24" \
  -c:v libx264 -preset slow -crf 22 \
  -profile:v high -level 4.0 -pix_fmt yuv420p \
  -g 6 -keyint_min 6 -sc_threshold 0 \
  -movflags +faststart -an \
  OUTPUT-scroll-mobile.mp4
```

Use `-g 6` at 24 fps, `-g 8` at 30 fps, or calculate approximately `FPS x 0.25`.
Long GOP files may play normally but will jump, lag, or decode slowly when scrubbed.

## Poster Export

Choose a frame that visually matches what the visitor should see before playback begins.

```bash
ffmpeg -y -ss 0.10 -i INPUT.mp4 \
  -frames:v 1 -vf "scale=1280:-2,format=yuvj420p" \
  -q:v 3 poster.jpg
```

Convert the JPEG to WebP or AVIF with the project's image pipeline when available. Do not
assume every FFmpeg installation includes a WebP or AVIF encoder.

## Responsive First-Paint Markup

Place the mobile source before the desktop source. The media query also catches touch-first
tablets that may have a wider viewport.

```html
<video
  id="introVideo"
  muted
  playsinline
  webkit-playsinline
  preload="auto"
  poster="media/intro-poster.jpg"
  disablepictureinpicture
>
  <source
    src="media/intro-mobile.mp4"
    type="video/mp4"
    media="(max-width: 767px), (hover: none), (pointer: coarse)"
  >
  <source src="media/intro-desktop.mp4" type="video/mp4">
</video>
```

Set `muted`, `defaultMuted`, and `playsInline` before calling `play()`. Only reveal the
video after the `play()` promise resolves. Until then, keep the poster or branded loading
state visible so the visitor never sees an empty black element.

## First-Paint Loading Sequence

Use this order:

1. Render HTML, background color, poster, logo, and progress treatment immediately.
2. Select the correct mobile or desktop source before calling `video.load()`.
3. Buffer the short intro before starting playback.
4. Start preloading the hero after intro playback begins.
5. Reveal the intro only after `video.play()` succeeds.
6. When the intro ends, place the hero on its exact handoff time.
7. Reveal the page after the hero has metadata and enough opening footage buffered.
8. Include a safety timeout so media failure cannot permanently lock scrolling.
9. Dispatch a page-ready event so lower media waits until first paint is complete.

Recommended starting thresholds:

- Desktop short intro: buffer approximately 90-99% before playback.
- Mobile short intro: buffer approximately 60-80% before playback.
- Mobile streaming exception: allow playback when `readyState >= 4` and at least half is buffered.
- Hero handoff: buffer several seconds past the handoff point before removing the intro.
- Safety timeout: approximately 10-15 seconds, followed by a poster-backed fallback.

Do not use a nearly 100% mobile buffer requirement. Some mobile browsers stop reporting
progress before that threshold even though playback is ready.

## Below-the-Fold Video Markup

```html
<video data-lazy-video muted playsinline loop preload="none" poster="media/section-poster.jpg">
  <source data-src="media/section-loop.mp4" type="video/mp4">
</video>
```

Do not let every video compete with the intro during first paint. Assign `src`, call
`load()`, and attempt playback only when the section approaches the viewport.

## Lazy-Loading Pattern

```js
const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    const video = entry.target;

    if (entry.isIntersecting) {
      const source = video.querySelector('source[data-src]');
      if (source && !source.src) {
        source.src = source.dataset.src;
        video.load();
      }

      video.muted = true;
      const playback = video.play();
      if (playback?.catch) playback.catch(() => {});
    } else {
      video.pause();
    }
  });
}, { rootMargin: '100% 0px', threshold: 0.01 });

document.querySelectorAll('video[data-lazy-video]').forEach((video) => {
  observer.observe(video);
});
```

## Scroll-Scrubbing Rules

- Calculate a target time from normalized section scroll progress.
- Update through `requestAnimationFrame`, never directly for every scroll event.
- Smooth the current time toward the target instead of making large jumps.
- Do not issue another seek while `video.seeking` is true.
- Throttle seeks to approximately 30-40 milliseconds.
- Ignore differences smaller than approximately 0.01 seconds.
- Stop scheduling frames when the target has settled.
- Keep copy animation and video timing derived from the same normalized progress value.
- Preserve the final copy state until the visitor scrolls backward.
- Use a continuous lightweight loop or simpler visual fallback on mobile when full scrubbing
  cannot maintain the target frame rate.

## Autoplay Rules

- Autoplay video must be muted.
- Use both `playsinline` and `webkit-playsinline`.
- Handle rejection from `video.play()` without throwing an uncaught error.
- Retry visible ambient media on `canplay` or the first intentional user interaction.
- Do not rely on autoplay for essential information. Keep a meaningful poster fallback.

## Reduced-Motion Rules

When `prefers-reduced-motion: reduce` is active:

- Skip cinematic intro playback or reveal it as a static poster.
- Disable scroll scrubbing and decorative loops.
- Preserve all copy, links, and calls to action.
- Never leave content hidden because its reveal animation was disabled.

## Git and Storage Decision

Use normal Git only for small first-paint assets that must always exist in the deployment.
Use Git LFS or external storage for large media.

Example `.gitattributes`:

```gitattributes
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.mov filter=lfs diff=lfs merge=lfs -text

# Optional: keep small first-paint files in normal Git when the host does not hydrate LFS.
media/intro-desktop.mp4 -filter -diff -merge -text
media/intro-mobile.mp4 -filter -diff -merge -text
```

Before using this exception, confirm that the intro is small enough to avoid long-term
repository bloat. At larger scale, host first-paint media on a properly cached CDN instead.

Do not assume an object-storage service automatically provides good video delivery. Confirm
range requests, caching, content type, cross-origin policy, and global delivery behavior.

Consider Mux, Cloudinary, Cloudflare Stream, or another video platform when:

- Individual assets exceed roughly 20-30 MB.
- Videos are long-form or numerous.
- Multiple resolutions or adaptive streaming are needed.
- Playback analytics, transformations, captions, or global optimization are required.

## Technical Verification Commands

Inspect codec, dimensions, frame rate, duration, size, pixel format, and bitrate:

```bash
ffprobe -v error \
  -show_entries format=duration,size,bit_rate:stream=codec_name,width,height,r_frame_rate,pix_fmt \
  -of default=nw=1 OUTPUT.mp4
```

Count keyframes in a scroll-scrubbed file:

```bash
ffprobe -v error -select_streams v:0 -skip_frame nokey \
  -show_entries frame=best_effort_timestamp_time -of csv=p=0 OUTPUT.mp4 | wc -l
```

Decode the complete file and report errors:

```bash
ffmpeg -v error -i OUTPUT.mp4 -f null -
```

Confirm fast-start atom order. `moov` should appear before `mdat`:

```bash
grep -abo 'moov\|mdat' OUTPUT.mp4 | head -2
```

Check the deployed response:

```bash
curl -I https://example.com/media/OUTPUT.mp4
curl -I -H 'Range: bytes=0-1023' https://example.com/media/OUTPUT.mp4
```

The deployed URL must return actual video bytes with the correct `video/mp4` content type.
A range request should normally return `206 Partial Content`. Never deploy a Git LFS pointer
text file in place of the MP4.

## Cold-Load Test Matrix

Complete every applicable test before launch.

- Fresh incognito browser with no warm cache.
- Hard reload with browser cache disabled.
- Throttled mobile connection.
- Real iPhone Safari.
- Android Chrome or an equivalent real device.
- Desktop Chrome.
- Desktop Safari.
- Narrow portrait viewport.
- Wide touch-first tablet viewport.
- Reduced-motion preference enabled.
- Page opened directly at an anchor or deep link.

## Visual Acceptance Criteria

- A poster or branded state appears immediately.
- No unexplained black screen appears before video playback.
- Intro begins at frame zero and plays without freezing.
- The handoff frame matches the first hero frame.
- The transition does not change brightness, crop, scale, or color unexpectedly.
- Mobile framing preserves the intended subject and negative space.
- Scroll video moves smoothly in both directions.
- Copy remains readable against every video frame.
- Copy does not disappear when the visitor reaches the end state.
- Links remain clickable after animations settle.
- Lower videos do not delay first paint.
- Offscreen loops pause and resume correctly.
- No media element changes page dimensions when it loads.

## Warning Signs and Likely Causes

| Symptom | Likely cause |
| --- | --- |
| Works after refresh only | Experience depends on a warm cache |
| Black screen before intro | Missing poster, hidden loader, or autoplay rejection |
| Intro is skipped on mobile | Buffer threshold is too strict or safety timeout fires first |
| Intro freezes and then becomes choppy | Playback starts before sufficient buffering |
| Scroll footage jumps | GOP is too long or keyframes are too sparse |
| Scroll footage feels delayed | Too many seeks, no RAF smoothing, or oversized resolution |
| Host displays loading or unpacking behavior | Media is oversized or incorrectly delivered through LFS |
| Mobile fails while desktop works | Wrong source selected, autoplay attributes missing, or file too heavy |
| Page downloads everything immediately | Below-fold videos use eager preload |
| Deployed file is only a few text lines | Git LFS pointer was deployed instead of the binary |
| Video looks soft | CRF too high, wrong scale, or repeated re-encoding |

## Go/No-Go Checklist

- [ ] Editing masters are excluded from the public site bundle.
- [ ] Desktop and mobile web variants exist.
- [ ] Mobile framing has been visually approved.
- [ ] Posters exist and match the opening state.
- [ ] MP4 files use H.264 and `yuv420p`.
- [ ] Silent files contain no audio stream.
- [ ] Fast-start metadata is present.
- [ ] Scroll files have a keyframe approximately every 0.20-0.30 seconds.
- [ ] First-paint videos are available from the production host.
- [ ] Lower media uses `preload="none"` and viewport-based loading.
- [ ] Offscreen videos pause.
- [ ] Autoplay rejection is handled.
- [ ] Reduced-motion behavior is complete.
- [ ] Incognito cold load passes.
- [ ] Real mobile device testing passes.
- [ ] Scroll playback passes in both directions.
- [ ] Browser console contains no media errors.
- [ ] Deployed range request returns the expected response.
- [ ] Final asset sizes are documented.

## Agent Handoff Template

Complete this block whenever video work is delivered.

```markdown
### Video Delivery Summary

Production URL: [URL]
Deployment commit: [COMMIT]

| Asset | Role | Desktop size | Mobile size | Duration | Resolution | Keyframes |
| --- | --- | ---: | ---: | ---: | --- | ---: |
| [NAME] | [INTRO / SCRUB / LOOP / CTA] | [MB] | [MB] | [SEC] | [SIZE] | [COUNT] |

First-paint strategy: [DESCRIPTION]
Mobile fallback: [DESCRIPTION]
Reduced-motion behavior: [DESCRIPTION]
Storage strategy: [GIT / LFS / CDN]

Tests completed:
- [ ] Cold incognito load
- [ ] Throttled connection
- [ ] iPhone Safari
- [ ] Android Chrome
- [ ] Desktop Chrome
- [ ] Desktop Safari
- [ ] Scroll and reverse-scroll
- [ ] Deployed range request

Known limitations: [NONE OR DESCRIPTION]
```

## Completion Rule

A video-heavy website is not complete merely because it works on the creator's computer.
It is complete when a first-time visitor on a mobile connection receives the correct asset,
sees an immediate intentional first frame, experiences smooth playback, and can still use
the complete interface if video playback fails.
