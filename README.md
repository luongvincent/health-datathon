# Predicting Pathological Complete Response in Breast Cancer from Clinical and Histopathology Features

## Overview

Response to neoadjuvant chemotherapy (NAC) varies widely across breast cancer patients. This project predicts pathological complete response (pCR, RCB 0) versus residual disease (RCB I to III) from pre-treatment information, and tests whether features from a frozen pathology foundation model (Phikon) add predictive value over clinical variables alone.

Model families, each benchmarked against a clinical-only baseline:

1. **Clinical only:** four pre-treatment variables (subtype, regimen, grade, clinical tumour size)
2. **Image only:** Phikon embeddings of H&E tiles, mean-pooled per patient and reduced by PCA
3. **Multimodal:** clinical and image features combined (also subtype plus image)

The accompanying paper reports the full methods and results.

## Data Information

**High-Resolution Digital Pathology Imaging of Breast Cancer** v1.0.0, Health Data Nexus (DOI: 10.57764/pm3v-b131).

- 157 patients from a single Canadian institution, treated with NAC between 2013 and 2018
- H&E whole-slide biopsy images (.tif), 40x, ~60,000 x 18,000 px at 0.2 um/px
- One folder per patient (NAC-1 to NAC-157) containing a .tif and a .docx clinical summary
- Clinical variables in `Master_Data_Sheet.csv` (157 rows)
- Label: pCR (RCB 0) vs non-pCR (RCB I to III)
- Subtype is parsed from the .tif file name (e.g. `Luminal_A-1`)

**Cohort used.** 106 of 157 patients had a usable RCB class; 51 had no usable RCB value and were excluded. Because 43 of those 51 are recorded as clinical complete responders, the excluded group probably contains many pCR cases, so the analysed pCR rate is likely an underestimate.

Class balance (106 analysed): pCR n = 30 (28.3%), non-pCR n = 76 (71.7%)
Development set (89 patients, used for all reported results): RCB 0 n = 25, RCB I n = 17, RCB II n = 34, RCB III n = 13. A further 17 patients were held out and not used for reported results.
Subtype counts (106 analysed): Luminal A 36, Luminal B 21, HER2-enriched 10, triple-negative 39

**Data access:** The data is credentialed and covered by the T-CAIREM Data Use Agreement. No patient-level data, slide images, tile images, split files, patient identifiers or derived patient-level features are included in this repository. Only code and aggregate results are shared.

## Methodology

**Validation**
- All splits at the patient level, so tiles from one patient never appear in both train and test
- Repeated stratified 5-fold cross-validation, 10 repeats (seed 0), on the 89 development patients, stratified on the four-class RCB label
- The pCR score is the out-of-fold probability of RCB 0
- AUC is the mean of per-repeat AUCs; 95% CIs are percentile intervals from a patient-level bootstrap (500 resamples); gains over a baseline use paired bootstrap differences

**Features**
- Clinical: molecular subtype (4 classes), chemotherapy regimen family (ddAC-T family vs FEC-D or other), Nottingham grade, clinical tumour size (log-transformed); 8 encoded inputs
- Image: tissue masked by Otsu thresholding of HSV saturation (threshold capped at 0.15), 112 um tiles (224 px at 0.5 um/px), tiles kept at 50% or more tissue overlap, tile quality filter, up to 500 tiles sampled per patient, embedded with **Phikon** (frozen ViT-Base, iBOT pre-trained on public TCGA tiles), mean-pooled to 768 dimensions per patient, then standardised and reduced to 5 principal components inside each training fold
- Multimodal: concatenated patient-level clinical and image features

**Models**
- Multinomial logistic regression (L2, C = 0.5) on frozen features
- No network trained or fine-tuned

**Leakage controls**
- Only information available at diagnosis is used. Post-treatment variables (surgery type, adjuvant radiation, adjuvant systemic therapy, survival outcomes) are excluded
- Imputation, scaling, one-hot encoding and PCA are fit inside training folds only
- Known caveat: the tile-quality percentile cutoffs were computed over the training patients of the 70/15/15 split, not recomputed inside each CV fold (no labels used)

**Metrics**
- Primary: ROC-AUC
- The model outputs a probability and no decision threshold was set, so recall, specificity, F1 and a confusion matrix are not reported
- Class balance (28% pCR) checked before choosing metrics

## Results

Cross-validated on the 89 development patients (10 x 5-fold). Values in brackets are 95% patient-level bootstrap CIs.

| Model | ROC-AUC (95% CI) |
|---|---|
| Clinical only (4 variables) | 0.856 (0.77 to 0.93) |
| Image only (Phikon, 5 PCs) | 0.704 (0.58 to 0.82) |
| Multimodal: clinical + image | 0.862 (0.77 to 0.93) |
| Subtype only | 0.858 (0.79 to 0.92) |
| Subtype + image | 0.882 (0.80 to 0.95) |


Image features alone carry response-related signal (AUC 0.704), but the gain from adding them to clinical variables is small and not distinguishable from zero. Paired AUC gain of image features: +0.006 (95% CI -0.031 to 0.045) over the full clinical set and +0.024 (-0.027 to 0.077) over subtype alone. Molecular subtype accounts for most of the discrimination.

## Limitations

- Single institution, so generalization to other sites, scanners, and staining protocols is unknown
- Small sample (106 with a recorded RCB, 89 used for development, 25 pCR events), which gives wide confidence intervals and a risk of optimistic estimates
- 51 patients excluded for no usable RCB value, many probably pCR, which may bias the cohort
- No external validation cohort
- Image features may capture scanner or stain artifacts rather than tumor biology
- Proof of concept only, not a clinical tool

## Acknowledgements

- Dataset: Health Data Nexus, High-Resolution Digital Pathology Imaging of Breast Cancer v1.0.0 (DOI: 10.57764/pm3v-b131)
- T-CAIREM Health Datathon
- Use of the data is governed by the T-CAIREM Data Use Agreement and the dataset LICENSE