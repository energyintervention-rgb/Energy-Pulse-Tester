# Energy-Pulse-Tester
# Energy Pulse Counter

A self-contained, offline-capable PWA for testing LED-blink pulse counters on
kWh meters. Point the phone camera at the meter's pulse LED, select the
region, run a timed test, and get a pulse count / kWh estimate with a
calibration-review chart. Single HTML file, no build step, no dependencies.

## What it does

- Live camera preview with tap-and-drag region-of-interest (ROI) selection
  over the meter's LED
- Frame-by-frame brightness sampling within the ROI (canvas-based, no
  OpenCV/native code)
- Adaptive HI/LO blink detection (see "Detection logic" below)
- Fixed-duration timed test (default 60s, configurable) with live pulse
  count and countdown
- Configurable meter constant (pulses per kWh) and sensitivity
- Auto-calibration: watches the LED for a few seconds and derives a
  sensitivity value from the observed brightness swing
- Live debug readout (brightness / baseline / delta / HI-LO state) for
  manual tuning before running a test
- Audible buzzer (Web Audio, no sound file) when a test completes
- Summary screen: pulse count, energy total, pulse log, and a signal-trace
  chart (brightness / baseline / delta / HI markers over time)
- Export: generates a standalone, fully offline HTML report (inline SVG
  charts, no external CDN) with pulse log and signal trace, downloadable
  via the browser
- Installable as a PWA (manifest + icons embedded inline as base64;
  service worker is best-effort and non-critical if unsupported)

## Deployment

**Must be served over HTTPS (or localhost) — will not work from a local
`file://` double-click.** Camera access (`getUserMedia`) is blocked by
browsers on insecure origins. Push to GitHub Pages (same pattern as the
other CTS tools) or any HTTPS static host.

No build step. It's one HTML file — copy it to the repo and it's live.

## Using it

1. Open the page, grant camera permission.
   - If autoplay is blocked (some in-app browsers / WebViews), tap the
     screen once to start the feed.
2. Tap and drag on the live preview to draw a box tightly around just the
   LED. Draw it small — a loose box picks up ambient light and hurts
   detection.
3. A debug row appears once the region is set: current brightness,
   baseline, delta threshold, and HI/LO state, updated live.
4. Tap **Auto** to auto-calibrate: it watches the LED for ~4s while it
   should be blinking, measures the brightness swing, and sets sensitivity
   from that. It reports what it measured either way — including when it
   sees no meaningful swing (check ROI placement / lighting in that case).
5. Tap the red button to start the timed test. Tap again to stop early.
6. On completion: pulse count, kWh estimate, pulse log, and the signal
   trace chart. Export to a standalone HTML report if you want to keep
   the record or hand it off.
7. Settings screen (gear icon): meter constant (pulses/kWh — check your
   meter's nameplate), test duration, sensitivity.

## Detection logic (read before trusting the numbers)

This isn't a fixed-threshold detector. It tracks a slow-moving baseline of
the "low" brightness level and looks for a jump above
`baseline + delta`, where `delta` is derived from the sensitivity slider
(more sensitive → smaller required jump). A hysteresis band (delta × 0.5)
brings it back to LOW so a single blink isn't double-counted.

This is a design choice made without hardware to test against at build
time — **not a validated algorithm**. It should track ambient light drift
reasonably well, but a slow or irregular blink rate, strong ambient
flicker (fluorescent lights), or a loosely-drawn ROI can all throw it off.
Use the live debug readout and the post-test signal trace to sanity-check
before trusting a result.

## Known limitations

- Auto-calibration is a single ~4s snapshot. If the meter's blink interval
  is longer than that window, it may not catch a full on/off cycle and
  under-calibrate. No fix implemented for this yet — would need a longer
  window or manual override if it turns out to be a problem in practice.
- Service worker registration uses a Blob URL, which not all browsers
  support for SW registration. It's wrapped in try/catch and is
  non-critical (the app works fine without it) — installability may just
  be less reliable on some browsers.
- Not tested against real meter hardware as of this build. Treat first
  real-world runs as calibration exercises, not trusted readings.

## File structure

Everything lives in one file: `energy-pulse-counter.html`. No external
assets — icons and manifest are embedded as base64 data URIs, charts are
hand-drawn inline SVG (no Chart.js/CDN dependency), and the buzzer is a
generated Web Audio tone (no sound file).
