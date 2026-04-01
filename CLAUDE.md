# STRF — Developer Reference

STRF (Satellite Tracking Radio Framework) is a C-based SDR toolkit for tracking satellites via Doppler-shift analysis of radio observations. It takes raw IQ recordings, generates spectrograms, extracts Doppler measurements, and fits or refines Two-Line Element (TLE) orbital parameters.

---

## Environment Variables

Set these before use (see `env-dist` for a template):

| Variable | Purpose |
|----------|---------|
| `ST_COSPAR` | Observer COSPAR site ID (4-digit integer) |
| `ST_TLEDIR` | Path to directory containing TLE catalog files |
| `ST_DATADIR` | Path to strf data directory (must contain `data/sites.txt`, `data/frequencies.txt`) |
| `ST_SITES_TXT` | Path to `sites.txt` observer location database |
| `ST_LOGIN` | space-track.org credentials (for `tleupdate`) |

---

## Core Data Pipeline

```
.cf32 (raw IQ)
    ↓  rffft
.bin (timestamped spectrograms)
    ↓  rffind
.dat (Doppler measurements)
    ↓  rffit
improved TLE
```

---

## Tool Reference

| Tool | Input | Output | Interactive |
|------|-------|--------|-------------|
| `rffft` | `.cf32` / raw IQ | `.bin` spectrograms | No |
| `rffind` | `.bin` | `.dat` Doppler points | No |
| `rffit` | `.dat` + TLE | improved TLE | Yes (PGPLOT) |
| `rfplot` | `.bin` | PNG / annotations | Yes (PGPLOT) |
| `rfpng` | `.bin` | PNG waterfall | No |
| `rfdop` | `.bin` | Doppler curves | No |
| `rfedit` | `.bin` | edited `.bin` | Yes (PGPLOT) |

---

## rffft — Spectrogram Generation

Converts raw IQ data into binary spectrogram files.

```bash
rffft -i <file.cf32> -F float -f <center_hz> -s <samplerate_hz> \
      -T <YYYY-MM-DDTHH:MM:SS.sss> -p <output_dir>
```

**Key flags:**

| Flag | Description | Default |
|------|-------------|---------|
| `-i <file>` | Input IQ file | stdin |
| `-F float` | Input format (`char`, `int`, `float`, `wav`) | `int` |
| `-f <hz>` | Center frequency (Hz) | required |
| `-s <hz>` | Sample rate (Hz) | required |
| `-T <time>` | Start time `YYYY-MM-DDTHH:MM:SS.sss` | system time |
| `-P` | Parse freq/rate/time from input filename automatically | off |
| `-p <dir>` | Output directory | `.` |
| `-c <hz>` | Channel resolution (Hz) | 100 |
| `-t <s>` | Integration time per spectrum (s) | 1 |
| `-n <n>` | Spectra per output file | 60 |
| `-b` | Digitise output to 8-bit bytes | off |
| `-q` | Quiet mode | off |

**Input filename formats** (parsed automatically with `-P` via `rffft_params_from_filename()` in `rffft_internal.c`):
- SatDump: `YYYY-MM-DD_HH-MM-SS_<rate>SPS_<freq>Hz.cf32`
- GQRX: `gqrx_YYYYMMDD_HHMMSS_<freq>_<rate>_fc.raw`
- SDR Console: `DD-MMM-YYYY HHMMSS.mmm <freq>MHz.wav`

**Output filename convention:** `YYYY-MM-DDTHH:MM:SS_NNNNNN.bin`

**Output `.bin` header** (256-byte ASCII):
```
HEADER
UTC_START    YYYY-MM-DDTHH:MM:SS.sss
FREQ         <hz>
BW           <hz>
LENGTH       <s>
NCHAN        <n>
NSUB         <n>
END
```

---

## rffind — Doppler Detection

Detects signals above a noise threshold in spectrogram files and outputs timestamped Doppler measurements.

```bash
rffind -p <path/YYYY-MM-DDTHH:MM:SS> -f <hz> -w <hz> -C <site_id> -o doppler.dat
```

**Key flags:**

| Flag | Description | Default |
|------|-------------|---------|
| `-p <prefix>` | Spectrogram file prefix | required |
| `-f <hz>` | Center frequency to zoom | all |
| `-w <hz>` | Bandwidth to zoom | all |
| `-C <id>` | COSPAR site ID | `$ST_COSPAR` |
| `-S <sigma>` | Detection SNR threshold | 5.0 |
| `-o <file>` | Output `.dat` file | `find.dat` |
| `-s <n>` | Starting subintegration | 0 |
| `-l <n>` | Number of subintegrations | 3600 |
| `-g` | GRAVES bi-static mode | off |

**Output `.dat` format** (one line per detected point):
```
MJD  FREQUENCY_HZ  SNR  SITE_ID
58349.602083 437000123.4 7.3 4353
```

---

## rffit — Interactive TLE Fitting

Fits TLE orbital parameters to Doppler measurements using a Nelder-Mead simplex optimiser. Requires a display (PGPLOT/X11).

```bash
rffit -d doppler.dat -c custom.tle -i <satno> -s <site_id>
```

**Key flags:**

| Flag | Description |
|------|-------------|
| `-d <file>` | Doppler data file (required) |
| `-c <catalog>` | TLE file |
| `-i <satno>` | NORAD number to load from TLE file |
| `-s <site_id>` | Observer COSPAR site ID |
| `-m <hz>` | Frequency offset (Hz) |
| `-g` | GRAVES bi-static mode |

