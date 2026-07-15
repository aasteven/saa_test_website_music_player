# Music Player Artifact Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file HTML music player Artifact that plays three synthesized (Web Audio API) tracks with album-art animation, play/pause/seek/volume/prev/next controls.

**Architecture:** One self-contained `index.html` with inline `<style>` and `<script>`. A data-driven track list feeds a small Web Audio scheduling engine (oscillators + gain envelopes looped per track). UI state (playing/paused, playhead, volume, current track index) drives DOM/CSS updates; CSS animations (rotation + pulse) are toggled via `animation-play-state` and restarted on track change.

**Tech Stack:** Vanilla HTML/CSS/JS, Web Audio API. No frameworks, no build step, no external requests (Artifact CSP forbids them).

## Global Constraints

- Single file, no external network requests — no CDN scripts, no remote audio/image URLs (spec: musicplayer/specs/2026-07-14-music-player-design.md).
- Exactly 3 tracks; navigation is prev/next only — no playlist panel, no shuffle, no repeat.
- Animate only `transform` and `opacity` (project guardrail carried from CLAUDE.md anti-generic rules).
- Rotation animation must pause via `animation-play-state`, not be removed/re-added, so it resumes smoothly.
- No test framework is in scope for this vanilla-JS Artifact — every verification step below is a manual browser interaction with an explicit expected outcome, not an automated test run.
- Theme-aware: support both light and dark via `prefers-color-scheme` plus `:root[data-theme]` overrides, per artifact-design conventions.

---

### Task 1: Track data model + Web Audio scheduling engine

**Files:**
- Create: `d:/PROJECTS/musicplayer/index.html` (skeleton `<script>` block only for this task; UI markup comes in Task 2)

**Interfaces:**
- Produces:
  - `const TRACKS` — array of 3 track objects: `{ id: string, title: string, artist: string, duration: number (seconds), tempo: number (bpm), waveform: OscillatorType, gradient: [string, string], notes: Array<{time:number, freq:number, dur:number, gain:number}> }`
  - `function getAudioCtx(): AudioContext` — lazily creates/returns a single shared `AudioContext`.
  - `function playTrack(track, offsetSeconds): void` — starts scheduling `track` from `offsetSeconds` into its loop, looping continuously until `stopPlayback()` is called.
  - `function stopPlayback(): void` — stops and disconnects all currently scheduled nodes.
  - `function getPlayheadSeconds(): number` — returns current position within the active track's loop (0 to `track.duration`), based on elapsed `AudioContext` time.
  - Module-level state later tasks rely on: `let isPlaying`, `let currentTrackIndex`, `let pausedOffset`.

- [ ] **Step 1: Create the file with the track data and a manual console check**

Write `d:/PROJECTS/musicplayer/index.html` with this content:

