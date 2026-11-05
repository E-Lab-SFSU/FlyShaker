# Fly Shaker

Fly Shaker is an affordable, robust system for researchers to conduct sleep deprivation assays on drosophila melanogaster (fruit flies). It integrates automated mechanical stimulation, continuous camera imaging, and computer-assisted activity monitoring to investigate sleep, wakefulness, and behavioral responses in experimental populations. Designed primarily for a Raspberry Pi 5 with an ESP32-controlled shaker, Fly Shaker provides a graphical interface for configuring experiments, calibrating wells and motion thresholds, tracking activity, and generating reproducible datasets. A Windows demonstration mode is available without physical hardware.

> **Version:** 1.7. Hardware behavior has not yet been validated on the Raspberry Pi/ESP32 for this build. In particular, burst detection depends on matching the ESP32's actual serial messages. Automated headless tests pass, but GUI and hardware testing remain necessary.

## Experiment Files and Data Organization

Fly Shaker creates a separate `expXXXXX` directory for each experiment under the configured output root (`~/FS_IMG` by default). Unlike a per-well `.npy` recording system, this version stores sequential PNG images and CSV-based per-well measurements.

The following tree illustrates the expected structure; `exp00001` is an example identifier, and some outputs depend on the chosen workflow and successful completion of analysis.

```text
FS_IMG/
└── exp00001/
    ├── experiment_settings.json
    ├── activity_config_used.py
    ├── initiation_frames/
    │   ├── frame_000001.png
    │   ├── frame_000002.png
    │   ├── ...
    │   ├── initiation_activity.csv
    │   └── initiation_plate.csv
    ├── frames/
    │   ├── frame_000001.png
    │   ├── frame_000002.png
    │   ├── SK_frame_000003.png
    │   └── ...
    ├── initiation_timestamps_exp00001.csv
    ├── image_timestamps_exp00001.csv
    ├── shaker_timestamps_exp00001.csv
    ├── plate_wells_exp00001.csv
    ├── well_threshold.csv
    ├── median_background.png
    ├── exp00001.csv
    ├── exp00001_plate.csv
    ├── sleep_timer.csv
    ├── sleep_data.csv
    ├── image_shaker_alignment.csv
    ├── shake_flag_report.txt
    ├── exp00001_first_last_delta.csv
    ├── exp00001_first_last_yellow_red_green_labeled.png
    ├── exp00001_first_last_2x2.png
    ├── experiment.mp4
    ├── plots/
    │   ├── A_plate_sleep_wake_raster.png
    │   ├── B_percent_awake_vs_time.png
    │   ├── C_stimulus_triggered_average_percent_awake.png
    │   ├── D_summary_statistics.png
    │   ├── stimulus_times.csv
    │   └── plot_summary.txt
    └── reanalysis_YYYYMMDD_HHMMSS/   # only when reanalysis is requested
        └── ...
```

### File descriptions

| File or directory | Purpose |
|---|---|
| `experiment_settings.json` | Snapshot of experiment parameters, including shaker settings, burst configuration, and death-timer value. |
| `activity_config_used.py` | Snapshot of the CAMS analysis configuration used for this experiment. |
| `initiation_frames/` | Images captured before the experiment for grid, background, and threshold calibration. |
| `frames/` | Images recorded during the actual shaker experiment. |
| `SK_frame_*.png` | Images identified as occurring during a shaking burst, including a one-second buffer on each side. Preserved for video but excluded from CAMS motion analysis. |
| `initiation_timestamps_*.csv`, `image_timestamps_*.csv` | Capture-time records for calibration and experimental images. |
| `shaker_timestamps_*.csv` | ESP32 shaker events, serial messages, and available sequence information. |
| `plate_wells_*.csv` | Well locations, identifiers, and image-region geometry. |
| `well_threshold.csv` | Per-well motion thresholds, fly counts, and viability values. |
| `median_background.png` | Background reference image for optical activity measurement. |
| `initiation_activity.csv`, `initiation_plate.csv` | Activity data generated during initiation/calibration. |
| `expXXXXX.csv` | Per-well optical activity, movement classification, inactivity time, and sleep/wake states. |
| `expXXXXX_plate.csv` | Plate-level activity, counts, and percentages of sleeping/awake wells. |
| `sleep_timer.csv` | Live per-image motion and sleep/awake tracking. |
| `sleep_data.csv` | Periodic/final summary of official CAMS sleep classification. |
| `image_shaker_alignment.csv` | Alignment of camera timestamps with shaker events. |
| `shake_flag_report.txt` | Counts of flagged shaking frames and any burst-detection warnings. |
| `expXXXXX_first_last_*` | First-versus-last frame comparison measurements and images. |
| `experiment.mp4` | Video generated from the experiment's recorded frames, including shaking frames. |
| `plots/` | Sleep/wake rasters, time-series plots, stimulus-response plots, summary statistics, and stimulus timing records. |
| `reanalysis_*/` | New analysis outputs from an existing experiment, kept separate from original results. |

