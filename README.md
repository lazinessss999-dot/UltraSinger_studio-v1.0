# 🎤 UltraSinger Studio

One-folder Windows app that turns an MP3 (or a YouTube URL) into
UltraStar.txt + MIDI + optional vocals/instrumental.

Built on [UltraSinger](https://github.com/rakuri255/UltraSinger)
(thanks, rakuri255!). Studio keeps the original pipeline and adds the
pieces that made it hard to run on a real Windows box:

| Before | After |
|---|---|
| 625 packages on the system Python, version wars | `install.bat` pins a venv **inside this folder** |
| Cyrillic profile path (`C:\Users\Алиса`) breaks uv/Python | Everything lives on an ASCII path; the installer checks |
| `UnicodeDecodeError` from ffmpeg on cp1251 | ffmpeg output decoded as UTF-8 |
| Console-only | Browser UI: drop a track → preset → log → ZIP |
| GTX 1070 (Pascal) dead on torch cu128 | torch **cu126**, Pascal sm_61 still there |

Russian README: [README.md](README.md). Change log / PR diary:
[CHANGELOG.md](CHANGELOG.md). GitHub PR draft:
[docs/PULL_REQUEST.md](docs/PULL_REQUEST.md).

## Quick start (Windows, no Docker)

Windows 10/11 x64 and internet. No Hyper-V.

1. Copy `ultrasinger-studio` to an **ASCII-only** path
   (`H:\Karaoke\ultrasinger-studio` or `C:\USStudio`). A Cyrillic path
   such as `C:\Users\Алиса\Desktop` will fail — `install.bat` will say so.
2. Double-click **`install.bat`**. It downloads `uv` + Python 3.12 into
   the folder (your system Python is not touched), installs torch +
   whisperx + demucs + swift-f0, and ffmpeg. If you have an NVIDIA GPU
   (including a GTX 1070) answer **y** — that installs torch **cu126**.
3. Double-click **`start.bat`**. Browser: **http://127.0.0.1:8000**
4. Drop an MP3 (or a YouTube URL) → pick a preset → **Create chart**.
   The UI is **English by default**. Switch **EN / RU** in the header
   (remembered in the browser).
5. Download the ZIP into UltraStar Deluxe.

Stop: close the `start.bat` window or run `stop.bat`.

CLI: `make.bat "H:\Karaoke\mp3\Artist - Title.mp3"` (output in
`data\cli_output`).

## Using the UI

1. **Input** — file (mp3/wav/m4a/flac/mp4/…) or YouTube. Name it
   `Artist - Title` (a leading `01 - ` is stripped). That becomes
   `#ARTIST` / `#TITLE`. MusicBrainz only fills year/cover.
2. **Preset**
   * ⚡ **Fast** — whisper base, no vocal split. Sanity check.
   * 🎯 **Balance** — whisper small + demucs. Start here.
   * 💎 **Quality** — best accuracy (slow on CPU).
3. **Lyrics** (optional, but much better): web first (LRCLIB, lyrics.ovh,
   amdm.ru), then file tags / a nearby `.lrc`. Words come from that text;
   a neural net only **aligns** them in time. Paste wins over search.
   LRC timings skip Whisper entirely.
4. **Advanced** — song language (default **Auto**), Whisper model,
   demucs, UltraStar format, quantize-to-key, karaoke stems.

## GPU notes (GTX 10xx / Pascal)

* PyTorch 2.8 **cu128/cu129 has no sm_61 kernels**. `install.bat` + `y`
  installs **cu126**.
* Need an NVIDIA driver that speaks CUDA 12.6 (≥ 560.x). Older driver →
  the header chip says **CPU mode** and it still runs, just slower.
* ctranslate2 will **not** use fp16 on CC 6.1 (needs ≥ 7.0). Auto-probe
  picks `int8_float32`. Do not force `float16` on this card.

## Layout

```
ultrasinger-studio/
├── install.bat / start.bat / stop.bat / make.bat
├── use_gpu.bat / use_hf_mirror.bat / download_models.bat
├── fix_torchcodec.bat / test_clip.bat
├── web/                 # FastAPI + UI
├── UltraSinger-src/     # UltraSinger v0.0.13.dev16 + patches
├── requirements*.txt
├── data/                # models + jobs
└── docker/              # optional, not the primary path
```

Docker is a short optional section only. The intended path is the `.bat`
files.

## Common problems

* **install.bat flashes and dies** — the `.bat` was re-saved with the
  wrong encoding. Copy a fresh ASCII/CRLF file. Run from cmd to see the
  error: `cd /d H:\Karaoke\ultrasinger-studio` then `install.bat`
* **Cyrillic in the path** — move the folder.
* **`Requested float16 compute type…`** — Pascal has no efficient fp16.
  Leave precision on Auto.
* **`Unknown Artist`** — name the file `Artist - Title`.
* **torchcodec “entry point not found”** — `fix_torchcodec.bat` (~2 MB),
  do not reinstall the whole venv.
* **HuggingFace timeouts** — the app retries and falls back to ModelScope.
  Or tick **HuggingFace mirror** under Advanced.


