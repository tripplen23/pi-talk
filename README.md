# dictate (ElevenLabs)

Minimal voice dictation for pi, backed by **ElevenLabs Realtime STT** (Scribe
v2 Realtime). No floating bubbles, no menu bar app, no notifications.

- **Toggle:** `alt+m` (press to start, press again to stop) — works **anywhere in pi**, not just the main chat input: quiz popups, `ask_user_question`, `ctx.ui.editor()`/`input()` dialogs, selectors. The key is intercepted at the TUI input layer, before whatever component has focus.
- **Cancel:** `alt+n` (discard the in-flight transcript; safe to press anytime).
- **Where text goes:** to whatever input field is focused **when you stop** (never replaces, always appends). If nothing text-capable is focused, the transcript is copied to the clipboard via `pbcopy` (macOS) so it is never lost.
- **Live feedback:** while recording the status row shows `● ▂▅▇ listening…` (a real-time mic level meter); on stop it flips to `finalizing…`.
- **Backend:** ElevenLabs Realtime Speech-to-Text, `scribe_v2_realtime`.
- **Latency:** audio is transcribed while you talk; committed text is inserted in one shot on stop.

## Install

```bash
pi install git:github.com/<you>/pi-dictate-elevenlabs
```

Or copy `index.ts` to `~/.pi/agent/extensions/dictate/index.ts`.

## One-time setup

```bash
brew install sox                                  # provides `rec` for audio capture
export ELEVENLABS_API_KEY=sk_xxxxxxxxxxxxxxxx     # add to ~/.zshrc or ~/.bashrc
```

The key is used only to mint a short-lived single-use token
(`POST /v1/single-use-token/realtime_scribe`); the WebSocket then connects with
`?token=…`. Sign up at https://elevenlabs.io. Scribe v2 Realtime is
~$0.39/hour (~$0.0065/min).

## Usage

1. Focus any pi input field.
2. `alt+m` to start. The status row shows a red `●` and a live level meter.
3. Talk.
4. `alt+m` again. A spinner shows `finalizing…`, then the text appears in the focused input.

Run `/reload` in pi after installation or edits.

## How it works

- Spawns `rec` (sox) capturing 16kHz mono 16-bit PCM to stdout.
- Mints a single-use realtime token with `ELEVENLABS_API_KEY`.
- Opens `wss://api.elevenlabs.io/v1/speech-to-text/realtime` with
  `model_id=scribe_v2_realtime&audio_format=pcm_16000&commit_strategy=vad&token=…`.
- Streams audio as `{"message_type":"input_audio_chunk","audio_base_64":…,"commit":false,"sample_rate":16000}`.
- Collects `committed_transcript` events; ignores `partial_transcript`.
- On stop, sends a final commit and waits for the last `committed_transcript`.
- Focus-aware delivery: captures pi's `TUI` via an invisible widget and installs
  `tui.addInputListener`, so `alt+m` works inside dialogs. On stop it inspects
  `tui.focusedComponent` — editor-like components get a direct `setText` append,
  opaque components get synthetic keystrokes.

## Customizing

Knobs at the top of `index.ts`:

- **Hotkey:** the `Key.alt("m")` / `Key.alt("n")` references.
- **Endpointing:** `vad_silence_threshold_secs` in `EL_STT_URL` (lower = faster commits, more fragmentation).
- **Commit strategy:** swap `commit_strategy=vad` for `manual` (you then commit explicitly).
- **Language:** add `&language_code=en` (or another ISO-639-1 code) to skip auto-detection.
- **Filler removal:** add `&no_verbatim=true` to drop "um"/false starts.
- **Level meter:** `METER_CELLS`, `METER_TICK_MS`, `METER_FLOOR_DB`, `METER_CEILING_DB`.

## Troubleshooting

- **"ELEVENLABS_API_KEY not set"** — export it in the shell that launches pi.
- **"Failed to spawn 'rec'"** — `brew install sox`, verify `which rec`.
- **No mic input** — check the level meter; if flat, grant your terminal microphone access (macOS: Privacy & Security → Microphone).
- **"ElevenLabs token error"** — bad/expired key, or no network to api.elevenlabs.io.
- **`alt+m` inserts `µ`** — your terminal isn't treating Option as Alt.
- **Debug:** run pi with `DICTATE_DEBUG=1` → appends events to `/tmp/dictate-debug.log`.

## Notes

Fork of [amosblomqvist/pi-dictate](https://github.com/amosblomqvist/pi-dictate)
(MIT), with the speech-to-text backend swapped from Deepgram Nova-3 to
ElevenLabs Scribe v2 Realtime. MIT-licensed — see `LICENSE`.
