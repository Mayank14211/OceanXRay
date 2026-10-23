# OceanXRay

OceanXRay (GUNI SSRIP 2026) uses AI to detect underwater activity from Sentinel-1 SAR imagery, with ERA5 and SST as context. This repository holds the raw Kaggle notebooks, with their saved outputs, for the internal-wave work: a detector (Faster R-CNN v2 trained on S1-IW-2023), inference on 47 real June-2021 scenes, a Tier 1 FFT wavelength/direction estimator with a manual review, and a Streamlit dashboard.

**Status**
- **Delivered:** internal-wave detector and benchmark; inference on 47 real scenes; Tier 1 FFT estimator; manual precision review (on the older detector); Streamlit dashboard.
- **Preliminary:** SAR/geo encoders and multimodal fusion (not in these notebooks).
- **Planned:** Tier 2 (back-tracking a wave to its source); measuring the v2 detector's precision at scale.

> These are raw Kaggle exports, shared to document results, not as a runnable package. Kaggle paths and dataset IDs will not work outside Kaggle, and the Cloudflare tunnel link printed in `improve-landing.ipynb` has expired. Read the saved outputs; you do not need to run anything.

## Project overview

**The idea.** Internal waves are waves inside the ocean, travelling between water layers of different density. They are not visible from above, but they leave a footprint on the surface: bands of rough and smooth water that radar can see. Sentinel-1 SAR works at night and through cloud, so detecting these bands is an indirect way to map what is happening below the surface. The study area is Indian coastal waters in June 2021.

**Track 1: internal-wave detector (delivered, in these notebooks)**
1. **Detect.** A Faster R-CNN v2 detector (1024 px input) draws a box around each wave packet in 25 km SAR tiles. It is trained on the human-labelled S1-IW-2023 dataset. The improved training recipe adds 8-way flips and rotations, brightness and contrast jitter, a warm-up plus cosine learning-rate schedule, and best-epoch selection on a validation split.
2. **Geolocate.** Each scene's ground control points are fitted with one affine transform, which turns box positions into latitude and longitude.
3. **Filter.** Detections centred on land or within 1 km of the coast are removed.
4. **Measure (Tier 1).** An FFT of each detection crop estimates wavelength and propagation direction.
5. **Explore.** A Streamlit dashboard: upload a tile and see the detected packets on the image and on a live map. It also shows sea-floor depth and distance to the shelf edge (GEBCO), nearby place names (Natural Earth) and scene-time weather (Open-Meteo / ERA5).

**Track 2: multimodal research (preliminary, not in these notebooks)**
- A self-supervised SAR encoder that learns to predict image gradients rather than pixels, so it does not blur away radar speckle texture.
- ERA5 weather and SST aligned to each SAR patch's real position and time.
- A fusion model that combines all three. So far it has been checked only against automatic (CFAR proxy) labels, not human labels.

**What it is not.** OceanXRay does not do real-time submarine tracking or real-time alerts.

**Next steps.** Hand-check a random sample of v2 detections to measure precision; implement Tier 2 (back-tracking a wave to its source); extend the pipeline beyond June 2021.

## Where to find each result

Cell numbers count from 1 (top of the notebook).

| Notebook | Cell | What the output shows |
|---|---|---|
| [`improving-iw-detector.ipynb`](notebooks/improving-iw-detector.ipynb) | 1–2 | Setup: install `pycocotools`, copy earlier training outputs. |
| | 3 | An earlier version that stopped with an error (a bug). It was fixed in the later cells and is left as-is. |
| | 4 | An intermediate version of the benchmark. |
| | **5** | **Final benchmark:** S1-IW-2023 test split (74 images, 108 waves), 3 seeds, 95% bootstrap ranges, individual runs and paired bootstrap differences. |
| [`improve-landing.ipynb`](notebooks/improve-landing.ipynb) | **1** | **47 scene geolocation transforms** saved to `scene_geo_table.json`. |
| | 2 | Path setup (no output). |
| | **3** | **Inference on real scenes:** 4,524 tiles, 4,316 raw detections (score ≥ 0.5); 1,002 centred on land (23.2%); 189 within 1 km of the coast (4.4%); **3,125 kept in 47 scenes**, mean confidence 0.743; lat 5.31 to 22.44, lon 69.66 to 92.52; 2,185 tiles with kept detections. |
| | 4–9 | Dashboard set-up and launch: install Streamlit, write the context, bathymetry and dashboard modules, start a Cloudflare tunnel. The outputs are install and launch logs. |
| [`backtracking-internal-waves.ipynb`](notebooks/backtracking-internal-waves.ipynb) | 1–3 | Earlier versions of the Tier 1 estimator, kept to show how it was debugged. Their outputs are superseded. |
| | **4** | **V4 (final version):** the manual precision review (see below). |

**All outputs in `backtracking-internal-waves.ipynb` come from the 798 detections of the older detector, not the v2 detector.**

## Detector benchmark

S1-IW-2023 test split (74 images, 108 waves). Each value is the mean over 3 seeds, without test-time augmentation, from Cell 5 of `improving-iw-detector.ipynb`. Cell 5 also prints the test-time-augmentation rows and per-seed runs.

