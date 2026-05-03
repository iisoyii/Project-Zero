# Project Zero

Desktop app for splitting audio/video files into stems (vocals, drums, bass, guitar, piano, other), mixing them with per-stem controls, and exporting as a single WAV/MP3.

**Latest:** [v0.8.0](https://github.com/iisoyii/Project-Zero/releases/tag/v0.8.0) — higher-fidelity 6-stem separation (overlap=0.5, multi-threaded CPU), musical key + Camelot detection, per-stem waveform visualizer, Auto-Key Sync in Mashup Lab, 5-second crossfade between tracks, Ensemble Engine auto-refine. Download from the [landing page](https://iisoyii.github.io/Project-Zero/).

- **Shell:** Electron (Windows .exe)
- **Frontend:** React + Vite + Tailwind (dark "studio" theme)
- **Backend:** FastAPI + Demucs + ffmpeg + librosa, launched as a local subprocess by Electron

## Prerequisites

1. **Python 3.10+** and **Node.js 18+**
2. **ffmpeg** on PATH — https://ffmpeg.org/download.html
3. **PyTorch** matching your hardware — https://pytorch.org/get-started/locally/ (CPU works, a CUDA GPU is much faster)

## First-time setup

```powershell
# Backend (one-time)
cd backend
python -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
deactivate

# Frontend (one-time)
cd ..\frontend
npm install
```

The Demucs model weights (~80 MB) are downloaded the first time you separate a track.

## Run as a desktop app (development)

From `frontend/`:

```powershell
# Terminal 1 — backend
cd ..\backend
.\.venv\Scripts\activate
uvicorn app.main:app --port 8000

# Terminal 2 — Electron + Vite dev server
cd frontend
npm run app:dev
```

`app:dev` runs Vite and Electron together; the window opens automatically.

## Build a clickable Windows app

The build chains two steps: PyInstaller bundles the backend into a single folder of native binaries, then electron-builder wraps it with the UI into an NSIS installer.

```powershell
# One-time: install PyInstaller into the backend venv
cd backend
.\.venv\Scripts\activate
pip install -r requirements-build.txt
deactivate

# Build the installer
cd ..\frontend
npm run app:build
```

The installer is written to `frontend/release/Project-Zero-Setup-<version>.exe`. Run it once, then launch **Project Zero** from the Start Menu or desktop shortcut.

The bundled backend (`backend/dist/project-zero-backend/`) contains its own Python runtime and all ML dependencies — installer recipients **do not** need Python, PyTorch, or pip. They still need **ffmpeg on PATH** (the bundle calls `ffmpeg.exe` via subprocess), and the first separation downloads the Demucs model weights (~80 MB) into the user's cache.

> The PyInstaller bundle is large (~1–2 GB depending on whether you have CUDA torch installed). If you want a smaller installer, build the backend in a venv that has the **CPU-only** PyTorch wheel.

## How it works

1. Upload an MP4/MOV/MP3/WAV/M4A. Video files are demuxed to WAV via ffmpeg.
2. The backend runs Demucs (`htdemucs_6s`, overlap=0.5) to produce 6 stems.
3. BPM is detected with `librosa.beat.beat_track`; musical key is estimated via Krumhansl-Schmuckler chroma correlation and shown on library cards as Camelot codes.
4. Per-stem amplitude peaks are pre-rendered to JSON during separation so the UI can show waveforms without re-decoding wavs.
5. The UI polls `/jobs/{id}` until status is `done`, then loads each stem in an `<audio>` element for in-browser playback with per-stem volume / mute / solo / boost / focus.
6. "Fine-Tune" applies pitch-shift (semitones) and gain (dB) on the server when you render.
7. "Export" sends the selected stems + per-stem settings + master FX (tempo, reverb, bass-boost) to `/render`, which mixes them and returns a single WAV or MP3.
8. "Mashup Lab" combines stems from any two library tracks; Auto-BPM Sync time-stretches the vocal, Auto-Key Sync pitch-shifts the vocal to the instrumental's key.
