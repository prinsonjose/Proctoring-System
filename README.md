# Proctoring System

Proctoring System is a Python desktop application that turns a webcam and microphone into a lightweight, automated exam-proctoring tool. It watches a candidate in real time, tracks head movement and ambient sound, and combines both signals into a single rolling "cheat probability" that is plotted live on screen.

## How it works

The app runs three monitoring loops in parallel threads:

- **Head pose estimation** (`src/head_pose.py`) — uses MediaPipe's Face Mesh to locate facial landmarks, solves for the head's 3D orientation with OpenCV's `solvePnP`, and flags whether the candidate is looking away from the screen (left/right/down) rather than facing forward.
- **Audio monitoring** (`src/audio.py`) — listens to the microphone via `sounddevice`, computes the rolling RMS amplitude of the input, and flags sustained loud sound (e.g. talking) as suspicious.
- **Cheat-probability detection** (`src/detection.py`) — combines the head-pose and audio flags using a weighted moving average into a single `PERCENTAGE_CHEAT` score, marks the candidate as cheating once it crosses a threshold, and plots the score live with Matplotlib.

`src/run.py` is the entry point: it starts the head-pose, audio, and detection loops as separate threads and lets them run concurrently while sharing state through module-level globals.

## Features

- Real-time head-pose tracking to detect when a candidate looks away from the screen
- Real-time audio amplitude monitoring to flag talking or background noise
- A combined, weighted "cheat probability" score that smooths out brief, isolated flags
- A live Matplotlib graph of the cheat probability over time
- Visual on-screen feedback (head-pose angle and direction) overlaid on the webcam feed

## Tech stack

- **Language:** Python
- **Computer vision:** OpenCV, MediaPipe (Face Mesh)
- **Audio:** sounddevice, NumPy
- **Visualization:** Matplotlib
- **Concurrency:** Python's `threading` module

## Project structure

```
Proctoring-System/
├── doc/
│   └── Project Report.pdf      # Written project report
├── src/
│   ├── run.py                  # Entry point — starts all monitoring threads
│   ├── head_pose.py            # Webcam capture + head pose estimation
│   ├── audio.py                # Microphone capture + amplitude (RMS) analysis
│   ├── detection.py            # Combines signals into a cheat-probability score + live plot
│   ├── graph.py                # Standalone Matplotlib demo (not wired into the main app)
│   ├── ui.py                   # Minimal, unfinished Tkinter UI stub
│   └── toDo.txt                # Known open items
├── unit_test/                  # Experimental / exploratory scripts, not part of the main app
│   ├── face-rec.py             # Face recognition demo (requires `face_recognition`)
│   ├── processes.py            # Windows-only: flags disallowed running apps (Discord, Zoom, etc.)
│   ├── screen_recorder.py       # Windows-only: records the screen with a webcam overlay
│   └── pyaudio_test.py          # PyAudio-based microphone amplitude test
└── requirements.txt
```

## Setup

### Prerequisites

- Python 3.9–3.11 (MediaPipe wheels are not available for every Python version)
- A working webcam and microphone
- On Linux, the system audio libraries required by `sounddevice` (PortAudio), e.g. `sudo apt install libportaudio2`

### Installation

```bash
git clone https://github.com/prinsonjose/Proctoring-System.git
cd Proctoring-System
python -m venv venv
source venv/bin/activate   # on Windows: venv\Scripts\activate
pip install -r requirements.txt
```

`requirements.txt` covers MediaPipe, OpenCV, and sounddevice. `src/detection.py` also imports `matplotlib`, which is not currently listed in `requirements.txt` — install it separately if needed:

```bash
pip install matplotlib
```

### Running the app

```bash
cd src
python run.py
```

This opens a webcam window showing head-pose estimation and a live graph window showing the rolling cheat-probability score. Press `Esc` in the webcam window to stop head-pose tracking; close the plot window or stop the process to end the session.

## Known limitations

- `run.py` starts threads for head pose, audio, and detection but never explicitly stops them — the listed `toDo.txt` item to "stop sound thread/stream" is still open, so the app needs to be terminated externally (e.g. `Ctrl+C` or closing the windows).
- The cheat-probability weighting in `detection.py` is hand-tuned with fixed constants rather than calibrated against real exam data.
- `src/ui.py` is an empty Tkinter stub and is not connected to the rest of the application.
- `src/graph.py` is a standalone Matplotlib demo and is not used by `run.py`.
- The `unit_test/` scripts are experiments rather than an automated test suite: `processes.py` and `screen_recorder.py` are Windows-only (they depend on `wmi` and `win32api`), and `face-rec.py` requires the separate `face_recognition` package, none of which are listed in `requirements.txt`.
- There is no automated test suite, and the project assumes a single candidate with one face in frame.

## License

No license file is included in this repository. Add one (e.g. MIT, Apache-2.0) if you intend to share or distribute this code.