## All Necessary Features

### Feature table

| Feature | Description | v1.7 status |
|---|---|---|
| Graphical experiment setup | Configure imaging, shaker parameters, duration, initiation, and calibration options. | Implemented |
| Raspberry Pi camera acquisition | Capture timestamped PNG images and maintain a live preview. | Implemented; hardware validation needed |
| ESP32 shaker integration | Send shaker configuration and start/stop commands through serial communication. | Implemented; hardware validation needed |
| Shaker parameters | Frequency, amplitude, duty cycle, burst, burst silence, sequence duration, and sequence silence. | Implemented |
| Experiment initiation | Capture calibration images before starting the timed experiment. | Implemented |
| Well-grid calibration | Define and inspect regions of interest for individual wells. | Implemented |
| Per-well motion thresholds | Determine activity thresholds and exclude invalid wells. | Implemented |
| Reuse calibration | Reuse well grids and thresholds, or selectively recalibrate wells. | Implemented |
| Standalone threshold calibration | Run calibration using existing images or a new initiation capture without starting the shaker. | Implemented |
| Continuous experiment imaging | Save frames throughout the shaker session. | Implemented |
| Live sleep/awake display | Show per-well activity-based sleep classification during recording. | Implemented |
| Periodic CAMS analysis | Recompute official CAMS activity and sleep classification every 300 experiment images. | Implemented |
| Shaking-frame exclusion | Flag burst-associated frames and omit them from motion classification without deleting them. | Implemented; ESP burst parsing needs validation |
| Sleep timing across gaps | Preserve confirmed inactivity across shaking gaps without counting unobserved time. | Implemented |
| Death timer | Store a configurable time value for future death detection. | Setting saved only; no death classification yet |
| Post-experiment analysis | Generate activity CSVs, sleep statistics, comparisons, plots, and timestamp alignment. | Implemented |
| Video export | Create an MP4 from the recorded experiment images. | Implemented |
| Reanalysis | Reprocess an existing experiment without overwriting original results. | Implemented |
| Storage check | Estimate available disk space before capture. | Implemented |
| Windows demo | Simulate camera images and shaker events without hardware. | Implemented |

## Pipeline Process

The experiment pipeline combines initiation, calibration, shaker execution, camera recording, and CAMS analysis in one workflow. **Calibration and recording are not independent experiments:** under normal operation, the application proceeds automatically from calibration to the shaker experiment after the user confirms the required steps.

