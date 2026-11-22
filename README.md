# Asthma-COPD Differentiation: Quantifying Phenotype-Definition Leakage

**How much of reported asthma-versus-COPD classification performance is just the labeling rule?**

> **Accepted at NSysS 2026**, the 13th International Conference on Next Generation Computing, Communication, Systems and Security (December 17-19, 2026, Cox's Bazar, Bangladesh). Paper: *Quantifying Phenotype-Definition Leakage in Asthma-COPD Classification Using NHANES Data*.

This repository contains the full analysis pipeline for a four-arm ablation study on NHANES 2007-2012. In population survey data the COPD phenotype is commonly defined by a spirometric threshold (FEV<sub>1</sub>/FVC < 0.70) while the same spirometric measurements are supplied as predictors, so a model can partially recover the labeling rule rather than learn an independent diagnostic signal. We measure how large that effect is, and whether removing the defining ratio is enough to eliminate it.

> **Headline finding:** Removing all spirometric predictors drops XGBoost AUROC from 0.959 to 0.818. Removing only the FEV<sub>1</sub>/FVC ratio accounts for just 23-51% of that effect, because FEV<sub>1</sub> and FVC individually carry the label-defining information. The residual persists for logistic regression, which cannot form a ratio internally.

---

## Key design features

| Feature | What it means |
|---|---|
| **Four-arm ablation** | A: full set (25 features); B: ratio removed (24); C: all spirometry removed (21); D: spirometry and race removed (20). |
| **Shared outer folds** | All four arms see byte-identical splits, so arm-to-arm differences are paired within folds rather than confounded by split variance. |
| **Nested repeated CV** | Stratified 5-fold x 10 repeats. Hyperparameters tuned on 75% of each outer training set, decision threshold selected on the remaining 25%, never on test. |
| **Fold-internal preprocessing** | Imputation, encoding, and scaling are fitted on training folds only. |
| **Label-constructing variables excluded** | The four questionnaire items defining the phenotypes are dropped from the predictor set. |
| **Survey-weighted descriptives** | Six-year MEC weights with Taylor linearization over SDMVPSU and SDMVSTRA for population estimates; predictive models fitted unweighted. |
| **Missingness sensitivity** | Complete-spirometry (n = 649) and complete-FeNO (n = 666) reruns test whether differential missingness inflates performance and feature importance. |

---

## Analysis protocol

<!-- Replace the placeholder path below with the workflow figure from the paper.
     Either commit the image to results/ and reference it, or paste a GitHub
     attachment URL as in the example comment at the end of this block. -->

<p align="center">
     <img width="3814" height="1661" alt="Analysis workflow: cohort construction, four-arm feature ablation, nested cross-validation, paired ablation analysis, and SHAP interpretability" src="https://github.com/user-attachments/assets/24bfaa59-453d-4136-ae10-dd66acdfd432" />
  <br>
  <em>Analysis workflow. Cohort construction and feature ablation feed a nested cross-validation protocol in which hyperparameters and the decision threshold are selected inside each training fold. The four arms share outer splits, so ablation effects are estimated pairwise within folds.</em>
</p>
---

## Repository layout

```
src/        Analysis notebooks (run in numbered order)
data/       NHANES .xpt files, by survey cycle (not tracked - see below)
results/    Figures and exported tables
```

---

## Data

This project uses the **National Health and Nutrition Examination Survey (NHANES)**,
cycles **2007-2008**, **2009-2010**, and **2011-2012**. These three cycles were chosen
because they are the last to carry both spirometry and fractional exhaled nitric oxide
(FeNO) examination components.

Nine components are pulled from each cycle:

| Component | Description |
|-----------|-------------|
| `DEMO` | Demographics |
| `BMX`  | Body measurements |
| `CBC`  | Complete blood count |
| `ENX`  | Exhaled nitric oxide |
| `MCQ`  | Medical conditions |
| `OCQ`  | Occupation |
| `RDQ`  | Respiratory health |
| `SMQ`  | Smoking |
| `SPX`  | Spirometry |

Cycle suffixes are `_E` (2007-2008), `_F` (2009-2010), and `_G` (2011-2012) — for
example, `DEMO_E.XPT`.

### Getting the data

The `.xpt` files are **not included in this repository**. They are U.S. federal
public-domain works and are redistributed by NCHS, not by us. Download them from the
CDC and place them in the matching cycle folder:

```
Dataset/
├── 2007_2008/   DEMO_E.XPT, BMX_E.XPT, CBC_E.XPT, ...
├── 2009_2010/   DEMO_F.XPT, BMX_F.XPT, CBC_F.XPT, ...
└── 2011_2012/   DEMO_G.XPT, BMX_G.XPT, CBC_G.XPT, ...
```

Download page: https://wwwn.cdc.gov/nchs/nhanes/

The loader walks `Dataset/` recursively and matches on filename, so subfolder naming
is flexible — but all 27 files (9 components x 3 cycles) must be present or the
notebook raises `FileNotFoundError` listing what is missing.

### Citing the data

> Centers for Disease Control and Prevention (CDC), National Center for Health
> Statistics (NCHS). *National Health and Nutrition Examination Survey Data,
> 2007-2008, 2009-2010, and 2011-2012.* Hyattsville, MD: U.S. Department of Health
> and Human Services, Centers for Disease Control and Prevention.
> https://wwwn.cdc.gov/nchs/nhanes/

NHANES is a complex, multistage probability sample. Any population-level estimate
derived from it must use the survey design variables (`WTMEC2YR`, `SDMVPSU`,
`SDMVSTRA`); see the NCHS analytic guidelines before reusing this code for
descriptive work.

---

## Setup

Requires [conda](https://docs.conda.io/) (or mamba/micromamba).

```bash
git clone https://github.com/[ORG]/asthma-copd-v2.git
cd asthma-copd-v2

conda env create -f environment.yml
conda activate asthma-copd
```

Then launch the notebooks:

```bash
jupyter lab
```

Run the notebooks in `Code/` in numbered order. The first builds the analytic cohort;
later ones depend on objects it defines.

To record exactly what you ran, export a lock file:

```bash
conda env export --no-builds > environment.lock.yml
```

---

## Contributing

Contributions are welcome. This repository uses the standard fork-and-pull-request
workflow:

1. **Fork** the repository to your own GitHub account.
2. **Clone** your fork locally and create a branch for your change:
   ```bash
   git checkout -b fix/threshold-selection
   ```
3. **Make your changes** and commit them with a clear message explaining *why*, not
   just what.
4. **Push** the branch to your fork:
   ```bash
   git push origin fix/threshold-selection
   ```
5. **Open a pull request** against the `main` branch of this repository, describing
   what you changed and how you verified it.

A few things that make review faster:

- Keep one logical change per pull request.
- If a change affects reported numbers, say which table or figure moves and by how
  much.
- Do not commit `.xpt` data files or regenerated CSV/PNG outputs — `.gitignore`
  covers these.
- Open an issue first for anything large, so we can agree on the approach before you
  spend time on it.

All participants are expected to follow the [Code of Conduct](CODE_OF_CONDUCT.md).

---

## Citation

If you use this code or protocol, please cite our paper:

```bibtex
@inproceedings{rishad2026quantifying,
  title     = {Quantifying Phenotype-Definition Leakage in Asthma-COPD
               Classification Using NHANES Data},
  author    = {Rishad, Nahian Islam and Afifa, Mst. Fabiha and
               Faisal, Farhan Bin and Nur, Kamruddin},
  booktitle = {Proceedings of the 13th International Conference on Next
               Generation Computing, Communication, Systems and Security
               (NSysS '26)},
  year      = {2026},
  month     = dec,
  address   = {Cox's Bazar, Bangladesh},
  publisher = {ACM},
  note      = {DOI to be added upon publication}
}
```

Citation metadata is also in [`CITATION.cff`](CITATION.cff), and GitHub will generate
a formatted citation from the "Cite this repository" button on the repository page.

---

## Authors and contributions

| Author | Contribution |
|---|---|
| **Nahian Islam Rishad** (American International University-Bangladesh) | Ideation, code, writing |
| **Mst. Fabiha Afifa** (American International University-Bangladesh) | Ideation, code, writing |
| **Farhan Bin Faisal** (American International University-Bangladesh) | Ideation, code, writing |
| **Kamruddin Nur** (American International University-Bangladesh) | Proofreading |

---

## License

The code in this repository is licensed under the license in the [LICENSE](LICENSE)
file — you are free to use and modify it.

**The data is not covered by this license.** NHANES data files are U.S. federal
public-domain works distributed by the National Center for Health Statistics. Please
refer to the NCHS terms of use and analytic guidelines before redistributing or
reusing them: <https://wwwn.cdc.gov/nchs/nhanes/>.
