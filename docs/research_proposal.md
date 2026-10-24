# Self-Supervised Pretraining with Positive-Unlabeled Fine-Tuning for Uranium Prospectivity Mapping in the Athabasca Basin

**Abhishek Kumar Gupta**. Research proposal, October 2026. *No experiments have been run yet; this document states the hypothesis and the plan to test it.*

---

## Abstract

Data-driven mineral prospectivity mapping (MPM) is limited by two properties of exploration data: deposit labels are scarce and positive-unlabeled (absence of a known deposit is not evidence of absence), and both labels and features are spatially autocorrelated. Unlabeled geophysical coverage, by contrast, is abundant. We propose to (i) pretrain a masked autoencoder on public airborne radiometric and magnetic grids covering the Athabasca Basin, Saskatchewan, (ii) fine-tune it with a non-negative positive-unlabeled (nnPU) objective using known uranium occurrences as positives, and (iii) evaluate with spatial block cross-validation using capture-efficiency metrics appropriate to PU data. We compare against a random-forest baseline trained with randomly sampled negatives, the most common practice in the MPM literature. The central question is whether self-supervised representations and a PU-correct objective each improve recovery of held-out occurrences under spatially honest evaluation, and whether the gains are additive.

## 1. Background

**Unconformity-associated uranium.** The Athabasca Basin hosts the world's highest-grade uranium deposits, which sit at or near the unconformity between Proterozoic sandstone and the underlying basement, typically along graphitic, reactivated basement faults (Jefferson et al., 2007). Exploration relies on airborne and ground geophysics (magnetics and EM to trace basement structure and conductors; gamma-ray spectrometry for surface signatures), lithogeochemistry and drilling. Deposits are often blind beneath hundreds of metres of sandstone, so surface signals are indirect and weak, which makes the problem a good test of how much a learned representation can extract from coarse regional data.

**Mineral prospectivity mapping.** MPM combines evidence layers into a map of deposit likelihood. Knowledge-driven methods (fuzzy logic, weights-of-evidence) encode expert mineral-system models; data-driven methods (random forests, gradient boosting, neural networks) learn from known occurrences. Data-driven MPM at continental scale has been demonstrated for sediment-hosted Zn-Pb (Lawley et al., 2022).

## 2. Problem statement

Let each grid cell *x* carry a multi-channel geophysical feature vector or patch. Known occurrences give a positive set P. All remaining cells form an unlabeled set U, which contains undiscovered deposits. Goal: a score *f(x)* such that ranking cells by *f* recovers held-out occurrences within as small an area as possible.

Two pitfalls in common practice motivate this work:

1. **Random negatives.** Treating a random sample of U as negatives and training binary cross-entropy injects label noise exactly where undiscovered deposits are, and can shrink and fragment predicted prospective areas (see the 2025 *Natural Resources Research* PU-annotation study in the references).
2. **Random splits.** Spatial autocorrelation means random train/test splits place near-duplicate cells on both sides, inflating metrics (Roberts et al., 2017).

## 3. Related work

- **Self-supervised geospatial pretraining.** Masked autoencoders (He et al., 2022) and their remote-sensing variants (SatMAE, Cong et al., 2022; Prithvi, Jakubik et al., 2023) learn transferable representations from unlabeled imagery. **GFM4MPM** (Daruna et al., 2024) applied masked-image pretraining to MPM for Zn-Pb and reported gains over training from scratch, with explainability analysis. This is the closest prior work; we extend the idea to uranium, to airborne radiometric and magnetic inputs, and to a PU objective.
- **PU learning.** Elkan and Noto (2008) formalised learning from positive and unlabeled data under the selected-completely-at-random assumption; Kiryo et al. (2017) introduced the non-negative risk estimator (nnPU) that stays stable with flexible models. In MPM, PU methods have been combined with random forests (e.g., critical-mineral mapping on the Gawler Craton) and used to construct better negative sets (2025 *Natural Resources Research*).
- **ML for Athabasca uranium.** Published ML work is mostly on drill-hole and lithogeochemical data, for example multivariate analysis of sandstone geochemistry above blind deposits (Chen, Hattori and Grunsky, 2018) and an open ML pipeline for drill-hole assays (Serdoun et al., IAEA). Several exploration companies report ML targeting, but without peer-reviewed methods.

**Gap.** I found no published study combining self-supervised pretraining and PU learning for uranium prospectivity. This is a literature-search observation, not a guarantee; the first step of the project is a more systematic search.

## 4. Data

| Layer | Source | Notes |
|---|---|---|
| Gamma-ray spectrometry: K (%), eU (ppm), eTh (ppm), and ratios eU/eTh, eU/K | NRCan Geoscience Data Repository for Geophysical Data | Regional airborne grids; resolution varies by survey and will be resampled to a common grid |
| Total magnetic intensity, residual field; derived first vertical derivative and tilt angle | NRCan Geoscience Data Repository | Basement structure proxy |
| Uranium occurrences, showings and deposits | Saskatchewan Mineral Deposit Index (SMDI) | Positives; filter by commodity and status, dedupe within a cell |
| (Optional) NURE aerial radiometric surveys | USGS | Cross-region transfer test |

Preprocessing: reproject to a common equal-area CRS, resample to a fixed cell size (initially ~500 m, tested at 250 m and 1 km), per-channel robust normalisation, and tiling into fixed-size patches (e.g., 32×32 cells) with overlap for pretraining. All provenance (survey IDs, download dates, licences) will be recorded alongside the download scripts.