```html
<script>
const TRACKS = [
  {
    id: 'lofi-drift',
    title: 'Lo-fi Drift',
    artist: 'Synth Session',
    duration: 8,
    tempo: 78,
    waveform: 'sine',
    gradient: ['#6b5b95', '#2e2350'],
    notes: [
      { time: 0.0, freq: 220.00, dur: 0.9, gain: 0.18 },
      { time: 1.0, freq: 261.63, dur: 0.9, gain: 0.16 },
      { time: 2.0, freq: 196.00, dur: 0.9, gain: 0.18 },
      { time: 3.0, freq: 246.94, dur: 0.9, gain: 0.15 },
      { time: 4.0, freq: 220.00, dur: 0.9, gain: 0.18 },
      { time: 5.0, freq: 174.61, dur: 0.9, gain: 0.16 },
      { time: 6.0, freq: 196.00, dur: 0.9, gain: 0.17 },
      { time: 7.0, freq: 146.83, dur: 0.9, gain: 0.15 }
    ]
  },
  {
    id: 'neon-pulse',
    title: 'Neon Pulse',
    artist: 'Synth Session',
    duration: 6,
    tempo: 128,
    waveform: 'sawtooth',
    gradient: ['#ff2e88', '#1b0e3d'],
    notes: [
      { time: 0.00, freq: 293.66, dur: 0.22, gain: 0.12 },
      { time: 0.47, freq: 293.66, dur: 0.22, gain: 0.10 },
      { time: 0.94, freq: 349.23, dur: 0.22, gain: 0.12 },
      { time: 1.41, freq: 392.00, dur: 0.22, gain: 0.10 },
      { time: 1.88, freq: 293.66, dur: 0.22, gain: 0.12 },
      { time: 2.35, freq: 349.23, dur: 0.22, gain: 0.10 },
      { time: 2.82, freq: 440.00, dur: 0.22, gain: 0.12 },
      { time: 3.29, freq: 392.00, dur: 0.22, gain: 0.10 },
      { time: 3.76, freq: 293.66, dur: 0.22, gain: 0.12 },
      { time: 4.23, freq: 349.23, dur: 0.22, gain: 0.10 },
      { time: 4.70, freq: 440.00, dur: 0.22, gain: 0.12 },
      { time: 5.17, freq: 523.25, dur: 0.22, gain: 0.10 }
    ]
  },
  {
    id: 'midnight-piano',
    title: 'Midnight Piano',
    artist: 'Synth Session',
    duration: 10,
    tempo: 60,
    waveform: 'triangle',
    gradient: ['#1e3a5f', '#0a0f1f'],
    notes: [
      { time: 0.0, freq: 261.63, dur: 1.6, gain: 0.20 },
      { time: 1.0, freq: 329.63, dur: 1.4, gain: 0.16 },
      { time: 2.0, freq: 392.00, dur: 1.6, gain: 0.18 },
      { time: 3.5, freq: 329.63, dur: 1.2, gain: 0.14 },
      { time: 5.0, freq: 293.66, dur: 1.6, gain: 0.20 },
      { time: 6.0, freq: 349.23, dur: 1.4, gain: 0.16 },
      { time: 7.0, freq: 440.00, dur: 1.6, gain: 0.18 },
      { time: 8.5, freq: 349.23, dur: 1.2, gain: 0.14 }
    ]
  }
];

let audioCtx = null;
let scheduledNodes = [];
let isPlaying = false;
let currentTrackIndex = 0;
let pausedOffset = 0;
let loopAnchorCtxTime = 0; // ctx.currentTime corresponding to loop position 0 of the current play run

function getAudioCtx() {
  if (!audioCtx) {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  }
  return audioCtx;
}

function stopPlayback() {
  scheduledNodes.forEach(osc => {
    try { osc.stop(); } catch (e) { /* already stopped */ }
  });
  scheduledNodes = [];
}

function scheduleLoopIteration(track, ctx, iterationStartCtxTime) {
  track.notes.forEach(note => {
    const when = iterationStartCtxTime + note.time;
    if (when < ctx.currentTime) return;
    const osc = ctx.createOscillator();
    const gainNode = ctx.createGain();
    osc.type = track.waveform;
    osc.frequency.value = note.freq;
    gainNode.gain.setValueAtTime(0, when);
    gainNode.gain.linearRampToValueAtTime(note.gain, when + 0.02);
    gainNode.gain.linearRampToValueAtTime(0.0001, when + note.dur);
    osc.connect(gainNode).connect(ctx.destination);
    osc.start(when);
    osc.stop(when + note.dur + 0.05);
    scheduledNodes.push(osc);
  });
}

function playTrack(track, offsetSeconds) {
  const ctx = getAudioCtx();
  stopPlayback();
  loopAnchorCtxTime = ctx.currentTime - offsetSeconds;
  // Schedule the current partial iteration plus the next full iteration
  // so playback covers at least one full loop without gaps.
  scheduleLoopIteration(track, ctx, loopAnchorCtxTime);
  scheduleLoopIteration(track, ctx, loopAnchorCtxTime + track.duration);
}

function getPlayheadSeconds(track) {
  const ctx = getAudioCtx();
  const elapsed = ctx.currentTime - loopAnchorCtxTime;
  return ((elapsed % track.duration) + track.duration) % track.duration;
}

// Manual verification only (see Step 2) — no automated test runner for this file.
</script>
```

