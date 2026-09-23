# ZCode agent session learning — 2026-09-23: Local voice dictation for an Electron app (ZCode Desktop)

## What was built
Voice input for ZCode Desktop's chat text input, fully local: hold Alt+V (or click a floating mic pill) → record via `getUserMedia` + `MediaRecorder` (webm/opus) → POST to a local mlx-whisper HTTP service (127.0.0.1:8399) → insert transcript at the caret via `document.execCommand('insertText')` (React-safe).

## Architecture that worked
1. **STT as a separate local service** (not in-app): `~/.venvs/voice-stt` + stdlib `http.server` + `mlx-whisper` (whisper-turbo on Apple MLX), LaunchAgent auto-start, ~1–2 s per utterance, zero cloud/keys. Decouples STT iteration from app patching; testable with curl alone.
2. **Bootstrap patch (Approach B)** instead of touching app bundles: rewrite Electron `package.json` `"main"` → a tiny ESM `voice-bootstrap.js` that registers `web-contents-created → did-finish-load → executeJavaScript(inject)` and then `await import('./out/main/index.js')`. Smallest revert surface; bundles untouched.
3. **Injection safety**: only inject into `file:`/`app:` URLs (never remote content); log injection failures.
4. **Caret insertion for React inputs**: `execCommand('insertText')` on the focused element fires the native input pipeline so React state updates; fallback `setRangeText` for textareas; clipboard fallback when no editable is focused.
5. **Closed-loop verification without a human speaker**: synthesize ground-truth audio with Kokoro TTS (EN) / Edge-TTS ru-RU-DmitryNeural (RU), POST to the service, assert `difflib` similarity ≥ 0.8 + determinism. Both languages scored 1.00.

## Pitfalls hit (and cures)
- **`MediaRecorder` produces webm/opus, not wav** — server must ffmpeg-normalize arbitrary input to 16 kHz mono wav before whisper.
- **Whisper hallucinates on silence** ("Thank you.") — add an RMS silence gate (threshold ~2e-4) before calling the model.
- **LaunchAgent env has no PATH** — mlx-whisper shells out to `ffmpeg`; set `PATH` explicitly in the plist or the service 400s everything.
- **venv surprise**: `mlx-whisper` pulls numpy but NOT `soundfile` — install explicitly or the silence gate kills every request.
- **asar repack gotchas**: (a) `@electron/asar` takes only the LAST `--unpack` flag — combine globs with braces `{a,b}`; (b) glob needs a `**/` prefix to match nested paths; (c) asar v4 dedupes identical content (overlapping offsets are fine); verify with file-list parity + random sha256 spot-checks against the extracted tree; (d) replicate the original unpacked set exactly (here: 3 native binaries) or native modules break.
- **HF anonymous model download can stall/restart** — pre-download with `hf download <model>` before first service start.
- **Kokoro TTS is EN-only** (no RU/HE voices) — Edge-TTS covers other languages for fixtures.
- **keyup-during-permission-prompt race**: `getUserMedia` is async; if the user releases the hotkey while the mic prompt is open, mark `stopRequested` and cancel after grant, or recording runs forever.

## Verification mindset
No-vision environment ⇒ verify numerically everywhere: HTTP status/evidence assertions, sha256 round-trips, difflib similarity, ffprobe durations. Never trust a worker's "done" — rerun the checks (worker 1 silently stalled; direct execution finished the task).

## Reusable snippet refs
- Server: `~/.zcode/voice/stt_server.py` (health + transcribe endpoints, 503-serialized inference, 50 MB cap)
- Inject: `~/.zcode/voice/voice-inject.js` (pill UI, hotkey, race-safe recorder, caret insertion)
- Test rig: `.smart/13cf2560/run_voice_tests.py` (12 adversarial + property tests, all green)
