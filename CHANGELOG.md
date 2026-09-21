# Changelog / development diary

This is the work log for **UltraSinger Studio**: a packaging layer and a set of
patches on top of [UltraSinger](https://github.com/rakuri255/UltraSinger)
(v0.0.13.dev16). The original author has approved this direction.

Two layers, on purpose:

| Layer | Where | What it is |
|---|---|---|
| **A. Upstream patches** | `UltraSinger-src/` | Changes that belong in rakuri255/UltraSinger. Cherry-pick these. |
| **B. Studio wrapper** | `web/`, `*.bat`, `requirements*.txt` | Windows one-click install, browser UI, model download, GPU/CPU fallbacks. Optional for Linux users who already run UltraSinger. |

Russian README: [README.md](README.md). English: [README.en.md](README.en.md).
Ready-to-paste GitHub PR body: [docs/PULL_REQUEST.md](docs/PULL_REQUEST.md).

---

## How to try this

On Windows 10/11 x64, ASCII path only (e.g. `H:\Karaoke\ultrasinger-studio`):

```
install.bat
start.bat
```

Browser opens `http://127.0.0.1:8000`. Drop an MP3, pick a preset, press
**Create chart**. UI language defaults to **EN**; switch **EN / RU** in the
header. Song language defaults to Auto.

CLI:

```
make.bat "H:\Karaoke\mp3\Artist - Title.mp3"
```

---

## Layer A — patches for UltraSinger itself

### A1. Windows / ffmpeg robustness

* `modules/ffmpeg_helper.py`
  * Decode ffmpeg/ffprobe stdout as **UTF-8 with `errors=replace`**. Fixes
    `UnicodeDecodeError: 'charmap' codec can't decode byte 0x98` on Russian
    Windows (cp1251).
  * **Embedded album art is not a video stream.** Streams with
    `attached_pic` are ignored; audio-only extensions (mp3/flac/wav/m4a/ogg/…)
    skip the video check. Stops
    `FFmpeg cannot edit existing files in-place` on VK/Yandex Music files.
  * Extracted audio is never written over the source file.

### A2. CLI flag bugs

* `UltraSinger.py` — `parse_bool_option()`
  * `--create_audio_chunks` without a value used to write `""` → chunks
    **never** turned on.
  * `--quantize_to_key` without a value used to write `""` → quantization
    **turned off**.
  * Added `--no_quantize_to_key`.

### A3. Artist / title from the filename

* `UltraSinger.py` + `modules/Speech_Recognition/lyrics.py`
  * `#ARTIST` / `#TITLE` and output folder names come from
    `Artist - Title.mp3` (leading track numbers stripped).
  * MusicBrainz still fills year/cover when it knows the song; a miss no
    longer writes `#ARTIST:Unknown Artist`.
  * New flags: `--artist`, `--title`.

### A4. Forced lyrics + alignment (`--lyrics_file`)

* `modules/Speech_Recognition/Whisper.py`, `lyrics.py`, `UltraSinger.py`
  * User-supplied `.txt` / `.lrc` is used as **words**.
  * WhisperX **aligns** those words to the audio (unless the file already
    has per-word LRC timings — then Whisper is skipped).
  * Cache key includes a lyrics hash so a changed text is not reused.

### A5. Demucs without `diffq`

* `modules/Audio/separation.py` — `resolve_demucs_model()`
  * `mdx_q` / `mdx_extra_q` need `diffq`, which has no Windows wheel for
    Python 3.11+. They are rewritten to `mdx` / `mdx_extra` instead of
    dying with `FATAL: Trying to use DiffQ`.

### A6. Optional sheet music

* `modules/sheet.py` — `music21` is optional. The core pipeline runs
  without it.

### A7. Whisper compute type on old GPUs

* `modules/Speech_Recognition/Whisper.py`
  * Ask ctranslate2 `get_supported_compute_types`. Pascal (GTX 10xx,
    CC 6.1) has **no efficient fp16** (needs CC ≥ 7.0). Auto-picks
    `int8_float32`. Never request `float16` on this hardware.

### A8. Karaoke / stems optional

* `UltraSinger.py` (`_export_stem_if_present`), `modules/Audio/convert_audio.py`
  * Unchecking karaoke / `--disable_karaoke` / `--disable_separation` must
    not abort the `.txt` because FFmpeg cannot find missing stems.

### A9. Human-style UltraStar charts

Target: a chart a human would tap (The Doors — *People Are Strange*), not a
machine-gun of 16th notes and not one endless line.

* `UltraSinger.py` — `split_syllables_into_segments`
  * Short syllables stay **one note** (слог ≈ нота).
  * Only holds longer than a quarter are sliced into eighths as `~`, then
    merged back if the pitch did not change.
  * Hyphenation language follows the **song**, not a hardcoded Russian
    setting (`#LANGUAGE:Russian` on an English song must not drive hyphens).
* `modules/Ultrastar/ultrastar_writer.py`
  * Caps: `MAX_NOTE_BEATS=24`, `MAX_LINE_BEATS=56`, `MAX_LINE_WORDS=8`,
    `MIN_GAP_BEATS=1`, `PAUSE_BREAK_BEATS=4`.
  * Phrase breaks on silence / punctuation / word count — not one infinite
    line.
  * Removed the leftover `silence_split_duration` (NameError after the
    rewrite).

### A10. Monotonic note times (lyric lag bug)

Symptom on *ЛСП — Монетка*: from ~line 541 the on-screen lyric lagged hard.
32 reverse starts in the `.txt` (199→182, 743→723 «Ба»), 21 leftover `~`.
UltraStar Deluxe plays **file order**; a later line with an earlier beat
shows empty bars and the real word appears late.

Cause: overlapping Whisper / backing-vocal words; 8th-note `~` of the
previous word ran past the next; the writer did not sort or skip.

Fix:

* `clip_transcribed_overlaps()` after `TranscribeAudio` —
  `previous.end ≤ next.start`, drop overlapping `~`.
* `order_and_clip_midi_segments()` after merge.
* Writer safety net: sort by start beat, drop reverse `~`, clamp real words.

---

## Layer B — UltraSinger Studio (wrapper)

Not required to merge A into upstream. This is how people actually run it
on Windows without Docker.

* `install.bat` / `start.bat` / `stop.bat` / `make.bat` / `use_gpu.bat` /
  `use_hf_mirror.bat` / `download_models.bat` / `fix_torchcodec.bat` /
  `test_clip.bat`
  * Pure ASCII + CRLF. No `chcp`, no Cyrillic inside `.bat` (two crash
    modes on Russian Windows).
  * Isolated venv + Python 3.12 via `uv` **inside the folder**. System
    Python (e.g. 3.14) is not touched.
  * Refuses to install on a path with non-ASCII characters
    (`C:\Users\Денис` breaks uv/Python).
  * GPU: torch **cu126** so Pascal sm_61 (GTX 1070) still works. cu128/cu129
    dropped those kernels.
* `web/app.py` — FastAPI: upload / YouTube, presets, SSE log, ZIP download.
* `web/static/index.html` — single-file UI, **EN default**, header **EN / RU**
  toggle (`localStorage us_ui_lang`).
* `web/lyrics_finder.py` — lyrics order: **web first** (LRCLIB, lyrics.ovh,
  amdm.ru), then file tags / sidecar `.lrc`, then manual paste. Text is
  always time-aligned, never dumped as untimed plain text.
* `web/prefetch_models.py` — retries, resume, HuggingFace → hf-mirror →
  ModelScope. Offline once cached (`HF_HUB_OFFLINE=1`).
* `web/pipeline_launcher.py` — `SetErrorMode` so a missing CUDA DLL becomes
  a log line, not a modal Windows dialog.
* Auto-probes: Whisper GPU vs CPU, ctranslate2 compute types, torchcodec
  wheel vs torch version (quiet when versions already match).
* Quantized demucs models hidden from the UI.

User veto: **no Docker as the primary path**. `docker/` stays a small
optional tail in the README.

---

## Round-by-round diary (how we got here)

Field-tested on one machine: Windows 10 22H2, i5-5200, GTX 1070 8 GB
(Pascal CC 6.1), profile `C:\Users\Денис`, install dir
`H:\Karaoke\ultrasinger-studio`.

| Round | What the user hit | What we changed |
|---|---|---|
| 1–6 | 625 deps, Cyrillic profile path, cp1251 ffmpeg crash, no GUI, GTX 1070 dead on torch cu128 | Isolated venv, UTF-8 ffmpeg, web UI, torch cu126 |
| 7 | Whisper invents lyrics | Online lyrics first, then tags, then manual; neural **alignment** only |
| 8–9 | `#ARTIST:Unknown Artist` on Russian songs | Filename before the hyphen |
| 10 | Picked Whisper `large-v1` thinking it was lighter | Alias to large-v3; real light models are medium / Balance (small) |
| 11 | Huge pitch jumps, notes too short to sing | Syllable ≈ note, no 16th machine-gun |
| 12 | Over-correction: one endless line, no syllable splits; English song tagged Russian | Human Doors-style phrases; hyphenation follows song language |
| 13 | Karaoke checkbox off → FFmpeg “No such file”; torchcodec warning every run | Skip missing stems; stay quiet when torchcodec versions match |
| 14 | `NameError: silence_split_duration` while writing the txt | Writer rewrite, no leftover name |
| 15 | *Монетка*: lyrics lag from line ~541, reverse beats, 21× `~` then «Ба» | Clip overlaps + sort/skip in the writer |
| 16 | Need EN UI + GitHub PR diary | EN default, EN/RU toggle, this file |

---

## Files touched in UltraSinger-src (for reviewers)

```
src/UltraSinger.py
src/modules/ffmpeg_helper.py
src/modules/Audio/convert_audio.py
src/modules/Audio/separation.py
src/modules/Speech_Recognition/Whisper.py
src/modules/Speech_Recognition/lyrics.py
src/modules/Speech_Recognition/hyphenation.py
src/modules/Ultrastar/ultrastar_writer.py
src/modules/sheet.py
src/modules/Midi/midi_creator.py   # pitch-band filter vs SwiftF0 C7 crumbs
pyproject.toml                     # drop unused heavy deps
```

Studio-only (not for the UltraSinger core PR unless wanted):

```
web/app.py  web/static/index.html  web/lyrics_finder.py
web/prefetch_models.py  web/pipeline_launcher.py
install.bat  start.bat  make.bat  use_gpu.bat  …
README.md  README.en.md  CHANGELOG.md
```