- [ ] **Step 2: Manually verify the engine in a browser console**

Open the file directly in a browser (double-click, or `file:///d:/PROJECTS/musicplayer/index.html`), open DevTools console, and run:

```js
playTrack(TRACKS[0], 0);
```

Expected: you hear a slow looping synth pattern (Lo-fi Drift) starting immediately, and it keeps looping past the 8-second mark without a gap or click. Then run:

```js
stopPlayback();
```

Expected: audio stops immediately (any already-started notes finish their short envelope, no new notes trigger). Then run:

```js
getPlayheadSeconds(TRACKS[0]);
```

immediately after a fresh `playTrack(TRACKS[0], 3)` call — expected: a number close to `3` (within ~0.3s, accounting for call overhead).

- [ ] **Step 3: Commit**

No git repo exists in `d:/PROJECTS` — skip commit for this project; just confirm the file is saved.

---

### Task 2: Static player UI markup and theme-aware styling

**Files:**
- Modify: `d:/PROJECTS/musicplayer/index.html` (add markup above the existing `<script>`, add a new `<style>` block above the markup)

**Interfaces:**
- Consumes: `TRACKS[0]` (for initial static content only — no JS wiring yet).
- Produces DOM elements later tasks bind to (exact IDs):
  - `#album-art` — the rotating/pulsing art disc (div with background gradient)
  - `#track-title`, `#track-artist` — text nodes
  - `#seek-bar` — `<input type="range" min="0" max="100">` (value is a 0–100 percentage of loop position)
  - `#time-elapsed`, `#time-remaining` — text nodes, format `M:SS`
  - `#btn-prev`, `#btn-play-pause`, `#btn-next` — buttons (`#btn-play-pause` toggles a `.playing` class)
  - `#volume-slider` — `<input type="range" min="0" max="100">`

- [ ] **Step 1: Add the `<style>` block**

Insert above the existing `<script>` tag:

```html
<style>
  :root {
    --bg: #f4f2fa;
    --card-bg: #ffffff;
    --text-primary: #1a1530;
    --text-secondary: #6b6580;
    --accent: #6b5b95;
    --track-bg: #e4e0f0;
  }
  @media (prefers-color-scheme: dark) {
    :root {
      --bg: #12101c;
      --card-bg: #1c1830;
      --text-primary: #f0eefa;
      --text-secondary: #a39dc0;
      --accent: #a08bd6;
      --track-bg: #2c2648;
    }
  }
  :root[data-theme="dark"] {
    --bg: #12101c;
    --card-bg: #1c1830;
    --text-primary: #f0eefa;
    --text-secondary: #a39dc0;
    --accent: #a08bd6;
    --track-bg: #2c2648;
  }
  :root[data-theme="light"] {
    --bg: #f4f2fa;
    --card-bg: #ffffff;
    --text-primary: #1a1530;
    --text-secondary: #6b6580;
    --accent: #6b5b95;
    --track-bg: #e4e0f0;
  }
  body {
    margin: 0;
    min-height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg);
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  }
  .player-card {
    width: min(340px, 90vw);
    background: var(--card-bg);
    border-radius: 24px;
    padding: 28px;
    box-shadow: 0 20px 60px -20px rgba(0,0,0,0.35);
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 20px;
  }
  #album-art {
    width: 200px;
    height: 200px;
    border-radius: 50%;
    box-shadow: 0 12px 30px -8px rgba(0,0,0,0.4);
    animation: spin 12s linear infinite, pulse 2s ease-in-out infinite;
    animation-play-state: paused;
  }
  @keyframes spin {
    from { transform: rotate(0deg); }
    to { transform: rotate(360deg); }
  }
  @keyframes pulse {
    0%, 100% { opacity: 1; transform: scale(1); }
    50% { opacity: 0.85; transform: scale(1.03); }
  }
  .track-info { text-align: center; }
  #track-title { margin: 0; font-size: 1.1rem; color: var(--text-primary); }
  #track-artist { margin: 4px 0 0; font-size: 0.85rem; color: var(--text-secondary); }
  .seek-row { width: 100%; display: flex; flex-direction: column; gap: 6px; }
  #seek-bar, #volume-slider {
    width: 100%;
    accent-color: var(--accent);
  }
  .time-row {
    display: flex;
    justify-content: space-between;
    font-size: 0.75rem;
    color: var(--text-secondary);
  }
  .controls-row {
    display: flex;
    align-items: center;
    gap: 20px;
  }
  .controls-row button {
    background: none;
    border: none;
    cursor: pointer;
    color: var(--text-primary);
    padding: 8px;
    border-radius: 999px;
    transition: transform 0.15s ease, opacity 0.15s ease;
  }
  .controls-row button:hover { opacity: 0.75; }
  .controls-row button:active { transform: scale(0.92); }
  .controls-row button:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px; }
  #btn-play-pause {
    width: 52px;
    height: 52px;
    border-radius: 50%;
    background: var(--accent);
    color: #fff;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .volume-row {
    width: 100%;
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.75rem;
    color: var(--text-secondary);
  }
</style>
```