## 5. Method

**5.1 Pretraining.** A ViT-Small MAE over multi-channel patches, with per-channel patch embedding so channels can be dropped at test time. High masking ratio (~75%), reconstruction loss on masked patches only. Pretrain on the entire basin plus a buffer, which is legitimate because no labels are used, but report a variant that pretrains only on training folds to rule out transductive leakage.

**5.2 Fine-tuning.** Pool the encoder output for the patch centred on each cell and attach an MLP head. Train with nnPU (Kiryo et al., 2017). The class prior π (fraction of truly prospective cells) is unknown; treat it as a hyperparameter swept over a plausible range, chosen by validation-fold capture efficiency, and report sensitivity.

**5.3 Baselines.**
- B1: random forest on per-cell features, random negatives, BCE (common practice).
- B2: random forest with PU bagging.
- B3: same network, random init, nnPU.
- B4: same network, MAE init, random-negative BCE.
- Proposed: MAE init + nnPU.

This 2×2 design (init × objective) isolates the two contributions.

## 6. Evaluation

- **Spatial block CV**: partition the basin into blocks larger than the empirical autocorrelation range of the labels (estimated by variogram), with k folds by block. Buffer zones between train and test blocks.
- **Metrics**: capture-efficiency (success-rate) curve: fraction of held-out occurrences captured vs fraction of area ranked most prospective; area under that curve; recall at top 5% and 10% of area. Accuracy and precision are not reported as headline metrics because U is not a negative set.
- **Robustness**: per-fold variance, sensitivity to π, cell size, and channel ablations.
- **Interpretability**: integrated gradients or occlusion per channel, checked against the known mineral-system model (structure and conductors in magnetics vs surface radiometric signal).

## 7. Risks and limitations

- **Signal may be weak at regional scale.** Many Athabasca deposits are deeply buried; surface radiometrics may carry little direct signal. A negative result, where SSL and PU do not beat the baseline under spatial CV, would still be informative and will be reported as such.
- **Label bias.** Occurrences cluster where exploration happened, violating the SCAR assumption behind nnPU. Mitigation: include exploration-density covariates as a diagnostic, and test a bias-aware PU variant if time allows.
- **Missing key layers.** Airborne EM and drill-hole data are central to real targeting but are not uniformly public; this study uses what is openly available and says so.
- **Small positive set.** High variance across folds; report distributions, not single numbers.

## 8. Timeline (indicative)

| Weeks | Milestone |
|---|---|
| 1 | Systematic literature search; data download scripts; provenance log |
| 2 | Preprocessing, tiling, label processing; RF baselines B1/B2 with spatial CV |
| 3–4 | MAE pretraining; nnPU fine-tuning; 2×2 ablation |
| 5 | Robustness, interpretability, write-up of results (positive or negative) |

## 9. Expected contribution

A reproducible, fully public-data benchmark for uranium prospectivity under spatially honest, PU-appropriate evaluation, and evidence on whether self-supervised pretraining and PU objectives help, separately and together, for this commodity and data type.

## References

- Chen, S., Hattori, K., and Grunsky, E. C. (2018). Identification of sandstones above blind uranium deposits using multivariate statistical assessment of compositional data, Athabasca Basin, Canada. *Journal of Geochemical Exploration*.
- Cong, Y., et al. (2022). SatMAE: Pre-training Transformers for Temporal and Multi-Spectral Satellite Imagery. *NeurIPS*.
- Daruna, A., et al. (2024). GFM4MPM: Towards Geospatial Foundation Models for Mineral Prospectivity Mapping. *ACM SIGSPATIAL*. https://arxiv.org/abs/2406.12756
- Elkan, C., and Noto, K. (2008). Learning classifiers from only positive and unlabeled data. *KDD*.
- He, K., et al. (2022). Masked Autoencoders Are Scalable Vision Learners. *CVPR*.
- Jakubik, J., et al. (2023). Foundation Models for Generalist Geospatial Artificial Intelligence (Prithvi). arXiv:2310.18660.
- Jefferson, C. W., et al. (2007). Unconformity-associated uranium deposits of the Athabasca Basin, Saskatchewan and Alberta. *Geological Survey of Canada Bulletin 588*.
- Kiryo, R., Niu, G., du Plessis, M. C., and Sugiyama, M. (2017). Positive-Unlabeled Learning with Non-Negative Risk Estimator. *NeurIPS*.
- Lawley, C. J. M., et al. (2022). Data-driven prospectivity modelling of sediment-hosted Zn-Pb mineral systems and their critical raw materials. *Ore Geology Reviews*.
- Roberts, D. R., et al. (2017). Cross-validation strategies for data with temporal, spatial, hierarchical, or phylogenetic structure. *Ecography*.
- Serdoun, et al. Efficient machine-learning pipeline for drill-hole data, Athabasca Basin. IAEA URAM contribution. https://members.loria.fr/FSur/articles/serdoun24efficient.pdf
- PU-based recursive annotation of negative samples for mineral prospectivity mapping (2025). *Natural Resources Research*. https://link.springer.com/article/10.1007/s11053-025-10510-0
- Multimodal learning framework for 3D mineral prospectivity modeling with jointly learned structure-fluid relationships (2023). arXiv:2309.02911.
