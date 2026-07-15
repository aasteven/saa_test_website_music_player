# Music Player App — Design Spec

Date: 2026-07-14

## Summary
A self-contained music player built as a single HTML Artifact. No external
network requests are possible (Artifact CSP blocks them), so audio is
synthesized at runtime via the Web Audio API rather than loaded from files
or a CDN.

## Scope
- 3 tracks, each a distinct synthesized melody/chord-loop preset (own
  waveform, tempo, key, name, artist label, duration).
- No playlist panel, no shuffle, no repeat — navigation is prev/next only.

## Layout
Single centered card:
- Album art: CSS-generated abstract gradient per track (no images).
  - Idle: subtle pulse.
  - Playing: continuous slow rotation (vinyl/CD feel) + soft glow pulse
    loosely timed to a beat interval.
- Track title + artist label below art.
- Seek bar with elapsed / remaining time.
- Control row: prev / play-pause / next.
- Volume slider.

## Playback Engine
- One `AudioContext` per session.
- Each track is a declarative note/beat schedule (array of {time, freq,
  duration, waveform}) looped for the track's stated duration.
- Play: create/resume the audio graph and start scheduling from the
  current playhead offset.
- Pause: suspend the context; retain playhead position.
- Seek: recompute playhead offset, restart scheduling from new offset
  (stop + reschedule remaining notes).
- Next/Prev: tear down current track's nodes, reset playhead, build and
  (if was playing) immediately start the new track's graph.

## Animation Rules
- Only `transform` and `opacity` are animated (per project guardrails).
- Rotation animation is paused via `animation-play-state` when paused,
  not removed/re-added, so it resumes smoothly.
- Beat-pulse glow uses a CSS animation keyed to each track's tempo,
  restarted on track change.

## Out of Scope
- Real/licensed audio file playback.
- Playlist browsing UI, shuffle, repeat.
- Persistence (no localStorage of playback state) — resets on reload.

## Open Questions
None — design approved by user on 2026-07-14.
