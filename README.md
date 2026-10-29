# Multiplex tonsil cell-instance and phenotype task

## One-paragraph pitch

A core-facility analyst must transfer manual cell-boundary interpretation from five annotated human-tonsil multiplex fields to five held-out fields, resolve nuclear/whole-cell instances under marker and signal variation, pair each cell with its nucleus, and quantify seven marker channels. The hard judgment is deciding which nuclear and cytoplasmic evidence defines a complete cell instance rather than a foreground region. A wrong answer is a threshold mask, copied visible annotation, or plausible marker average with incorrect cell boundaries.

## Scientific data

This candidate uses real public data rather than a synthetic simulator:

- **Source:** *Annotated Multiplex Immunofluorescence Images of Human Tonsil Tissue*
- **DOI:** [10.5281/zenodo.22107836](https://doi.org/10.5281/zenodo.22107836)
- **License:** CC BY 4.0
- **Source creators:** Olivia Aguado, María Sangüesa, Carlos E. de Andrea, and Mikel Ariz
- **Panel:** CD21, CD23, CD20, CD4, CK, CD8, DAPI
- **Data format:** seven-page float32 TIFF stacks, 100 × 100 pixels per field

Five annotated fields are exposed as examples. Five fields are held out; their manual masks remain only in the sealed verifier. Agent-visible identifiers are opaque. Source attribution, original filenames, mapping, and checksums are in `authoring/provenance/source_manifest.json`.

The project-specific AfterQuery instructions require the `afterquery/` task-name prefix, so the manifest uses `afterquery/multiplex-phenotyping` rather than the generic template prefix.

## Difficulty

The manifest uses a three-hour focused expert estimate rather than the authoring time spent building the bundle; revise it if the platform's difficulty probe shows a materially different solve time.

The task requires transferring an instance-level biological interpretation across fields with different signal distributions. The agent must use nuclear and cytoplasmic/marker evidence together, resolve touching cells, preserve instance identity, associate cells with nuclei, and quantify marker profiles. A foreground mask, copied visible mask, hard-coded field average, or unverified table will fail.

The source annotations are public. Opaque field staging and held-out fields reduce accidental answer lookup, but they do not prove that the task is immune to online retrieval; the platform originality and anti-cheat gates remain mandatory.

The task must still pass the platform's empirical eight-attempt difficulty probe; local checks do not establish that result.

## Reference solution

The oracle learns pixel-level nuclear and cell evidence from the visible annotations using normalized seven-channel intensities and local multiscale features. It derives nuclear instances with marker-seeded watershed, uses nuclear instances as seeds for whole-cell watershed, and computes each cell's centroid, area, nuclear association, and raw marker means. It is one valid route; the verifier grades scientific outputs rather than requiring this implementation.

`authoring/independent_solver.py` is a separate implementation with a different forest size, depth, and random seeds. It scores 1 on the same held-out set and is not mounted into the agent environment.

## Verification

All gates are binary only at the final reward level; the pytest report retains diagnostics.

| Gate | Acceptance rule | Reference worst case | Independent worst case | Nearest wrong route |
|---|---|---:|---:|---|
| Schema | All 10 fields, both label arrays, required CSV columns, finite values, positive counts | pass | pass | no output / malformed output |
| Nuclear instances | Recall ≥ 0.70, precision ≥ 0.65, mean IoU ≥ 0.45 on every field | 0.742 / 0.708 / 0.502 | 0.773 / 0.680 / 0.508 | copied masks: 0.048 / 0.030 / 0.093 |
| Whole-cell instances | Recall ≥ 0.72, precision ≥ 0.70, mean IoU ≥ 0.45 on every field | 0.773 / 0.750 / 0.516 | 0.848 / 0.741 / 0.538 | no-cell-model route: 0.667 / 0.609 / 0.391 |
| Marker self-consistency | Reported means must match means recomputed from the predicted mask: `abs(error) ≤ 0.02 + 0.04 × abs(expected)` | pass | pass | zeroed/copy table fails |
| Held-out profile agreement | Matched-cell marker error median ≤ 0.35 and 75th percentile ≤ 0.70 | 0.056 / 0.120 | 0.052 / 0.112 | copied masks: 0.082 / 0.159 for matched cells, but fails instance gates |

The thresholds are above the measured no-cell and copied-mask routes while leaving both valid implementations margin. Calibration used five visible and five held-out fields from one fixed public source; there is no synthetic reseed axis. The verifier recomputes marker statistics from raw source images rather than trusting the submitted table.

## Ablation ladder

Measured minimums are recorded in `authoring/evidence/ablation_runs.md`.

- **Full reference:** all gates pass; held-out profile median/75th `0.056/0.120`.
- **Independent parameterization:** all gates pass; `0.052/0.112`.
- **No cell model, DAPI/max-marker route:** fails nuclear recall and whole-cell gates (`0.652` nuclear recall; `0.667` cell recall; `0.609` cell precision).
- **Copy visible masks:** fails held-out instance gates (`0.048` nuclear recall; `0.196` cell recall).
- **No-op:** fails schema immediately and receives reward `0`.
