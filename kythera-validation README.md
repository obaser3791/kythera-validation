# kythera-validation

[![CI](https://github.com/obaser3791/kythera-validation/actions/workflows/ci.yml/badge.svg)](https://github.com/obaser3791/kythera-validation/actions/workflows/ci.yml)
[![Python](https://img.shields.io/badge/python-3.10%20%7C%203.11%20%7C%203.12%20%7C%203.13-blue.svg)](https://www.python.org/)
[![Licence: MIT](https://img.shields.io/badge/licence-MIT-green.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-2.0.0-informational.svg)](CHANGELOG.md)
[![Tests](https://img.shields.io/badge/tests-110%20passing-brightgreen.svg)](tests/)

Reference implementation of the provenance-anchored nine-block validation framework applied to
the Kythera Labs all-payer real-world data asset.

This repository accompanies **Baser O. Validation of the Kythera Labs all-payer real-world data
asset: a provenance-anchored nine-block framework for tokenized evidence generation.** *BMC
Medical Research Methodology*.

The package does three separable things, and the distinction matters when reading the output:

1. **It states the framework.** All seventeen estimators, all sixteen pre-registered acceptance
   thresholds, the thirteen intake conformance gates, the six claims-based condition
   definitions and the two-factor measurement model are declared in code and configuration,
   not in prose. They can be inspected, criticised and reused without access to any licensed
   data.
2. **It verifies the manuscript against itself.** `kythera-validate verify` recomputes the
   published quantities that are recoverable from the values printed in the tables — the
   cohort ladder, the linkage two-by-two table, the reliability coefficients, the fit indices —
   and fails if the code and the manuscript disagree.
3. **It runs the whole pipeline against a surrogate cohort.** Because the licensed asset cannot
   be redistributed, the repository ships a generator that produces a cohort with the same
   schema, the same missingness structure and the same marginal distributions as the analytic
   file. Running the pipeline against it demonstrates that the code executes, that each block
   returns what the tables require, and that the thresholds are applied — it does **not**
   reproduce the published point estimates, and no output from it should be cited as though it
   did.

## Contents

- [Installation](#installation)
- [Quick start](#quick-start)
- [What each block does](#what-each-block-does)
- [Reproducibility scope](#reproducibility-scope)
- [Repository layout](#repository-layout)
- [Additional files](#additional-files)
- [Reproduction notebook](#reproduction-notebook)
- [Configuration](#configuration)
- [Tests](#tests)
- [A note on the random streams](#a-note-on-the-random-streams)
- [Data availability](#data-availability)
- [Ethics approval and funding](#ethics-approval-and-funding)
- [Licence and citation](#licence-and-citation)
- [Contributing](#contributing)
- [Contact](#contact)

## Installation

```bash
git clone https://github.com/obaser3791/kythera-validation.git
cd kythera-validation
pip install -e .
```

Or with conda:

```bash
conda env create -f environment.yml
conda activate kythera-validation
```

Requires Python 3.10 or later; continuous integration runs the suite on 3.11, 3.12 and 3.13.
The dependency set is deliberately small: numpy, pandas, scipy, scikit-learn, matplotlib and
pyyaml. No proprietary or commercially licensed statistical software is needed to run anything
in this repository.

## Quick start

```bash
# What is declared: thresholds, blocks, conditions, intake gates
kythera-validate info

# The pre-registered acceptance criteria, with source and rationale for each cutoff
kythera-validate thresholds
kythera-validate thresholds --json

# Check the code against the published values
kythera-validate verify --n 20000

# Run all nine blocks against a surrogate cohort and write tables, figures and results
kythera-validate run --n 250000 --simulate-linkage --n-pairs 1000000 --out outputs/full

# Write a surrogate cohort to disk for independent inspection
kythera-validate surrogate --out data/surrogate --n 250000
```

Or use the Makefile, which fixes the seed and cohort size used in the manuscript:

```bash
make verify     # agreement check against the published tables
make run        # full pipeline, 250,000 surrogate patients
make test       # the test suite
make figures    # regenerate the manuscript figures
make all
```

Every command is deterministic given `--seed` (default 20250731). Two runs at the same seed and
cohort size return identical results, and the seed is recorded alongside the package version in
the `results.json` written to the output directory.

## What each block does

| Block | Question | Primary estimators | Pre-registered criterion |
| --- | --- | --- | --- |
| A | Do composite measures cohere and hold still over time? | Cronbach's alpha, two-way random-effects ICC, positive specific agreement | alpha >= 0.80, ICC >= 0.70 |
| B | Do prevalence and utilisation estimates agree with national survey benchmarks? | MAPE, Lin's concordance correlation | MAPE <= 0.05, CCC >= 0.90 |
| C | Does the intended factor structure hold? | Polychoric CFA by WLSMV, CFI, TLI, RMSEA, SRMR, Fornell-Larcker | CFI and TLI >= 0.95, RMSEA <= 0.06, SRMR <= 0.08 |
| D | Is the cohort representative of the United States population? | Standardized mean differences against ACS, iterative proportional fitting | absolute SMD < 0.10 after raking |
| E | Is missingness ignorable, and what does it cost? | Little's MCAR test, MICE with Rubin pooling, fraction of missing information | pooled FMI <= 0.30 |
| F | How accurate is tokenized linkage? | Fellegi-Sunter weights, derived two-threshold operating point, blinded adjudication | match rate >= 0.90, false-match rate <= 0.005 |
| G | Do independent coders agree on diagnostic assignment? | Cohen's kappa with Landis-Koch bands, positive predictive value | kappa >= 0.61 per category |
| H | Is data quality stable across the study period? | Mann-Kendall trend test, CUSUM change detection, minimum detectable effect | absolute Kendall tau <= 0.20 |
| I | Is the asset fit for prediction? | Logistic and gradient-boosted models, AUROC, calibration slope and intercept, Brier score | AUROC >= 0.75, Brier <= 0.20 |

Every block returns its own pre-registered checks as structured records, and every check
carries the observed value, the operator, the threshold, the literature source of the cutoff
and whether it was met. `run` collects them into a single summary table, so a failure is
visible rather than buried.

## Reproducibility scope

This is stated plainly because the alternative is to imply more than the repository can
deliver:

**Fully reproducible from public inputs.** All seventeen equations; the NHANES, MEPS and ACS
comparison targets; and Blocks B, C, D and H, which operate on published marginals and summary
statistics rather than on record-level data.

**Structure reproducible against the surrogate cohort only.** Blocks A, E and I require
record-level data. Running them against the surrogate demonstrates that the code is correct
and complete; it does not reproduce the published estimates, and the surrogate's parameters
were chosen to match published marginals rather than derived from the licensed file.

**Not reproducible outside the licensed environment.** The Block F adjudication file and the
Block G intercoder assignments contain protected health information and cannot be released.
They are published instead as de-identified aggregate classification tables, which are
sufficient to recompute every accuracy statistic reported in the manuscript but not to
reconstruct any individual record.

## Repository layout

```
config/            pre-registered thresholds, condition definitions, CFA model,
                   and the values transcribed from the manuscript tables
src/kythera_validation/
  equations.py     the seventeen estimators, each with its variance formula
  rngs.py          named independent random streams
  config.py        typed loaders for every configuration file
  blocks/          one module per validation block, A through I
  linkage/         Fellegi-Sunter weights, threshold derivation, adjudication
  provenance/      intake conformance gates and the conditionality statement
  cohort/          the attrition ladder and the condition rules
  surrogate/       the surrogate cohort generator
  reporting/       manuscript tables and figures
  cli.py           the command-line interface
additional_files/  the five additional files submitted with the manuscript,
                   in markdown, Word and PDF, plus the nine runnable workflows
notebooks/         an end-to-end reproduction notebook
tests/             the test suite
```

## Additional files

The five additional files submitted with the manuscript are versioned here, in markdown
alongside the typeset Word and PDF renderings under `additional_files/docx/` and
`additional_files/pdf/`.

| File | Contents |
| --- | --- |
| 1 | Estimators, variance formulae, pre-specified thresholds with the rationale for each cutoff, and a worked numerical example for all seventeen equations |
| 2 | Methodological appendix: intake conformance gates, the conditionality statement, the linkage operating point and its sensitivity, and the surrogate cohort specification |
| 3 | Codebook and data dictionary: every variable, its type, permissible values, derivation rule and missingness treatment |
| 4 | Example workflows: nine runnable scripts, one per block, with the command line and the captured output |
| 5 | Reproduction notebook: cell-by-cell description of the end-to-end run and the checks it asserts |

The workflow scripts in `additional_files/workflows/` execute against the installed package:

```bash
PYTHONPATH=src python additional_files/workflows/workflow_block_a.py
```

## Reproduction notebook

`notebooks/reproduction.ipynb` runs the framework end to end at the manuscript's cohort size
and seed, printing each pre-registered check beside its threshold. It asserts forty-seven
checks and is expected to finish with none unmet.

```bash
jupyter lab notebooks/reproduction.ipynb
```

## Configuration

Nothing substantive is hard-coded in the analysis modules. Four YAML files hold the
pre-registered content, and changing a criterion means editing configuration rather than code:

| File | Holds |
| --- | --- |
| `config/thresholds.yaml` | the sixteen acceptance criteria, each with operator, value, literature source and the rationale for rejecting the neighbouring cutoffs |
| `config/conditions.yaml` | the six claims-based condition definitions, as code lists with the required diagnosis pattern |
| `config/cfa_model.yaml` | the two-factor measurement model, its indicators and its estimator settings |
| `config/reported_values.yaml` | the values transcribed from the manuscript tables, against which `verify` checks the code |

## Tests

```bash
make test
# or
PYTHONPATH=src pytest -q
PYTHONPATH=src pytest -q -m "not slow"   # skip the end-to-end pipeline tests
```

The tests compare each estimator against a closed-form value, a published worked example, or
an analytic property it must satisfy — not against whatever the implementation currently
returns. A test that only records current behaviour would still pass after the code broke.

Two classes of test exist because two classes of error occur. The first checks the mathematics.
The second checks that the configuration and the manuscript have not drifted apart: that every
threshold key is still present, that the cohort ladder is monotone, that the linkage
two-by-two table still yields the sensitivity printed in the paper.

## A note on the random streams

Every stochastic step draws from a named stream derived from the run seed. This is not
fastidiousness. During development, the train-test split indicator was drawn from the same seed
as the surrogate data; the two streams coincided, the split stopped being independent of the
outcome, and an apparent c-statistic moved by more than a tenth. The failure produced entirely
plausible numbers and raised nothing. `rngs.stream(seed, purpose)` makes the collision
impossible, and a test asserts that no two streams in the package agree.

## Data availability

The Kythera Labs all-payer data asset is licensed and cannot be redistributed; it is available
from Kythera Labs under a data use agreement. The public survey benchmarks used in Block B and
Block D — NHANES, MEPS and the American Community Survey — are freely available from their
respective agencies, and the targets used here are recorded in `config/`. The Block F
adjudication and Block G intercoder files contain protected health information and are released
only as de-identified aggregate classification tables, which are sufficient to recompute every
accuracy statistic in the manuscript. Everything else needed to run the framework is in this
repository.

## Ethics approval and funding

The study was determined to be exempt and no institutional review board approval was required,
as the analysis used de-identified data and did not constitute human subjects research. Funding
was received from Columbia Data Analytics.

## Licence and citation

MIT licence. See `LICENSE`.

If you use this framework, please cite the manuscript. `CITATION.cff` carries machine-readable
metadata, and GitHub's "Cite this repository" control will render it in BibTeX or APA.

```bibtex
@article{baser_kythera_validation,
  author  = {Baser, Onur},
  title   = {Validation of the Kythera Labs all-payer real-world data asset:
             a provenance-anchored nine-block framework for tokenized evidence generation},
  journal = {BMC Medical Research Methodology},
  year    = {2026}
}
```

## Contributing

Issues and pull requests are welcome, particularly ones that identify an estimator implemented
incorrectly or a threshold whose stated justification does not hold. Please open an issue before
a substantial change so the design question can be settled first. A pull request should keep the
test suite green, add a test that would have failed before the change, and leave the
pre-registered content in `config/` untouched unless the change is specifically to that content
— in which case say so explicitly, because it changes what the framework claims.

## Contact

Onur Baser — onur.baser@sph.cuny.edu
Graduate School of Public Health and Health Policy, City University of New York