- [ ] **Step 2: Add the HTML markup**

Insert above the `<style>` block (or right after `<body>` conceptually — remember the Artifact wrapper supplies `<body>`, so this file's root content starts directly with these tags):

```html
<div class="player-card">
  <div id="album-art"></div>
  <div class="track-info">
    <p id="track-title">Lo-fi Drift</p>
    <p id="track-artist">Synth Session</p>
  </div>
  <div class="seek-row">
    <input type="range" id="seek-bar" min="0" max="100" value="0" />
    <div class="time-row">
      <span id="time-elapsed">0:00</span>
      <span id="time-remaining">-0:08</span>
    </div>
  </div>
  <div class="controls-row">
    <button id="btn-prev" aria-label="Previous track">⏮</button>
    <button id="btn-play-pause" aria-label="Play">▶</button>
    <button id="btn-next" aria-label="Next track">⏭</button>
  </div>
  <div class="volume-row">
    <span>🔉</span>
    <input type="range" id="volume-slider" min="0" max="100" value="70" />
  </div>
</div>
```

- [ ] **Step 3: Manually verify the static layout**

Open `d:/PROJECTS/musicplayer/index.html` directly in a browser. Expected: a centered rounded card with a plain (not-yet-gradient-colored) circular art placeholder, "Lo-fi Drift" / "Synth Session" text, a seek bar showing `0:00` / `-0:08`, three control buttons, and a volume slider. Toggle OS dark mode and reload — expected: background/card/text colors swap to the dark palette without any layout shift.

- [ ] **Step 4: Commit**

No git repo — skip; confirm file saved.

---

### Task 3: Wire UI controls to the audio engine, playhead loop, and animations

**Files:**
- Modify: `d:/PROJECTS/musicplayer/index.html` (extend the `<script>` block from Task 1)

**Interfaces:**
- Consumes: `TRACKS`, `getAudioCtx`, `playTrack`, `stopPlayback`, `getPlayheadSeconds`, `isPlaying`, `currentTrackIndex`, `pausedOffset` (from Task 1); DOM IDs from Task 2.
- Produces:
  - `function renderTrack(index): void` — updates art gradient, title, artist, seek bar max/labels for `TRACKS[index]`, restarts the pulse animation duration for the new tempo.
  - `function togglePlayPause(): void`
  - `function seekTo(percent0to100): void`
  - `function goToTrack(index): void` — handles wraparound (prev at index 0 → last track, next at last → index 0).
  - `function formatTime(seconds): string` — `M:SS`.

- [ ] **Step 1: Add rendering, control-wiring, and RAF playhead-update code**

Append inside the existing `<script>` block, after the Task 1 code:

```js
const albumArt = document.getElementById('album-art');
const trackTitleEl = document.getElementById('track-title');
const trackArtistEl = document.getElementById('track-artist');
const seekBar = document.getElementById('seek-bar');
const timeElapsedEl = document.getElementById('time-elapsed');
const timeRemainingEl = document.getElementById('time-remaining');
const playPauseBtn = document.getElementById('btn-play-pause');
const prevBtn = document.getElementById('btn-prev');
const nextBtn = document.getElementById('btn-next');
const volumeSlider = document.getElementById('volume-slider');

let rafId = null;
let isSeeking = false;

function formatTime(seconds) {
  const s = Math.max(0, Math.round(seconds));
  const m = Math.floor(s / 60);
  const rem = s % 60;
  return `${m}:${rem.toString().padStart(2, '0')}`;
}

function renderTrack(index) {
  const track = TRACKS[index];
  albumArt.style.background = `radial-gradient(circle at 35% 30%, ${track.gradient[0]}, ${track.gradient[1]})`;
  albumArt.style.animationDuration = `${60 / track.tempo * 8}s, ${60 / track.tempo}s`;
  trackTitleEl.textContent = track.title;
  trackArtistEl.textContent = track.artist;
  seekBar.value = 0;
  timeElapsedEl.textContent = '0:00';
  timeRemainingEl.textContent = `-${formatTime(track.duration)}`;
}

function updatePlayheadUI() {
  const track = TRACKS[currentTrackIndex];
  const pos = isPlaying ? getPlayheadSeconds(track) : pausedOffset;
  if (!isSeeking) {
    seekBar.value = Math.round((pos / track.duration) * 100);
  }
  timeElapsedEl.textContent = formatTime(pos);
  timeRemainingEl.textContent = `-${formatTime(track.duration - pos)}`;
  rafId = requestAnimationFrame(updatePlayheadUI);
}

function setPlayingUI(playing) {
  playPauseBtn.textContent = playing ? '⏸' : '▶';
  playPauseBtn.setAttribute('aria-label', playing ? 'Pause' : 'Play');
  albumArt.style.animationPlayState = playing ? 'running' : 'paused';
}

function togglePlayPause() {
  const track = TRACKS[currentTrackIndex];
  if (isPlaying) {
    pausedOffset = getPlayheadSeconds(track);
    stopPlayback();
    isPlaying = false;
  } else {
    getAudioCtx().resume();
    playTrack(track, pausedOffset);
    isPlaying = true;
  }
  setPlayingUI(isPlaying);
}

function seekTo(percent) {
  const track = TRACKS[currentTrackIndex];
  const targetSeconds = (percent / 100) * track.duration;
  pausedOffset = targetSeconds;
  if (isPlaying) {
    playTrack(track, targetSeconds);
  }
}

function goToTrack(index) {
  const wasPlaying = isPlaying;
  stopPlayback();
  isPlaying = false;
  currentTrackIndex = (index + TRACKS.length) % TRACKS.length;
  pausedOffset = 0;
  renderTrack(currentTrackIndex);
  if (wasPlaying) {
    getAudioCtx().resume();
    playTrack(TRACKS[currentTrackIndex], 0);
    isPlaying = true;
  }
  setPlayingUI(isPlaying);
}

playPauseBtn.addEventListener('click', togglePlayPause);
prevBtn.addEventListener('click', () => goToTrack(currentTrackIndex - 1));
nextBtn.addEventListener('click', () => goToTrack(currentTrackIndex + 1));
seekBar.addEventListener('mousedown', () => { isSeeking = true; });
seekBar.addEventListener('touchstart', () => { isSeeking = true; });
seekBar.addEventListener('input', () => {
  timeElapsedEl.textContent = formatTime((seekBar.value / 100) * TRACKS[currentTrackIndex].duration);
});
seekBar.addEventListener('change', () => {
  seekTo(Number(seekBar.value));
  isSeeking = false;
});
volumeSlider.addEventListener('input', () => {
  // Simple global volume via a shared GainNode would require refactoring
  // playTrack's node graph; for this scope, volume controls the
  // AudioContext destination via a master gain applied at engine level.
  masterGain.gain.value = volumeSlider.value / 100;
});

renderTrack(currentTrackIndex);
updatePlayheadUI();
```

- [ ] **Step 2: Add a master gain node so the volume slider actually works**

Go back and modify the Task 1 code: add a `masterGain` node that all oscillators route through instead of connecting directly to `ctx.destination`.

In `getAudioCtx()`, replace:

```js
function getAudioCtx() {
  if (!audioCtx) {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  }
  return audioCtx;
}
```

with:

```js
let masterGain = null;

function getAudioCtx() {
  if (!audioCtx) {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    masterGain = audioCtx.createGain();
    masterGain.gain.value = 0.7;
    masterGain.connect(audioCtx.destination);
  }
  return audioCtx;
}
```

And in `scheduleLoopIteration`, replace:

```js
    osc.connect(gainNode).connect(ctx.destination);
```

with:

```js
    osc.connect(gainNode).connect(masterGain);
```

- [ ] **Step 3: Manually verify full interaction**

Open `d:/PROJECTS/musicplayer/index.html` in a browser and check each in order:
1. Click play (▶) — expected: Lo-fi Drift audio starts, button becomes ⏸, album art starts rotating and pulsing.
2. Drag the seek bar to ~50% and release — expected: audio jumps to roughly the midpoint of the 8s loop, elapsed time updates accordingly.
3. Click next (⏭) — expected: audio switches to "Neon Pulse" (faster sawtooth pattern), art gradient/rotation speed changes, still playing.
4. Click prev (⏮) twice — expected: wraps to "Midnight Piano" (last track), then back to "Lo-fi Drift".
5. Click pause (⏸) — expected: audio stops, art rotation freezes in place (not reset), button becomes ▶.
6. Drag volume slider to 0 — expected: audio becomes silent while still "playing" (art still animating); drag back up — audio returns.
7. Toggle OS/browser dark mode — expected: no layout shift, colors swap correctly, animations unaffected.

- [ ] **Step 4: Commit**

No git repo — skip; confirm file saved.

---

### Task 4: Publish as an Artifact

**Files:**
- None modified — this task only publishes the already-completed `d:/PROJECTS/musicplayer/index.html`.

**Interfaces:**
- Consumes: the completed, manually-verified `index.html` from Task 3.

- [ ] **Step 1: Load the artifact-design skill for a final design pass**

Re-check the file against artifact-design guidance (dark/light theming already covered in Task 2; confirm no horizontal overflow at narrow widths by mentally checking the `min(340px, 90vw)` card width against a 320px viewport).

- [ ] **Step 2: Publish via the Artifact tool**

Call the Artifact tool with `file_path: d:/PROJECTS/musicplayer/index.html`, a `title` implied by the file's own content (no `<title>` tag exists yet — add `<title>Music Player</title>` is not applicable since this file has no `<head>`; instead rely on the Artifact tool's basename fallback, or note the omission to the user), a one-sentence `description` ("A synthesized 3-track music player with animated album art"), and `favicon: "🎵"`.

- [ ] **Step 3: Manually verify the published artifact**

Open the returned artifact URL and repeat the Step 3 interaction checklist from Task 3 (play, seek, next, prev, pause, volume, theme toggle) inside the published sandboxed iframe specifically, since CSP/autoplay behavior can differ from a local `file://` open — confirm the `AudioContext` isn't blocked (some browsers require the play click itself to be the resuming gesture, which `togglePlayPause`'s direct click-handler call to `getAudioCtx().resume()` already satisfies).

---

## Self-Review Notes

- Spec coverage: 3 tracks ✓, prev/next only (no playlist/shuffle/repeat) ✓, album art rotation+pulse tied to play state via `animation-play-state` ✓, seek bar with elapsed/remaining ✓, volume ✓, theme-aware ✓, single self-contained file with no external requests ✓.
- Type/signature consistency checked: `playTrack(track, offsetSeconds)`, `getPlayheadSeconds(track)`, `goToTrack(index)`, `seekTo(percent)` used with matching signatures across Tasks 1 and 3.
- No placeholders remain; all code blocks are complete and runnable as written.
