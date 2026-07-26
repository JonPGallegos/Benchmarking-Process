---
name: ipeds-benchmarking-process
description: Build, explain, reproduce, or validate an institutional benchmarking workflow using IPEDS data. Use when an agent must retrieve annual IPEDS survey files, union collections across years, interpret IPEDS variables, calculate benchmarking metrics, identify peer institutions, apply k-means/PCA/cosine similarity, or implement the methodology demonstrated by JonPGallegos/Benchmarking-Process.
---

# IPEDS Benchmarking Process

Use this skill as a methodology and data-meaning guide. Do not assume the example notebooks implement every metric described here.

## Establish the analysis

1. Define the institutional question and intended audience before selecting metrics.
2. Choose metrics aligned with institutional strategy, accreditation, continuous improvement, or peer comparison.
3. Identify the target institution by `UNITID`; use `156107` only when reproducing the WSU Tech example.
4. Define the reporting years, applicable IPEDS surveys, peer-pool criteria, and expected output grain.
5. Consult the IPEDS data dictionary for every survey year. Variable names, availability, codes, and meanings can change between collections.

Maintain one row per institution and collection year before joining metric inputs.

## Understand the demonstrated repository

Treat the repository as two related demonstrations:

- `Create Dataset NB.ipynb` retrieves annual IPEDS files, standardizes them, unions like surveys across years, joins metric inputs, and calculates the Outcome Efficiency Index for WSU Tech.
- `PCA and Cosine Similarity NB.ipynb` uses a prepared, single-year institutional metric table to cluster potential peers, visualize them with PCA, and rank similarity to a reference institution.
- `2026-air-forum-handout.pdf` describes the broader benchmarking methodology and additional metrics. Most of those additional metrics are definitions, not implemented pipeline code in the repository.

The committed example covers collection years 2021 through 2024.

## Retrieve and organize IPEDS data

For each year and required survey:

1. Generate or confirm the NCES/IPEDS download location.
2. Download the ZIP archive and verify the response before parsing it.
3. Prefer the final revised `_RV` CSV when it exists; otherwise use the standard CSV.
4. Decode source CSVs using `ISO-8859-1` when necessary.
5. Normalize column names by trimming whitespace, replacing periods with underscores, and removing characters other than letters, digits, and underscores.
6. Preserve raw annual extracts under `data/{year}/`.

The example retrieves:

- `C{year}_B`: completions data.
- `HD{year}`: institutional directory and characteristics.
- `F{priorYY}{yearYY}_F1A`: finance data for public institutions.

IPEDS access differs across years. The notebook uses direct Data Center ZIP URLs before 2023 and the data-generator endpoint for 2023 onward. Reconfirm current access behavior rather than assuming those URLs remain stable.

## Union annual collections

Union the same survey component across reporting years:

1. Load the revised file when available.
2. Add `YEAR` as the collection year.
3. Add `REVISED`, using `Y` for `_RV` sources and `N` otherwise.
4. Concatenate by column name and allow fields absent from older collections to remain null.
5. Save the result under `data/unioned_table/`.

Interpret `YEAR` and `REVISED` as pipeline-derived provenance fields, not original IPEDS variables.

Do not treat automatic column alignment as proof of semantic consistency. Verify that similarly named fields retain the same meaning across years.

## Join metric inputs

Join cross-survey facts on both `UNITID` and `YEAR`. Use `UNITID` alone only for attributes intentionally taken from a single directory snapshot.

The example:

1. Inner-joins unioned `CYEAR_B` to unioned `FYRYR_F1A` on `UNITID` and `YEAR`.
2. Selects `F1C191` from finance.
3. Joins `INSTNM` from `HD2024` on `UNITID`.
4. Filters `INSTNM` to WSU Tech.

Recognize the implications:

- Inner joins discard institution-years missing from either source.
- Applying `HD2024` to all years uses a current directory snapshot for historical records.
- Filtering by institution name is less stable than filtering by `UNITID`.
- Before joining tables with coded subgroups, filter each table so its grain is one row per `UNITID` and `YEAR`.

## Interpret the implemented Outcome Efficiency Index

Use:

```text
OUTCOME_EFFICIENCY_INDEX = CSTOTLT * 100,000 / F1C191
```

Interpret the inputs as:

- `CSTOTLT`: total distinct students completing during the period.
- `F1C191`: total expenses.
- Result: completions per $100,000 of total expenses in the same fiscal year.

Higher values indicate more completions per $100,000 spent. Do not describe this as cost per completion.

The notebook code and committed output implement this formula correctly. Its explanatory Markdown reverses the meanings of `CSTOTLT` and `F1C191`; do not repeat that prose error.

For the committed WSU Tech output, validate:

```text
2021: 3.2273701664
2022: 3.2865177877
2023: 3.2829089103
2024: 2.9656197379
```

Reject or explicitly handle zero, missing, negative, or nonnumeric expenses before division. State whether completions and expenses truly refer to compatible periods.

## Apply the broader metric definitions

Treat these as methodology definitions from the handout. Verify every variable and code against the applicable annual IPEDS dictionary before implementation.

### First-Year Retention Rate

```text
(RET_NMF + RET_NMP) / (RRFTCTA + RRPTCTA)
```

Source: `EF20##D`.

### Completion and Transfer Rate

```text
(OMCERT8 + OMASSC8 + OMBACH8 + OMENRAI) / OMACHRT
```

