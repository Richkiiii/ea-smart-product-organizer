# REA Smart Product Organizer

Offline Windows 11 application that finds images of the same product among thousands of
photos, groups them, and selects the best image per product. Architecture and roadmap:
see the Phase 1 blueprint document.

## Status

| Phase | Scope | State |
| --- | --- | --- |
| 1 | Architecture blueprint | Approved |
| 2 | Engine foundation: scan, validate, hash, embed, candidate search, database, CLI, evaluation, CI | Delivered (0.2.0) |
| **3** | **Identity verification (SIFT geometry, OCR, size/brand/variant vetoes), confidence, grouping** | **Engine done (0.3.0.dev0); needs real OCR models + real photos to close the gate** |
| 4 | Quality engine and best-image selection | Planned |
| 5 | Desktop interface (PySide6, 11 screens) | Planned |
| 6 | Output, quarantine, undo, reports | Planned |
| 7 | 50k-100k scale hardening | Planned |
| 8 | Signed installer and release | Planned |

## What Phase 2 contains

```
src/rea_spo/
  app/        paths (%LOCALAPPDATA%\REA\SmartProductOrganizer), settings, rotating logs
  core/       project management, pipeline orchestrator (stages 1-4 + 6), pause/resume
  files/      read-only discovery, long paths, drive-type detection (SSD/HDD/USB)
  vision/     decode + validation (EXIF, ICC->sRGB, alpha, JPEG quality), segmentation,
              pHash/dHash + multi-index hash table, colour/silhouette, RootSIFT + VLAD
  ai/         ONNX Runtime layer (CUDA / DirectML / CPU with self-test fallback),
              model registry with SHA-256 checks, SigLIP 2 embedder, classic fallback
  matching/   candidate generation (hash + embedding + VLAD), FAISS or NumPy kNN
  storage/    SQLite schema v1 (blueprint schema), migrations, single batched writer
  workers/    pause/resume/cancel, throttled progress
  devtools/   synthetic labelled dataset, evaluation harness
  cli.py      command-line interface
tools/export_models/export_embedder.py   SigLIP 2 / DINOv2 -> ONNX (+INT8), parity-checked
tools/eval/check_gate.py                 Phase 2 gate check for CI
packaging/                               PyInstaller spec, Inno Setup script, build_windows.ps1
.github/workflows/windows.yml            Windows CI: tests, model export + recall, EXE build
tests/                                   27 automated tests (unit, integration, safety)
```

Stages 5 (OCR) and 7-12 are Phase 3+ and intentionally not present yet.

## Run it (developer machine)

```powershell
pip install -e ".[fast,dev]"
rea-spo project create "Store photos" --source "E:\Product photos"
rea-spo analyse "Store photos"          # Ctrl+C pauses safely; run again to resume
rea-spo status "Store photos"
rea-spo candidates "Store photos" --csv pairs.csv
```

Evaluate against labels (CSV with `rel_path,product_id`):

```powershell
rea-spo eval "Store photos" --labels labels.csv --sweep
rea-spo gen-dataset synth --families 30 --singletons 80     # synthetic test set
rea-spo bench synth --labels synth/labels.csv
```

Install the production embedder (needs internet once, on the build machine):

```powershell
pip install -e ".[export]"
python tools/export_models/export_embedder.py --model siglip2 --int8
```

Without the model the engine runs on the classic (non-neural) fallback embedder and says so.

## Safety guarantees implemented now

* Source folders are only listed and opened with `rb`; a test runs the whole pipeline on a
  read-only copy and verifies every byte and timestamp is unchanged.
* No sidecar files are written next to originals; thumbnails, keypoints and the database
  live in the project folder under app data.
* Corrupt, truncated and unsupported files are recorded and skipped, never fatal.

## Phase 2 measurements (this build environment: 2 CPU threads, Linux, no GPU, no SigLIP 2)

| Dataset | Images | Time | Speed | Candidate recall | Candidates / image |
| --- | --- | --- | --- | --- | --- |
| Synthetic, 30 families + 80 singles | 680 | 35 s | 18 img/s | 93.4% | 34 |
| Synthetic, 120 families + 400 singles | 2,968 | 168 s | 16.5 img/s | 91.4% | 38 |

These use the **classic fallback embedder**. The Phase 2 gate (candidate recall >= 99.5%)
is defined for the production SigLIP 2 model and must be measured by the `eval` job in
Windows CI and on real product photos. The synthetic set is deliberately hard: every
product has near-identical size and flavour siblings.

## Phase 3 (0.3.0.dev0): identity verification and grouping

New modules: `ocr/attributes.py` (size, pack count, %, variant, brand parsing + vetoes, EN/HU),
`ocr/engine.py` (PP-OCRv5 via RapidOCR), `core/ocr_stage.py` (lazy OCR), `matching/verify.py`
(RootSIFT + MAGSAC geometry, colour, proportions, name-text similarity), `matching/scoring.py`
(fitted, sign-constrained logistic model + isotonic calibration, decision rules),
`matching/grouping.py` (conflict-safe grouping), `core/verification.py`, `devtools/phase3.py`.

```powershell
rea-spo verify "Store photos"          # OCR (if installed) + pair decisions + groups
rea-spo groups "Store photos" --csv groups.csv
rea-spo train-matcher "Store photos" --labels labels.csv   # refit the confidence model on real labels
rea-spo eval-groups "Store photos" --labels labels.csv
```

Out-of-sample result (model fitted on the 30-family set, tested on the separate 2,968-image set,
simulated OCR with ~30% of photos lacking a readable size):

| Metric | Result | Target |
| --- | --- | --- |
| Automatic merges that were correct | 100% (1,287 pairs) | >= 99.5% |
| Size or flavour wrongly merged | 0 | 0 |
| Groups mixing two products | 0 of 1,993 | 0 |
| True pairs merged automatically or sent to review | 97.5% | >= 98% |
| True pairs merged automatically | 33% | >= 90% |
| Images needing a human decision | 58% | <= 10% of groups |

Safety targets are met. Automation targets are not: most reviews come from photos whose size
cannot be read while other sizes of the same product exist, which the rules deliberately send
to a person. Real photos and real OCR are needed to see the true review load.
