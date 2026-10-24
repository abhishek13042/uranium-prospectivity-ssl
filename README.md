# uranium-prospectivity-ssl

**Self-supervised pretraining + positive-unlabeled learning for uranium prospectivity mapping, built on public airborne geophysics from the Athabasca Basin.**

> **Status: research proposal (October 2026).** No code or results yet. This repo holds the problem framing, literature review and experiment plan; code and results will be added here as they exist. Nothing in this README is a claimed result.

## The problem

Mineral prospectivity mapping asks: given geophysical, geochemical and geological layers over a region, where is a deposit most likely? Two properties make it a poor fit for off-the-shelf supervised learning:

1. **There are no true negatives.** We know where deposits *are*; "no known deposit here" usually just means "not drilled yet". The labels are **positive-unlabeled (PU)**, and sampling random cells as negatives biases the model.
2. **Labels are few and spatially autocorrelated.** A few hundred occurrences over a basin, and neighbouring cells look alike, so random train/test splits leak and inflate scores.

Meanwhile the *unlabeled* signal is plentiful: continuous airborne radiometric and magnetic grids cover the whole region.

## The idea

Use the abundant unlabeled grids to learn representations, and use the scarce labels correctly:

1. **Pretrain** a masked autoencoder (MAE) on multi-channel tiles of public airborne geophysics (radiometric eU, eTh, K and total magnetic intensity plus derivatives).
2. **Fine-tune** a light head on the frozen or partly-unfrozen encoder with a **non-negative PU loss (nnPU)**, using known uranium occurrences as positives and everything else as unlabeled.
3. **Evaluate honestly** with **spatial block cross-validation**, against a random-forest baseline on the same features with random negatives (the common practice).

GFM4MPM (Daruna et al., 2024) showed masked-image pretraining helps prospectivity mapping for Zn-Pb. I could not find published work combining self-supervised pretraining **and** PU learning **for uranium**; this project tests that combination.

## Data (all public)

| Layer | Source | Use |
|---|---|---|
| Airborne gamma-ray spectrometry (K %, eU ppm, eTh ppm) | Natural Resources Canada, Geoscience Data Repository for Geophysical Data | Input channels |
| Airborne total magnetic intensity (+ first vertical derivative, tilt) | Natural Resources Canada, Geoscience Data Repository | Input channels |
| Uranium occurrences and deposits | Saskatchewan Mineral Deposit Index (SMDI), Saskatchewan Geological Survey | Positive labels |
| Optional US analogue: NURE aerial radiometric surveys | USGS | Transfer test |

Data licences and exact survey IDs will be recorded in `data/README.md` when downloaded. Raw data will not be committed.

## Planned layout

```
docs/research_proposal.md   full write-up: background, related work, method, evaluation, risks
references.bib              bibliography
data/                       download scripts + provenance (no raw data)
src/                        tiling, MAE pretraining, nnPU fine-tuning, spatial CV
notebooks/                  exploration and figures
```

## Evaluation plan (summary)

- Spatial block CV (blocks larger than the label autocorrelation range), reported per fold.
- Metrics suited to PU: recall of held-out occurrences in the top-k% area (capture-efficiency / success-rate curve), area under that curve. Precision is not meaningful without true negatives.
- Ablations: random init vs MAE init; random-negative BCE vs nnPU; channel ablations (radiometric only vs magnetic only vs both).

See [docs/research_proposal.md](docs/research_proposal.md) for the full method, risks and timeline.

## Author

Abhishek Kumar Gupta. CS undergraduate working on applied ML and computer vision.

## License

MIT for code and text in this repository. Third-party data keeps its own licence.