1. **Configure the experiment.** Open Fly Shaker and set the image capture rate, shaker frequency/amplitude, duty cycle, burst and silence parameters, session duration, initiation duration, and calibration mode. Choose whether to calibrate from scratch or reuse prior settings.
2. **Validate settings and storage.** The application checks parameter ranges and estimates disk space. If the check fails, experiment creation is stopped before capture begins.
3. **Create an experiment directory.** A new `expXXXXX/` folder is created with `initiation_frames/` and `frames/`. Experiment settings and the CAMS configuration are recorded.
4. **Begin initiation imaging.** The camera opens and captures images into `initiation_frames/` for the specified initiation period. These images establish the current imaging conditions before shaker operation.
5. **Confirm the calibration path.** After initiation, the application offers calibration options, including a separate thresholding-and-exit path. For a normal experiment, continue to calibration.
6. **Calibrate the well grid.** Use the CAMS well-grid tuner to identify the positions and boundaries of wells. When reusing a grid, confirm its alignment with the current image.
7. **Measure background activity and set thresholds.** CAMS estimates a median background and activity levels, then opens the motion-threshold tuner for wells requiring calibration. Existing thresholds can be reused or selected wells recalibrated. The application checks that wells are calibrated or explicitly excluded.
8. **Start the actual experiment.** Calibration activity outputs are archived in `initiation_frames/`. The camera continues operating but switches image saving to `frames/`. The configured shaker session is started through the ESP32 interface.
9. **Record and monitor.** Experiment images and capture timestamps are saved continuously while shaker events are logged. The preview displays elapsed time, shaker information, and live sleep/awake status based on per-well movement.
10. **Run periodic analysis.** Every 300 experimental images, CAMS measures activity and classifies sleep to update official results. The live sleep timer and official CAMS outputs are distinct: live display updates per image, while official CAMS analysis runs in batches.
11. **Handle shaking intervals.** Images associated with shaker bursts, plus a one-second buffer before and after, are excluded from motion analysis. The program does not award inactivity time for unmeasured intervals. At finalization, identified images are renamed with the `SK_` prefix but retained.
12. **Finish the session and analyze.** The shaker and image saving stop. CAMS performs final activity measurement and sleep classification, compares first/last images, generates plots, creates `experiment.mp4`, and aligns camera/shaker timestamps.
13. **Review or reanalyze.** Inspect CSVs, plots, and video in the experiment folder. The separate reanalysis workflow can create new results from existing experiment images without overwriting the original analysis.

**Alternative path — calibration only:** After initiation, choose **Separate thresholding (save and exit)** to retain calibration images, grid, activity measurements, and thresholds without starting the shaker or recording an experimental session. The Settings window also offers **Motion threshold calibration only...**.

**Sleep interpretation:** A well is classified as asleep after 300 seconds of confirmed inactivity in the official CAMS analysis. The live display also uses a provisional 30-second criterion during the first five minutes. Shaking-associated image gaps are excluded from confirmed inactivity accumulation. Optical activity is measured at the well level and is not a direct measurement of each individual fly.

## Programs

### Main application and supporting modules

| Program | Purpose |
|---|---|
| `main.py` | Main PyQt6 interface and orchestration of initiation, calibration, recording, monitoring, and finalization. |
| `camera_service.py` | Camera connection, image acquisition, timestamps, and synthetic camera frames in demo mode. |
| `shaker_service.py` | ESP32 serial communication, shaker commands, event logging, and demo shaker simulation. |
| `calibration_tools.py` | Helpers for copying, validating, and reusing well-grid and threshold calibration. |
| `standalone_calibration.py` | Supports calibration workflows separate from a full experiment. |
| `config_writer.py` | Configures CAMS input/output paths and timing for the current experiment. |
| `live_engine.py` | Per-image live motion/sleep tracking and shaking-window handling. |
| `timestamp_sync.py` | Aligns camera image timestamps with shaker event timestamps. |
| `disk_check.py` | Estimates required storage and verifies available disk space. |
| `video_tools.py` | Converts experiment frames into an MP4 using the configured image rate and playback-speed rules. |
| `img_to_vid.py` | Reusable image-to-video creation helper. |
| `reanalysis.py` | Re-runs the CAMS analysis pipeline on previously recorded experiments. |
| `ui_reanalysis.py` | Graphical dialog for selecting and processing existing experiments. |
| `reference_RASPI_Imaging_v4.py` | Reference Raspberry Pi imaging implementation. |
| `reference_shakerTerminal_v6.py` | Reference shaker terminal implementation. |