| Model | mAP@0.5 | mAP@0.5:0.95 |
|---|---|---|
| Faster R-CNN v2, improved training | 92.27 | 64.38 |
| Faster R-CNN, improved training | 91.08 | 61.26 |
| SSD300 (VGG16) | 89.84 | 57.46 |
| FCOS (R50-FPN) | 89.48 | 64.04 |
| Faster R-CNN, original training | 85.02 | 57.75 |
| RetinaNet (R50-FPN v2) | 84.38 | 56.41 |

Notes:
- The test set is small, so 95% bootstrap ranges for a single model are about ±4–5 points. Cell 5 prints each range.
- Faster R-CNN v2 and FCOS cannot be told apart on mAP@0.5:0.95 (paired difference +0.32, 95% range −2.91 to +3.56).
- The dashboard checkpoint is the seed-2 run (`improved_v2_s2`, 91.52 mAP@0.5), not the 3-seed mean.
- Cell 5 also lists published paper numbers. Those use a different, larger evaluation set (S1-IW-2023 plus 1,133 added images, re-split), so they are context only and not comparable.

## Manual precision review

From Cell 4 of `backtracking-internal-waves.ipynb`. This review used the **older** detector.

- 48 detections were sampled at random from our June 2021 scenes, after the land and coast filters. Each was labelled: wave packet 14, coast/land 3, single line (ship wake, slick or lone front) 8, texture/swell 13, unsure 10.
- Of the 38 decided (excluding the 10 unsure), 14 were real internal-wave packets: **37% (95% CI 23–53%)**.
- Tier 1 direction was judged correct in **9 of 9** reviewed packets (95% range 70–100%).
- The precision of the v2 detector at scale is **unmeasured**.

## Limitations

- **Coverage:** the scenes cover the Bay of Bengal and Arabian Sea coasts of India and Sri Lanka. There are no kept detections east of about 92.5°E.
- **Not a hotspot map:** the detector fires in roughly half of all tiles, so the detection map mirrors where the scenes are.
- **Likely false alarms:** about 50% of kept detections are over water deeper than 1,000 m, which likely includes false alarms. Wave packets are expected mostly near the continental shelf edge.
- **Positions are approximate:** each scene is geolocated with one affine fit to its ground control points, accurate to about 1 km.
- **Direction:** Tier 1 direction has a 180° ambiguity. Tier 2 (back-tracking to the source) is not implemented.
- **Dashboard weather:** comes from Open-Meteo (ERA5 reanalysis). Marine wave and SST fields show n/a for 2021.

## Data and credits

- **S1-IW-2023** internal-wave dataset, Zenodo: https://doi.org/10.5281/zenodo.11090328 (dataset; no explicit citation given)
- **Sentinel-1** SAR imagery: Copernicus
- **ERA5** reanalysis: ECMWF
- **SST** (sea-surface temperature)
- **GEBCO** sea-floor depth, used by the dashboard
- **Natural Earth** place names, used by the dashboard

**Team:** Jinal H. Bhavsar, Mayank Gupta, Sriram Rajesh Manwani
**Mentors:** Dr. Meghna Patel, Pooja Pancholi

## Saved result files

These are the files the notebooks wrote, downloaded from Kaggle and committed unchanged. Folder names are as Kaggle exported them. Each `.virtual_documents/__notebook_source__.ipynb` is a source-only copy that Kaggle generated, not a notebook with outputs.

| Folder | Produced by | Contents |
|---|---|---|
| [`Outputs/results (1)/`](<Outputs/results (1)>) | `improve-landing.ipynb` | `scene_geo_table.json` (47 scene geolocation transforms), `real_internal_wave_detections_v2.parquet` (the 3,125 kept v2 detections), `iw_demo_candidate_filenames.txt` (tiles with kept detections, highest confidence first) |
| [`Outputs/results (2)/outputs/iw_improve/`](<Outputs/results (2)/outputs/iw_improve>) | `improving-iw-detector.ipynb` | `report.txt` (the same benchmark report as Cell 5), `results.csv` (per-run metrics and training settings), `history_*.csv` (per-epoch training history), `preds_*.parquet` (test-set predictions per run, with and without TTA), `comparison_chart.png`, `presentation_chart.png` |
| [`Outputs/results (3)/`](<Outputs/results (3)>) | `backtracking-internal-waves.ipynb` | `tier1_fft_results.parquet` (Tier 1 estimates), `tier1_inspect.png` (12 inspection crops), `review_sheet_1-4.png` (the 48 crops shown in the manual review), `review_labels.csv` (the review template; its label columns are blank, and the filled-in labels used in Cell 4 are not in this folder). Older detector. |
| [`Outputs/regional_map.png`](Outputs/regional_map.png) | — | Map of the 3,125 kept v2 detections over GEBCO depth. It shows where the scenes are, not hotspots. |
| [`Evaluation/`](Evaluation) | dashboard | Dashboard screenshots (`testing(2–4).jpeg`), an annotated example tile (`tile_snapshot.jpg`) and the canyon-head case-study map (`testing(1).jpeg`). The 32 canyon-head detections in that map have **not** been checked by eye; treat them as candidates, not confirmed waves. |

## Not in this repository

- **Model weights:** the 18 `.pth` checkpoints (95–173 MB each, about 2.5 GB in total) exceed GitHub's 100 MB file limit.
- **Data:** SAR tiles, `.SAFE` folders and the filled-in manual review labels.
- **Other material:** the presentation deck, the encoder and fusion code, and Tier 2.