**Interactive keyboard commands:**

| Key | Action |
|-----|--------|
| Mouse drag | Select/deselect data points |
| `1`–`7` | Toggle orbital parameters to fit (see table below) |
| `8` | Toggle rest frequency fitting |
| `f` | Run fit on selected points |
| `c` | Manually edit a TLE parameter |
| `w` | Write improved TLE to file |
| `p` | Toggle model curve overlay |
| `q` | Quit |

**Orbital parameter keys:**

| Key | Parameter | `ia[]` index |
|-----|-----------|-------------|
| `1` | Inclination | 0 |
| `2` | RAAN | 1 |
| `3` | Eccentricity | 2 |
| `4` | Argument of perigee | 3 |
| `5` | Mean anomaly | 4 |
| `6` | Mean motion | 5 |
| `7` | B* drag | 6 |

**Data point flags** (internal `struct point.flag`):
- `0` — deleted, excluded from everything
- `1` — visible but not selected
- `2` — selected, included in fit and RMS

**Output TLE** includes a provenance comment:
```
# TSTART-TEND, N measurements, X.XXX kHz rms
```

---

## TLE Improvement Workflow (satellite not in any catalog)

For a satellite with an approximate TLE not published anywhere:

```bash
# 1. Generate spectrogram from recording
rffft -i 2024-01-15_08-30-00_2048000SPS_437000000Hz.cf32 -F float \
      -P -p ./work

# 2. Extract Doppler points
rffind -p ./work/2024-01-15T08:30:00 -f 437000000 -w 50000 \
       -C 4353 -S 5.0 -o doppler.dat

# 3. Interactive fit
rffit -d doppler.dat -c mysat.tle -i 99999 -s 4353
# Inside rffit: select points → toggle params (e.g. keys 5,6) → f → w
```

**Iterative refinement approach:**
1. Select all clean data points (drag-select in GUI)
2. Toggle mean motion (`6`) only → press `f` → check RMS
3. Add mean anomaly (`5`) → press `f` → check RMS
4. Add inclination/RAAN (`1`, `2`) for longer passes → press `f`
5. Deselect outliers → refit until RMS is acceptable
6. Press `w` to save

---

## Core Source Files

| File | Role |
|------|------|
| `rffft.c` | FFT spectrogram generation |
| `rffft_internal.c` | Input filename parsing (`rffft_params_from_filename`) |
| `rffind.c` | Doppler signal detection |
| `rffit.c` | Interactive TLE fitting (PGPLOT GUI) |
| `rfplot.c` | Interactive spectrogram viewer |
| `rfpng.c` | Batch PNG waterfall generator |
| `sgdp4.c` / `sgdp4h.h` | SGP4/SDP4 orbital propagator |
| `satutl.c` / `satutl.h` | TLE parsing (`read_twoline`), alpha-5 numbering |
| `simplex.c` / `versafit.c` | Nelder-Mead optimiser |
| `rfsites.c` / `rfsites.h` | Observer site lookup |
| `rftles.c` / `rftles.h` | TLE catalog management |
| `rfio.c` / `rfio.h` | Spectrogram binary I/O, `struct spectrogram` |
| `rftime.c` / `rftime.h` | Time conversion (`nfd2mjd`, `mjd2nfd`) |

---

## Automation Roadmap (`rfauto` — planned)

`rffit` has no headless mode. A planned tool `rfauto` will automate the full pipeline non-interactively:

```bash
rfauto -i recording.cf32 -c mysat.tle -n 99999 -s 4353 -P 56 -o improved.tle
```

**Planned flags:**

| Flag | Description | Default |
|------|-------------|---------|
| `-i <cf32>` | Input recording (timestamp parsed from filename) | required |
| `-c <tle>` | Initial TLE file | required |
| `-n <satno>` | NORAD number to load | required |
| `-s <site_id>` | Observer COSPAR site ID | `$ST_COSPAR` |
| `-P <params>` | Parameters to fit, e.g. `"56"` = mean anomaly + mean motion | `"56"` |
| `-o <file>` | Output TLE file | `fit.tle` |
| `-S <snr>` | Minimum SNR for Doppler points | 5.0 |
| `-m <hz>` | Frequency offset | 0 |
| `-p <dir>` | Working directory for temp `.bin`/`.dat` files | auto `/tmp` |
| `-k` | Keep temp files | off |
| `-g` | GRAVES bi-static mode | off |

**Implementation plan:**
- Extract non-GUI fitting functions from `rffit.c` into new `rffit_core.c` / `rffit_core.h`
  (functions: `fit_curve`, `chisq`, `compute_rms`, `print_tle`, `read_data`, `velocity`, `format_tle`, and helpers)
- `rffit.c` updated to `#include "rffit_core.h"`, removing extracted functions (no behaviour change)
- `rfauto.c` calls `rffft` and `rffind` via `system()`, then uses `rffit_core` functions directly
- Does **not** link against PGPLOT/X11 — no display required
- Makefile: `rfauto` links `rfauto.o rffit_core.o sgdp4.o satutl.o deep.o ferror.o simplex.o versafit.o rfsites.o rftles.o rftime.o rffft_internal.o -lm -lgsl -lgslcblas`

---

## Building

```bash
# Linux
make -f Makefile.linux

# macOS
make -f Makefile.osx

# Run tests
make tests
```

Dependencies: `fftw3`, `gsl`, `pgplot`, `cpgplot`, `X11`, `png`, `sox`, `cmocka` (tests only).