### CAMS analysis programs (`cams/`)

| Program | Purpose |
|---|---|
| `activity_config.py` | Analysis settings such as experiment paths, timing, and activity parameters. |
| `well_grid_tuner.py` | Interactive identification and calibration of plate wells. |
| `measure_activity.py` | Measures optical activity from image differences and generates activity/background outputs. |
| `motion_threshold_tuner.py` | Interactive calibration of per-well movement thresholds. |
| `classify_sleep.py` | Converts measured activity into movement, inactivity, and sleep/awake classifications. |
| `sleep_timing.py` | Applies inactivity timing rules across gaps in measurable frames. |
| `shake_frames.py` | Interprets shaker burst logs and identifies/flags affected images. |
| `compare_first_last.py` | Compares initial and final experimental images for visual assessment. |
| `generate_plots.py` | Creates activity, sleep/wake, and stimulus-response visualizations. |
| `extract_frames.py` | CAMS utility for extracting frames from video. |

### Launchers, dependencies, and tests

| File | Purpose |
|---|---|
| `run_raspberry_pi.sh` | Starts Fly Shaker on Raspberry Pi. |
| `launch_flyshaker.sh` | Desktop-oriented launcher/setup script. |
| `FlyShaker.desktop` | Raspberry Pi desktop shortcut. |
| `run_windows_demo.bat` | Starts the Windows demonstration workflow. |
| `requirements-pi.txt` | Raspberry Pi Python dependencies. |
| `requirements-windows.txt` | Windows demo Python dependencies. |
| `tests/test_shake_flow.py` | Automated tests for shaker-frame and sleep-timing behavior. |
| `CHANGES_V1_7.md` | Version-specific changes, corrections, and known limitations. |

## Running Fly Shaker

### Raspberry Pi 5

Run these commands from the application directory on Raspberry Pi OS:

```bash
sudo apt update
sudo apt install -y python3-venv python3-opencv libxcb-cursor0
python3 -m venv --system-site-packages .venv
source .venv/bin/activate
pip install -r requirements-pi.txt
./run_raspberry_pi.sh
```

The default hardware device paths are `/dev/video0` (camera) and `/dev/ttyUSB0` (ESP32); verify these against the connected hardware. If serial access is denied, add the current user to the `dialout` group and log out/in:

```bash
sudo usermod -aG dialout "$USER"
```

### Windows demonstration

From PowerShell in the application directory, with Python installed and available on PATH:

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements-windows.txt
.\run_windows_demo.bat
```

The demo simulates a well plate and shaker activity. It is intended for interface/workflow testing, not biological data collection.

### Output location

The default output directory is `~/FS_IMG`. Set `FLYSHAKER_ROOT` before launching to save to another location, such as an external drive:

```bash
export FLYSHAKER_ROOT=/media/pi/MyDrive/FS_IMG
```

### Tests

```bash
python -m unittest discover -s tests -v
```

## Notes and Current Limitations

- **ESP32 burst parsing must be verified on actual hardware.** The code currently expects burst-related messages resembling `BURST START`, `BURST DURATION n`, and `BURST SILENCE START`. Review `shake_flag_report.txt` and compare against the actual serial output. You can inspect a recorded log with `python cams/shake_frames.py <path-to-shaker-log.csv>`.
- **Death timer is not a mortality classifier.** The value is recorded in `experiment_settings.json` but is not used to determine death in v1.7.
- **Some stimulus plots may have gaps** when their exact requested sampling frames fall inside excluded shaking intervals.
- **Abrupt application closure can bypass finalization**, including `SK_` renaming and final plots/video. Existing timestamps and logs still support shaking-frame exclusion during reanalysis.
- **Calibration reuse requires physical alignment.** Reconfirm the grid if the camera or plate has moved.

For implementation details, consult `CHANGES_V1_7.md`, `cams/README.md`, and `cams/docs/`.
