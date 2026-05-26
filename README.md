# dj-set-manager

A Python tool that automates the curation of DJ sets from a large music library. It analyses tracks using AI-powered genre and energy models, filters them by BPM, genre, energy level and key, copies the selected tracks to a destination folder, writes enriched ID3 tags, and synchronises Mixed In Key cue-point sidecars — all in a single pipeline.

---

## Table of Contents

- [Background & Motivation](#background--motivation)
- [Full DJ Workflow](#full-dj-workflow)
- [Features](#features)
- [How It Works](#how-it-works)
- [Why These Tools?](#why-these-tools)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Windows](#windows)
  - [WSL (Windows Subsystem for Linux)](#wsl-windows-subsystem-for-linux)
- [Configuration](#configuration)
- [Running the Script](#running-the-script)
  - [Windows](#running-on-windows)
  - [WSL](#running-in-wsl)
- [Pipeline Phases](#pipeline-phases)
- [Exporting to USB for Pioneer CDJs](#exporting-to-usb-for-pioneer-cdjs)
- [Known Limitations](#known-limitations)
- [Data Files](#data-files)

---

## Background & Motivation

This project was born from the challenge of curating a DJ set from a library of tens of thousands of tracks. The goal was to select roughly 500 tracks that are strong candidates for a live set on **Pioneer CDJ-2000NXS2** players, based on a handful of manually chosen favourites.

Off-the-shelf tools each solve part of the problem but none covers it end-to-end:

- **Mood Detector** quickly assigns mood, energy, BPM, and key to every file in a large library — but its genre classification is rule-based and not nuanced enough for electronic music sub-genres.
- **Mixed In Key (MIK)** provides industry-standard harmonic key detection (Camelot Wheel) and automatic hot-cue generation — but it struggles with batches larger than ~200 files, its ID3 comment-tag writer is buggy in version 11.1, and it has no concept of percentage-based genre certainty.
- **Essentia's *discogs-effnet* model** was trained on over two million Discogs tracks and outputs probability scores for 400 sub-genres (including granular electronic styles like Detroit Techno, Minimal Techno, and Acid). It runs entirely locally with no rate limits.
- **AlphaTheta Rekordbox 7 (free / Export Mode)** is the only tool needed to write the Pioneer CDJ proprietary database onto a USB drive. The paid tiers add nothing required for this workflow.

`dj-set-manager` acts as the glue layer that orchestrates all of these tools in the correct order.

---

## Full DJ Workflow

The script sits in the middle of a broader preparation pipeline. Here is the complete end-to-end workflow from raw library to CDJ-ready USB stick:

```
[23 000+ track library]
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 1 — Mood Detector                                     │
│  Analyses every file (BPM, energy, mood, key).              │
│  Outputs: music_library_cache.json                          │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 2 — dj-set-manager Phase 1 (run in WSL)               │
│  Runs each MP3 through Essentia's discogs-effnet model to   │
│  extract percentage-based electronic sub-genre scores.      │
│  Outputs: genre_refinement_results.json                     │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 3 — dj-set-manager Phase 2 (run in WSL)               │
│  Filters by BPM, energy, genre profile and copies           │
│  ~500 matching tracks to the destination folder.            │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 4 — Mixed In Key 11 (Windows, batches ≤ 200 tracks)   │
│  Provides accurate Camelot keys, energy ratings, and        │
│  generates structural hot-cue points.                       │
│  Export: select all tracks → Export to CSV                  │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 5 — dj-set-manager Phase 3 & 4 (run in WSL)           │
│  Phase 3: writes Essentia genres, Mood Detector energy,     │
│           and MIK key/BPM into ID3 tags via mutagen.        │
│  Phase 4: reads MIK sidecar files and syncs TKEY tags;      │
│           exports cue-point JSON.                           │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 6 — Rekordbox 7 (free / Export Mode, Windows)         │
│  Import the destination folder. Rekordbox reads the         │
│  ID3 tags written in Phase 3 automatically.                 │
│  Use the XML bridge to import MIK cue points:               │
│    File → Export collection in XML format →                 │
│    open in MIK → Export Cue Points →                        │
│    refresh XML tree in Rekordbox → re-import tracks.        │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 7 — USB Export (Rekordbox → FAT32 USB drive)          │
│  Sync the final playlist to a FAT32-formatted USB.          │
│  Plug into Pioneer CDJ-2000NXS2 — done.                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Features

- **AI genre extraction** using Essentia + Discogs EfficientNet TensorFlow models
- **Multi-criteria filtering**: BPM range, energy level, genre profile, and Camelot key
- **Batch file copy** with progress and ETA reporting
- **ID3 tag enrichment**: writes genre, BPM, key (Camelot notation) and comments
- **Mixed In Key sidecar sync**: exports and applies cue-point data from `.cue`/`.txt`/`.csv`/`.xml` sidecars
- **Resumable Phase 1**: periodically saves genre analysis results so a long run can be continued
- **Cross-platform paths**: automatically translates Windows drive paths to `/mnt/<drive>/…` when running inside WSL

---

## How It Works

The pipeline is split into five phases that are run interactively one after another:

| Phase | Name | Description |
|-------|------|-------------|
| 1 | Genre Extraction | Loads each MP3 through the Essentia embedding model and a genre classifier, then saves per-track genre percentages to a JSON file |
| 2 | Filter & Copy | Reads the Mood Detector cache and Phase 1 results, applies BPM / energy / genre filters, and copies matching MP3s to the destination folder |
| 3 | ID3 Write | Writes enriched tags (genre list, BPM, Camelot key, energy comment) to every copied file |
| 4 | Mixed In Key Sync | Reads Mixed In Key sidecar files and updates key tags; exports a cue-point JSON for the destination set |
| 5 | Rekordbox Import | Manual step — import the destination folder into Rekordbox or your preferred DJ software |

---

## Why These Tools?

### Essentia + discogs-effnet for genre detection

Essentia (maintained by the Music Technology Group, latest release May 2026) is the best fully local, free option for percentage-based multi-label genre classification. The *discogs-effnet-bs64* embedding model combined with the *genre_discogs400* classifier was trained on over two million Discogs tracks covering 400 sub-genres. It is especially strong on electronic music because Discogs is heavily used by electronic music collectors and labels.

Unlike rule-based tools, the model outputs a probability score (0–1) for every sub-genre simultaneously. A single track might score 0.80 Techno, 0.25 Detroit Techno, and 0.22 Minimal Techno — exactly the kind of nuanced, overlapping classification needed for advanced set curation. This script keeps only the `Electronic---*` labels above 20 % confidence and stores them as a percentage list in the ID3 `TCON` tag (e.g. `Techno (80%), Detroit Techno (25%), Minimal Techno (22%)`).

### Mood Detector for energy and BPM pre-filtering

[Mood Detector](https://github.com/andrasfuchs/mood-detector) analyses the full library cheaply (~2 seconds per track) and provides energy levels and BPM. Its energy estimates are more accurate than Mixed In Key's 1–10 scale for the purpose of filtering a large collection because they are expressed as continuous floats (0.0–1.0) derived directly from audio features rather than rounded to integer steps.

### Mixed In Key for harmonic key and cue points

Mixed In Key is the industry standard for Camelot-Wheel harmonic key detection. Its key accuracy is significantly higher than librosa-based methods (used by Mood Detector) because it performs deeper structural harmonic analysis. It also auto-generates hot-cue points at musical transitions (drops, breakdowns, intros). Because MIK's ID3 comment-tag writer is buggy in version 11.1, this script reads the key and BPM values from MIK's sidecar files and applies them to the ID3 tags itself via `mutagen`.

### Rekordbox free / Export Mode

The free tier of AlphaTheta Rekordbox 7 fully supports Export Mode, which is the only feature needed to write the Pioneer CDJ proprietary database to a USB drive. The paid tiers (Performance and Creative plans) add DJ software controller features that are irrelevant when playing from CDJs.

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | **3.11** | Essentia wheels are only published for CPython 3.11 |
| pip | latest | Bundled with Python |
| [Mood Detector](https://github.com/andrasfuchs/mood-detector) | any | Provides `music_library_cache.json` used in Phase 2 |
| [Mixed In Key](https://mixedinkey.com/) *(optional)* | **11** | Phase 4 key sync; export your library as CSV or use sidecar files |
| [Rekordbox](https://rekordbox.com/) *(optional)* | **7.2+** | Free Export Mode is sufficient; used after Phase 4 to build the CDJ USB |
| WSL 2 + Ubuntu | any | Recommended runtime; Essentia TensorFlow support is more stable on Linux |

> **Why Python 3.11?**  
> The `essentia` package distributes pre-compiled TensorFlow wheels only for CPython ≤ 3.11. The `requirements.txt` file enforces this with a `python_version < "3.12"` marker.

---

## Installation

### Windows

1. **Install Python 3.11**  
   Download the installer from <https://www.python.org/downloads/release/python-3110/> and make sure *"Add Python to PATH"* is checked.

2. **Clone the repository**

   ```powershell
   git clone https://github.com/andrasfuchs/dj-set-manager.git
   cd dj-set-manager
   ```

3. **Create and activate a virtual environment**

   ```powershell
   python -m venv env
   .\env\Scripts\Activate.ps1
   ```

   > If you see an execution-policy error, run:  
   > `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`

4. **Install dependencies**

   ```powershell
   pip install -r requirements.txt
   ```

5. **Verify Essentia is available**

   ```powershell
   python -c "import essentia; print(essentia.__version__)"
   ```

---

### WSL (Windows Subsystem for Linux)

Essentia's TensorFlow support works most reliably under Linux. Running the script in WSL is the **recommended** approach.

#### 1 — Enable WSL and install a distro

Open PowerShell **as Administrator** and run:

```powershell
wsl --install
```

This installs WSL 2 and Ubuntu by default. Restart when prompted, then complete the Ubuntu first-run setup (create a UNIX username and password).

#### 2 — Install Python 3.11 inside WSL

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3.11-dev python3-pip
```

Confirm the version:

```bash
python3.11 --version
```

#### 3 — Clone the repository (inside WSL)

Windows drives are automatically mounted at `/mnt/<letter>/` inside WSL. You can clone directly into your Windows workspace so files are accessible from both environments:

```bash
cd /mnt/c/Work
git clone https://github.com/andrasfuchs/dj-set-manager.git
cd dj-set-manager
```

Or, if you have already cloned on Windows, simply navigate to the existing folder:

```bash
cd /mnt/c/Work/dj-set-manager
```

#### 4 — Create and activate a virtual environment

```bash
python3.11 -m venv env
source env/bin/activate
```

#### 5 — Install dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

#### 6 — Verify Essentia

```bash
python -c "import essentia; print(essentia.__version__)"
```

---

## Configuration

Open `dj_set_manager.py` and adjust the variables in the `--- CONFIGURATION ---` section near the top of the file:

| Variable | Default | Description |
|----------|---------|-------------|
| `cache_file` | `c:\Work\Mood-Detector\music_library_cache.json` | Path to the JSON cache produced by Mood Detector |
| `source_folder` | `R:\Media\Music\GroovyTunes\MP3` | Root folder of your music library (set to `None` to scan everywhere) |
| `destination_folder` | `r:\Media\Music\GroovyTunes\2026-05-25_Selected` | Folder where matching tracks will be copied |
| `phase1_output_file` | `r:\Media\Music\GroovyTunes\genre_refinement_results.json` | Where Phase 1 genre results are saved |
| `phase4_cue_export_file` | `r:\Media\Music\GroovyTunes\phase4_mik_cues.json` | Where Phase 4 cue data is exported |
| `allowed_camelot_keys` | `("7A","8A","9A","10A","11A")` | Camelot keys that are allowed through the filter |
| `phase2_preferred_genres` | Techno, Minimal Techno, Tech Trance, … | Set of genre names considered a match |
| `phase2_min_genre_percentage` | `20.0` | Minimum confidence % for a genre to count |

> **Windows vs. WSL paths**  
> You can use Windows-style paths (e.g. `r:\Media\Music\...`) in the configuration even when running inside WSL — the script automatically converts them to `/mnt/r/Media/Music/...` at runtime.

---

## Running the Script

### Running on Windows

```powershell
cd C:\Work\dj-set-manager
.\env\Scripts\Activate.ps1
python dj_set_manager.py
```

### Running in WSL

> **WSL is the recommended runtime.** Essentia's TensorFlow backend has a compatibility issue on Windows that prevents it from running correctly outside WSL.

From a **PowerShell** terminal, launch the script directly in WSL with this exact command:

```powershell
wsl env TF_CPP_MIN_LOG_LEVEL=2 python3 /mnt/r/Media/Music/GroovyTunes/Scripts/dj_set_manager.py
```

> `TF_CPP_MIN_LOG_LEVEL=2` suppresses TensorFlow's verbose C++ info and warning messages so that only the script's own output is visible in the terminal.

Adjust the path after `python3` to match wherever you have stored `dj_set_manager.py`. If you cloned the repository to `C:\Work\dj-set-manager`, the equivalent path inside WSL is `/mnt/c/Work/dj-set-manager/dj_set_manager.py`.

Alternatively, open a WSL shell first and run it from there:

```bash
source /mnt/c/Work/dj-set-manager/env/bin/activate
TF_CPP_MIN_LOG_LEVEL=2 python3 /mnt/c/Work/dj-set-manager/dj_set_manager.py
```

When the script starts, it will ask you interactively which phases to run:

```
Run Phase 1 (Genre extraction)? [Y/n]:
Run Phase 2 (Filtering and copy)? [Y/n]:
Run Phase 3 (ID3 write)? [Y/n]:
Run Phase 4 (Mixed In Key sync)? [Y/n]:
```

Press **Enter** (or type `Y`) to run a phase, or type `N` to skip it.

---

## Pipeline Phases

### Phase 1 — Genre Extraction

- Scans every MP3 in `source_folder` that is not already present in `phase1_output_file`
- Generates a 1-second-window audio embedding using the *discogs-effnet-bs64* model
- Classifies the embedding with the *genre_discogs400-discogs-effnet* classifier
- Keeps only `Electronic---*` labels with confidence ≥ 20 %
- Saves results to `phase1_output_file` every `phase1_save_every_tracks` tracks or every `phase1_save_every_seconds` seconds, so the run is resumable

### Phase 2 — Filter & Copy

- Loads `cache_file` (Mood Detector output) and the Phase 1 genre lookup
- Applies all filters simultaneously: BPM 120–128, energy 0.5–0.79, genre profile match
- Copies matching files to `destination_folder`, skipping files that already exist there

### Phase 3 — ID3 Tag Write

- Writes the following ID3 frames to every file in the destination folder:
  - `TCON` — genre list with percentages
  - `TBPM` — rounded BPM
  - `TKEY` — Camelot key (e.g. `08A`)
  - `COMM` — energy value comment

### Phase 4 — Mixed In Key Sync

- Searches `destination_folder` (or `phase4_mik_export_search_root`) for Mixed In Key sidecar files (`.cue`, `.txt`, `.csv`, `.xml`)
- Updates `TKEY` tags from MIK key data when available
- Exports all cue-point entries to `phase4_cue_export_file` as JSON

### Phase 5 — Rekordbox Import *(manual)*

After Phase 4 completes, import the contents of `destination_folder` into Rekordbox (or another DJ application) and build your set.

---

## Exporting to USB for Pioneer CDJs

After Phase 4 is complete, the destination folder contains fully tagged MP3s ready to import into Rekordbox.

### 1 — Import into Rekordbox

1. Open Rekordbox 7 (free tier).
2. Drag the destination folder into the **Collection** panel. Rekordbox will read the `TCON`, `TBPM`, and `TKEY` ID3 tags written by Phase 3 automatically.
3. Analyse the tracks in Rekordbox to build beatgrids (`Track` → `Analyze Tracks`).

### 2 — Bridge MIK cue points via XML

Because hot-cue points cannot be stored in ID3 tags, an XML bridge is needed:

1. In Rekordbox: **File → Export collection in XML format**. In Preferences, set *Export BeatGrid Information* to `BPM Change Points`.
2. Open Mixed In Key 11, navigate to **Personalise → Export Cue Points**, and point it at the XML file you just exported.
3. Back in Rekordbox, open the XML tree in the browser, right-click the updated tracks, and choose **Import to Collection** to fuse the cue points.

### 3 — Format the USB drive (FAT32)

Pioneer CDJ-2000NXS2 players require **FAT32** — they do **not** support exFAT. Windows 11 artificially limits FAT32 formatting to 32 GB in the GUI. Use one of these methods for a 256 GB+ drive:

**Option A — PowerShell (no extra software needed)**

```powershell
# Run PowerShell as Administrator
# Replace 'X' with your USB drive letter and adjust the label
format X: /FS:FAT32 /Q /V:DJUSB
```

**Option B — Rufus (free GUI tool, recommended)**

1. Download [Rufus](https://rufus.ie/) — a single portable `.exe`, no installation required.
2. Select your USB drive, set *Boot selection* to `Non bootable`, and choose `FAT32` as the file system.
3. Click **Start**.

> The only practical disadvantage of FAT32 is a **4 GB per-file size limit**. This is irrelevant for individual MP3 tracks.

### 4 — Sync to USB via Rekordbox Export Mode

1. In Rekordbox, switch to **Export** mode (top-left mode selector).
2. Connect the FAT32 USB drive.
3. Drag your final playlist from the Collection onto the USB device in the left panel.
4. Rekordbox writes the Pioneer proprietary database (`PIONEER/` folder) to the USB.
5. Safely eject and plug into your CDJ-2000NXS2 — waveforms, cue points, and key/BPM data will load instantly.

---

## Known Limitations

| Limitation | Details |
|------------|---------|
| **Python 3.11 only** | Essentia TensorFlow wheels are not published for Python 3.12+. Use the `env/` virtual environment created with Python 3.11. |
| **Mixed In Key comment tag bug** | MIK 11.1's built-in ID3 comment writer is buggy. This script intentionally reads MIK data from sidecar files and writes tags itself via `mutagen` as a workaround. |
| **MIK batch size** | Mixed In Key may freeze or crash if fed more than ~200 files at once. Process your selected tracks in batches before running Phase 4. |
| **Electronic genres only** | Phase 1 discards any classification label that does not begin with `Electronic---`. Tracks from other genres will have an empty genre list after Phase 1. |
| **Cue points require the XML bridge** | ID3 tags cannot carry hot-cue data. Hot cues generated by MIK must be transferred to Rekordbox manually via the XML export/import bridge described above. |
| **FAT32 4 GB file limit** | The CDJ USB must be FAT32. Individual MP3 files are never near 4 GB, but WAV/AIFF files from very long mixes could exceed this limit. |

---

## Data Files

| File | Description |
|------|-------------|
| `data/music_library_cache.json` | Sample / cached Mood Detector output |
| `data/genre_refinement_results.json` | Saved Phase 1 genre analysis results |
| `data/phase4_mik_cues.json` | Exported Mixed In Key cue-point data |
| `models/*.pb` | TensorFlow SavedModel graph files for embedding and genre classification |
| `models/*.json` | Label files corresponding to each `.pb` model |
