# dj-set-manager

A Python tool that automates the curation of DJ sets from a large music library. It analyses tracks using AI-powered genre and energy models, filters them by BPM, genre, energy level and key, copies the selected tracks to a destination folder, writes enriched ID3 tags, and synchronises Mixed In Key cue-point sidecars — all in a single pipeline.

---

## Table of Contents

- [Features](#features)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Windows](#windows)
  - [WSL (Windows Subsystem for Linux)](#wsl-windows-subsystem-for-linux)
- [Configuration](#configuration)
- [Running the Script](#running-the-script)
  - [Windows](#running-on-windows)
  - [WSL](#running-in-wsl)
- [Pipeline Phases](#pipeline-phases)
- [Data Files](#data-files)

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

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | **3.11** | Essentia wheels are only published for CPython 3.11 |
| pip | latest | Bundled with Python |
| [Mood Detector](https://github.com/andrasfuchs/mood-detector) | any | Provides `music_library_cache.json` used in Phase 2 |
| Mixed In Key *(optional)* | any | Only needed for Phase 4 cue-point sidecar sync |

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

From a **PowerShell** terminal you can launch the script directly in WSL with a single command:

```powershell
wsl python3 /mnt/c/Work/dj-set-manager/dj_set_manager.py
```

Or open the WSL shell first and run it from there:

```bash
cd /mnt/c/Work/dj-set-manager
source env/bin/activate
python dj_set_manager.py
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

## Data Files

| File | Description |
|------|-------------|
| `data/music_library_cache.json` | Sample / cached Mood Detector output |
| `data/genre_refinement_results.json` | Saved Phase 1 genre analysis results |
| `data/phase4_mik_cues.json` | Exported Mixed In Key cue-point data |
| `models/*.pb` | TensorFlow SavedModel graph files for embedding and genre classification |
| `models/*.json` | Label files corresponding to each `.pb` model |
