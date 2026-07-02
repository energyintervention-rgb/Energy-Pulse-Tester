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
4. Tap **Auto** to auto-calibrate: it watches the LED for the configured
   calibration window (default 4s, adjustable in Settings — see below)
   while it should be blinking, measures the brightness swing, and sets
   sensitivity from that. It reports what it measured either way —
   including when it sees no meaningful swing (check ROI placement /
   lighting in that case).
5. Tap the red button to start the timed test. Tap again to stop early.
6. On completion: pulse count, kWh estimate, pulse log, and the signal
   trace chart. Export to a standalone HTML report if you want to keep
   the record or hand it off.
7. Settings screen (gear icon): meter constant (pulses/kWh — check your
   meter's nameplate), test duration, sensitivity, and auto-calibration
   window (1–30s — must be long enough to catch one full blink cycle of
   your meter's LED; shorten it for fast-blinking meters, lengthen it for
   slow ones).

## Modes

**Test mode** (default) — runs for a fixed duration (Settings), then shows
a summary with the signal-trace chart, as described above.

**Collector mode** — runs continuously until you stop it, for use as a
comparison check against the real Energy Eye device's hourly readings.
No fixed duration, no signal-trace chart (skipped to avoid unbounded
memory growth over a long run), just a running pulse count and elapsed
time. Every detected pulse is logged with a timestamp. Stop it whenever
you want a reading, then export.

Switch modes in Settings. The toggle is disabled while a run is in
progress — stop first, then switch.

**Important limitation, not a bug I can fix in this app:** a browser tab
loses camera access and gets throttled or paused by the phone/OS the
moment the screen locks or the tab leaves the foreground. Collector mode
only works while the device stays awake and the tab stays in view the
whole time. It is not a substitute for the actual device's unattended
hourly reporting — treat it strictly as a side-by-side comparison tool
for while you're present and watching it.

## CSV export

Available from the summary screen after any run (Test or Collector).
Columns: `timestamp` (ISO 8601, absolute), `elapsed_seconds`,
`pulse_number`, `cumulative_kWh`. A few `#`-prefixed metadata lines
(mode, meter constant, generated-at) sit above the header row — standard
CSV readers and Excel/Sheets tolerate this fine, but if you're parsing it
programmatically, skip lines starting with `#`.

For long Collector runs, the on-screen pulse-log table caps at the most
recent 200 rows for display performance — the CSV export always contains
the complete log regardless of what's shown on screen.

There's no auto-upload to Google Drive or automatic hands-off saving —
every export is a manual button tap that triggers a browser download.
I looked at Drive auto-upload and decided against building it: it needs
a Google Cloud OAuth client you'd have to set up yourself (tied to your
Google account, so I can't create it for you), and browsers can't
securely hold long-lived login tokens, so "hands-off forever" isn't
something I could honestly promise would keep working unattended.

## Works on desktop/laptop, not just phones

The camera and ROI-selection code uses standard `getUserMedia` + canvas +
mouse/touch events — nothing phone-specific. It should work the same way
in a desktop browser with a webcam. The one mobile-oriented detail is
`facingMode: 'environment'` in the camera request, which just asks for a
rear camera *if available*; it's a soft "ideal" hint, not a hard
requirement, so on a laptop with a single webcam it's ignored and that
webcam is used normally.


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

## GPS and device time

Each run requests the device's GPS position once, at the moment you tap
start (non-blocking — the test starts immediately, location fills in
when it resolves). It does not poll continuously; a stationary meter
test doesn't need a moving position, so one reading per run is what's
captured.

**On timestamp accuracy — this does not make timestamps more accurate.**
Browser Geolocation returns a position and a "time" for that position,
but that time is derived from the device's own system clock, not read
directly from the GPS satellite signal. The CSV timestamp already comes
from the same system clock via `Date.now()`. Adding GPS gets you *where*
a reading was taken, not a more accurate *when*. I don't have solid
information on whether some phone/OS combinations occasionally sync
their system clock using GPS timing in the background — if that happens
it would already be reflected in the system clock the app uses either
way, not something gained by calling the Geolocation API separately.

**Where it shows up:**
- Summary screen: a GPS status line (coordinates + accuracy, or an
  "unavailable" reason if it failed/was denied)
- CSV export: a `# gps:` metadata comment line, plus `latitude`,
  `longitude`, `gps_accuracy_m` columns on every row, plus a
  `device_local_time` column (human-readable, alongside the existing
  ISO/UTC `timestamp_utc` column)
- HTML report export: same GPS line as the summary screen

**Known limitation:** GPS accuracy on phones without a clear sky view
(indoors, meter rooms, basements) is often poor or unavailable — this is
a general, well-established property of GPS, not something specific to
this app. On laptops without GPS hardware, the browser typically falls
back to WiFi/IP-based location, which can be much less precise. Expect
the accuracy figure shown to vary a lot by device and location, and
expect it to sometimes fail entirely indoors — the app reports that
honestly ("unavailable...") rather than guessing.

## Known limitations

- Auto-calibration is a single snapshot over a fixed window (adjustable
  in Settings, default 4s). If the meter's blink interval is longer than
  the chosen window, it may not catch a full on/off cycle and
  under-calibrate — lengthen the window in that case. There's no
  automatic detection of "window too short"; it's on you to judge from
  the meter's known blink rate or from a failed/implausible calibration
  result.
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