Source: `OM20##`. Filter `OMCHRT = 50` in the demonstrated definition.

### Graduation Rate Within 150 Percent

Use `GRTOTLT` from `GR20##`. For two-year schools, divide the row with `GRTYPE = 3` by the row with `GRTYPE = 2`. For four-year schools, divide `GRTYPE = 30` by `GRTYPE = 29`. Confirm the codes for the selected collection.

### Composite Financial Index

Calculate four weighted components, clip each normalized score to `[-4, 10]`, and sum them:

```text
PRR  = (F1A17 + F1A15) / F1D02
C1   = 0.35 * clip(PRR / 0.133, -4, 10)

ORR  = (F1B27 - F1C191) / (F1B27 - F1C19IN)  # through 2019
ORR  = F1N01 / F1N02                          # after 2019
C2   = 0.10 * clip(ORR / 0.013, -4, 10)

RONA = F1D03 / F1D04
C3   = 0.20 * clip(RONA / 0.02, -4, 10)

VR   = F1N05 / F1A10
C4   = 0.35 * clip(VR / 0.417, -4, 10)

CFI  = C1 + C2 + C3 + C4
```

The handout assigns the maximum viability component when viability inputs are missing. Make that policy explicit if reproducing it. Verify the finance-variable transition around 2020.

### Other documented metrics

- Expenditure per FTE student: `F1C191 / EFTEUG`; sources `F2#2#_F1A` and `EFIA20##`.
- Student-to-faculty ratio: `STUFACR`; source `EF20##D`.
- Fall enrollment: `ENRTOT`; source `DRVEF20##`.
- Weighted Market Share Index: combine estimated recent high-school graduates within 25 miles with the target institution's share of local degree-seeking enrollment. Compute degree-seeking enrollment as `EFUG1ST + EFUGTRN + EFUGCNT`, use `EFRES02` for recently graduated high-school students where applicable, and source local grade-12 enrollment from Common Core Data. Document the geographic method, state filter, graduation-rate proxy, academic-year alignment, and denominator population.

## Establish a baseline peer pool

Filter institutions to comparable high-level characteristics before modeling. The demonstrated approach uses directory-survey fields for:

- Control: public versus private.
- Institution size: `CARNEGIESIZE`.
- College type or basic classification: `C21BASIC`.

The handout's WSU Tech example uses public, medium-to-very-large, associate's colleges and reports a baseline pool of 444 institutions. Treat this number as an example, not a permanent expected result.

Prefer a defensible peer pool over applying clustering to all institutions.

## Prepare peer-model features

The sample peer dataset has one row per `UNITID` and these five features:

- `FIRST_YEAR_RET_RATE`
- `GRAD_RATE_WITHIN_150`
- `COMP_TRAN_RATE`
- `OUTCOME_EFFICIENCY_INDEX`
- `FALL_ENROLLMENT_TOTAL`

Prepare it as follows:

1. Keep `UNITID` only as an identifier; exclude it from modeling.
2. Require numeric feature columns.
3. Impute missing feature values with each feature's median.
4. Standardize each feature to zero mean and unit variance.
5. Fit transformations on the intended comparison population. Avoid data leakage when applying a model to held-out or future data.

Standardization is mandatory for the demonstrated distance-based methods.

## Cluster and visualize peers

1. Evaluate plausible `k` values using inertia and silhouette score.
2. Choose `k` using evidence and interpretability, not a permanently hardcoded value.
3. Fit k-means with a fixed `random_state` and documented `n_init`.
4. Record the fitted preprocessing and model configuration.
5. Locate the reference institution by `UNITID`.
6. Reduce the standardized features to two principal components for visualization.
7. Report explained variance and label axes `Principal Component 1` and `Principal Component 2`.
8. Treat PCA plots as interpretive aids; overlap in two dimensions does not prove institutions are dissimilar or misclustered.

The sample chooses `k = 8` and `random_state = 10`. Numeric k-means labels have no inherent rank or meaning. The notebook's descriptive cluster mapping is specific to that fitted sample and must not be reused automatically after refitting.

## Rank with cosine similarity

Calculate cosine similarity in the same standardized feature space used for clustering:

```text
similarity(reference_vector, institution_vector)
```

Interpret `1` as identical orientation, `0` as orthogonal, and `-1` as opposite orientation. Rank higher scores as more similar, while retaining substantive peer criteria.

Find the reference row by `UNITID`; never hardcode row index `95`. The example uses index 95 only because `UNITID = 156107` happens to occupy that position in the committed 408-row sample.

The reference institution must have similarity approximately equal to `1`. Do not impose a "closest-to-farthest" cluster order unless that order was derived and documented for the current fitted model.

## Validate the result

Require:

- Unique `UNITID` plus `YEAR` at every annual metric grain.
- No unintended row multiplication after joins.
- Explicit counts of records excluded by inner joins and filters.
- Provenance for survey name, collection year, revision status, and download date.
- Formula checks using hand-calculated examples.
- Expected units and plausible numeric ranges for every metric.
- Reference cosine similarity approximately `1`.
- Reproducible cluster results from fixed seeds.
- Review of annual IPEDS definitions for renamed variables and changed codes.
- Clear separation between raw fields, derived provenance fields, calculated metrics, and modeling labels.

When asked to reproduce the repository exactly, preserve its behavior and report known limitations. When asked to implement the methodology robustly, replace hardcoded institution names, row positions, cluster meanings, year ranges, and file conventions with validated parameters.
