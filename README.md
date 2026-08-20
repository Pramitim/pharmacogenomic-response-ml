Can increasingly nonlinear models predict sensitivity to a particular cancer drug from gene-expression data better than linear regression?

1. Build the merged dataset — pull the overlapping cell lines for your chosen drug out of the TPM file, transpose it, and join it with the AUC values to get your final X (gene expression) and y (AUC) — this is Steps 3–6 in your README.
2. Handle dimensionality — with likely 500ish samples and 20,000 genes, you'll need to decide on feature selection or PCA, and you'll want to try that interactively.
3. Train/test split, standardize features — using the training set only, to avoid leakage.
4. Build and compare models — Linear Regression → Random Forest → PyTorch MLP (per your README, with XGBoost as optional), running and comparing each in cells so you can see results immediately.
   Evaluate — RMSE/MAE/R² per model, plots comparing them.
   Write up your findings — markdown cells explaining what you found, whether your hypothesis held.

01_data_exploration.ipynb — everything you just did: EDA, missing values, choosing the drug (keep this as-is, it's your record of why you picked Docetaxel/whichever ID).
02_build_dataset.ipynb — merging TPM + GDSC data for your chosen drug, building final X/y, dimensionality handling.
03_modeling.ipynb — train/test split, standardization, and the model comparison (Linear Regression → Random Forest → PyTorch MLP).

STEP 1: Choose one drug
↓
STEP 2: Get all cell lines tested with that drug
↓
STEP 3: Check which of those SANGER_MODEL_IDs
also exist in the TPM file
↓
STEP 4: Keep only the overlapping cell lines
↓
STEP 5: Transpose the TPM data
↓
STEP 6: Build X and y

                 TPM DATA
                     ↓

SIDM01132 → [gene1, gene2, gene3, ..., geneN]
SIDM00848 → [gene1, gene2, gene3, ..., geneN]
↓
X

                GDSC2 DATA
                     ↓

SIDM01132 → AUC = 0.930
SIDM00848 → AUC = 0.615
↓
y

# Two data sources

1. [RNA Sequence TPM](data/processed/rnaseq_merged_rsem_tpm_20260323.csv)

Each SIDM... value identifies a specific cancer cell model. Each column corresponds to one cell model, while each row corresponds to a gene. The number where a gene and cell model intersect represents that gene's measured expression level in that cell, expressed as TPM. A value of zero means no measurable expression was detected or the processed expression value is zero; it does not necessarily mean the gene is absent from the cell. Because the data is TPM-normalized, the expression values across all genes in a sample sum to approximately 1,000,000 rather than 100.

2. [Cell response to drug](data/processed/GDSC2_fitted_dose_response_27Oct23.csv)

Each cell line is tested against a drug. The sensitivity of the response is recorded in different measurements (AUC, LN_IC50, RMSE)

### Which drug to choose:

should choose a drug that has a good number of cell lines with drug-response measurements and overlapping TPM expression data.

What matters is the overlap: cell lines that have both a drug-response measurement AND expression data. If a drug looks well-tested in GDSC but half those cell lines aren't in your Cell Model Passports expression file, your real usable sample size is much smaller than it looks.

#### Process for choosing drug

1. Check sample size per drug

First, merge your two datasets on cell line ID (using that model_id/COSMIC mapping we talked about) — even before picking a drug.
Then group by drug and count non-null response values within that merged table only. This is now your real, honest sample size per drug — accounts for both response and expression availability at once. You want a drug tested across as many cell lines as possible — more samples means a more trainable, more defensible model. Aim for at least 300-400 cell lines; below that, your train/val/test splits get too small to trust.

2. Check response variance
   For your top candidates by sample size, check the spread (standard deviation) of IC50/AUC values. If a drug's response is nearly identical across all cell lines, there's nothing for your model to actually learn or predict. You want real variance — some clearly sensitive cell lines, some clearly resistant.

3. Prefer a targeted therapy over broad chemo
   From the drugs that pass both filters, prefer a targeted therapy with a known biomarker over a broad cytotoxic chemo drug. Something like Erlotinib (EGFR inhibitor) or a BRAF inhibitor has a known, well-documented gene-expression/mutation relationship to response. This matters because it means you can sanity-check your model's top predictive features against real biology — if your model highlights genes unrelated to the known pathway, that's a useful, explainable finding either way.

4. Sanity check: has anyone published on this drug already
   Search 'Erlotinib GDSC prediction' or similar for your top candidate. If there's existing published work predicting response to that drug from expression data, that becomes your direct comparison point — you can state how your result compares to a known benchmark, which is exactly the kind of rigor that makes a project credible.

### WHY AUC?

- area under curve
  "I will use AUC as the continuous target variable representing drug sensitivity."

AUC vs LN_IC50: AUC (area under the dose-response curve) is often the preferred target in GDSC-based modeling over IC50, because it summarizes the whole curve rather than one point — it doesn't require the curve to actually reach 50% inhibition within the tested concentration range, so drugs that were only mildly effective still get a well-defined AUC value (whereas their IC50 might be extrapolated/unreliable). Lower AUC = more sensitive, higher AUC = more resistant, values are typically normalized between 0 and 1.

## Process

1. First, understand exactly what your prediction problem is

Your project can be reduced to:

Input: gene-expression profile of a cancer cell line
Output: that cell line's response to one specific drug

So conceptually:

Cell line → gene expression → ML model → predicted drug response

The first thing you need to decide is:

Am I predicting the response to one drug, or many drugs?

For your three-day project, I strongly recommend starting with ONE drug.

For example:

Gene expression of Cell Line A → predicted response to Drug X

Gene expression of Cell Line B → predicted response to Drug X

etc.

That makes your research question much cleaner:

Can increasingly nonlinear models predict sensitivity to a particular cancer drug from gene-expression data better than linear regression?

Once that works, you could potentially extend it to multiple drugs.

2. Understand what your dataset actually looks like

Before modeling anything, you need to understand the relationship between the datasets.

You will probably encounter separate pieces of information such as:

Gene-expression data
Cell line Gene 1 Gene 2 Gene 3 ...
Cell A 4.2 8.7 1.3 ...
Cell B 5.1 6.4 2.1 ...
Cell C 3.8 7.9 1.8 ...

This becomes X.

Drug-response data
Cell line Drug Response
Cell A Drug X 0.72
Cell B Drug X 0.34
Cell C Drug X 0.61

This becomes y.

You then have to join these datasets using the cell-line identifier.

That's a very important part of the project.

Conceptually:
Gene expression Drug response
↓ ↓
Cell A → [4.2, 8.7, 1.3, ...] Cell A → 0.72
Cell B → [5.1, 6.4, 2.1, ...] Cell B → 0.34
Cell C → [3.8, 7.9, 1.8, ...] Cell C → 0.61
↓
JOIN
↓
Final ML dataset
↓
X y

       3. Decide exactly what the response variable means

This is another thing you need to figure out conceptually.

“Drug sensitivity” isn't just one universal number.

Pharmacogenomic datasets can contain different response measures, such as:

IC50
AUC
viability
sensitivity scores
other drug-response measurements

You need to choose one response metric.

For example:

"I will use [response metric] as the continuous target variable representing drug sensitivity."

Then your problem is clearly a regression problem.

You're not asking:

Sensitive vs resistant?

You're asking:

What numerical response should we expect for this cell line?

That's why linear regression, Random Forest regression, XGBoost regression, and a neural-network regressor make sense.

his is probably the single most important conceptual question.

For your initial experiment, I would want your final dataset to conceptually look like:

Cell line Gene 1 Gene 2 Gene 3 ... Drug response
Cell A ... ... ... ... 0.72
Cell B ... ... ... ... 0.34
Cell C ... ... ... ... 0.61

Therefore:

One row = one cancer cell line tested against one particular drug.

And:

Columns = gene-expression features.

Target = drug response.

If you choose one drug, every row corresponds to a different cell line.

That's a very clean experimental setup.

5. Then understand the dimensionality problem

This is where pharmacogenomic ML becomes interesting.

You might have something like:

Hundreds/thousands of cell lines

but:

Thousands/tens of thousands of genes

So you could end up with:

500 cell lines
×
20,000 gene-expression features

That's a huge number of features relative to your number of samples.

And this creates an important ML problem:

Overfitting.

Your model might effectively memorize the training data instead of learning relationships that generalize to unseen cell lines.

This will also help explain why a neural network doesn't necessarily outperform linear regression.

6. Decide how you're going to handle the high dimensionality

You don't need to solve this immediately, but you need to understand the options.

You could potentially use:

Option A — Use a subset of genes

For example:

20,000 genes
↓
filter/select
↓
1,000 genes
↓
ML model
Option B — Feature selection

Select genes according to some criterion.

Option C — Dimensionality reduction

For example, PCA:

20,000 genes
↓
PCA
↓
100 components
↓
ML model
Option D — Regularization

Especially useful for linear models.

For your first version, don't make this unnecessarily complicated.

But you need to understand that:

“I have 20,000 features and only a few hundred observations” is a major experimental consideration.

7. Understand your train/test split VERY carefully

This is especially important for biological data.

You want to answer:

Can the model predict the response of a cell line it hasn't seen before?

So conceptually:

All cell lines
↓
┌────┴────┐
↓ ↓
Training Test
80% 20%
↓ ↓
Learn Evaluate

The test set should remain untouched until you're evaluating the final model.

You don't want to accidentally let information from your test data influence preprocessing or model selection.

This is called data leakage, and it's something you should specifically understand before starting.

8. Understand standardization

Your genes may have very different numerical ranges.

For example:

Gene A: 0–5
Gene B: 0–1,000
Gene C: 0–20

You may standardize the features so they're on comparable scales.

Conceptually:

Raw gene expression
↓
Standardization
↓
Comparable feature scales
↓
ML model

But here's an important detail:

You fit the scaler on the training data only.

Then use that fitted scaler to transform validation/test data.

This is another place where data leakage can happen.

9. Understand what each model is actually testing

Don't think of your models as four unrelated things.

Think of them as an experimental progression.

Linear Regression

Question:

Can a relatively simple linear relationship explain drug response?

Random Forest

Question:

Does allowing nonlinear relationships and feature interactions improve prediction?

XGBoost

Question:

Does a more powerful nonlinear boosting approach improve prediction further?

Neural Network

Question:

Can a learned nonlinear function capture patterns that classical models miss?

So your experiment becomes:

Simple
↓
Linear Regression
↓
Random Forest
↓
XGBoost
↓
Neural Network
↓
More flexible

But more complex ≠ automatically better.

That's actually one of the most interesting things you can investigate.

10. Decide what "better" means

You need to establish this before looking at your results.

For regression, you could use:

RMSE
MAE
R²

For example:

Model RMSE ↓ MAE ↓ R² ↑
Linear Regression 0.42 0.31 0.38
Random Forest 0.39 0.29 0.44
XGBoost 0.36 0.27 0.51
Neural Network 0.45 0.33 0.30

You'd then ask:

Did the more complex model actually generalize better?

Notice that RMSE/MAE are better when lower, while R² is better when higher.

11. You need a hypothesis

Your project becomes much more research-like if you establish a hypothesis before running everything.

Something like:

Hypothesis: Nonlinear machine-learning models will achieve better predictive performance than a linear regression baseline because drug response may depend on nonlinear relationships and interactions among gene-expression features.

But importantly, you shouldn't be disappointed if this is wrong.

Suppose you get:

Linear Regression R² = 0.42
Random Forest R² = 0.39
XGBoost R² = 0.45
Neural Network R² = 0.31

That's actually interesting.

You now have something to investigate.

12. One thing I would change from your original plan

I would not commit to XGBoost yet.

Your core experiment should be:

Linear Regression → Random Forest → PyTorch MLP

XGBoost becomes an optional fourth model.

Why?

Because you're trying to learn ML, not maximize the number of algorithms in the README.

If you spend half of Day 2 fighting with preprocessing and XGBoost instead of understanding your data and PyTorch, that's counterproductive.

13. Your actual starting point

So I would divide your work into three phases.

Phase 1 — Conceptual understanding

Before coding, make sure you can answer:

What is the biological question?
What is a cell line?
What is gene expression?
What is pharmacogenomic drug sensitivity?
What does your response metric mean?
What exactly is one observation?
What are X and y?
What is regression?
Why is linear regression your baseline?
What makes Random Forest nonlinear?
What is a neural network actually learning?
What is overfitting?
What is data leakage?
Why do we need train/test separation?
What do RMSE, MAE, and R² tell you?

You do not need to master all of these before coding.

You just need enough understanding that you're not blindly running scikit-learn functions.

14. Phase 2 — Figure out the data

Then your first actual coding milestone should NOT be training a model.

It should be:

Create one clean dataset containing gene-expression features and the drug-response target for one drug.

Your first notebook might literally be:

01_data_exploration.ipynb

And your goal is simply:

Download data
↓
Understand files
↓
Inspect columns
↓
Identify cell-line IDs
↓
Identify gene-expression data
↓
Identify drug-response data
↓
Choose drug
↓
Join datasets
↓
Inspect final X and y

At the end, you should be able to print something conceptually like:

Samples: 350
Features: 1000
Target: Drug X response

That's your first major milestone.

15. Phase 3 — Build the simplest possible model

Only after the dataset is correct:

X, y
↓
Train/test split
↓
Preprocessing
↓
Linear Regression
↓
Predictions
↓
RMSE / MAE / R²

Then save those results.

8. Understand standardization

Your genes may have very different numerical ranges.

For example:

Gene A: 0–5
Gene B: 0–1,000
Gene C: 0–20

You may standardize the features so they're on comparable scales.

Conceptually:

Raw gene expression
↓
Standardization
↓
Comparable feature scales
↓
ML model

But here's an important detail:

You fit the scaler on the training data only.

Then use that fitted scaler to transform validation/test data.

This is another place where data leakage can happen.

9. Understand what each model is actually testing

Don't think of your models as four unrelated things.

Think of them as an experimental progression.

Linear Regression

Question:

Can a relatively simple linear relationship explain drug response?

Random Forest

Question:

Does allowing nonlinear relationships and feature interactions improve prediction?

XGBoost

Question:

Does a more powerful nonlinear boosting approach improve prediction further?

Neural Network

Question:

Can a learned nonlinear function capture patterns that classical models miss?

So your experiment becomes:

Simple
↓
Linear Regression
↓
Random Forest
↓
XGBoost
↓
Neural Network
↓
More flexible

But more complex ≠ automatically better.

That's actually one of the most interesting things you can investigate.

10. Decide what "better" means

You need to establish this before looking at your results.

For regression, you could use:

RMSE
MAE
R²

For example:

Model RMSE ↓ MAE ↓ R² ↑
Linear Regression 0.42 0.31 0.38
Random Forest 0.39 0.29 0.44
XGBoost 0.36 0.27 0.51
Neural Network 0.45 0.33 0.30

You'd then ask:

Did the more complex model actually generalize better?

Notice that RMSE/MAE are better when lower, while R² is better when higher.

11. You need a hypothesis

Your project becomes much more research-like if you establish a hypothesis before running everything.

Something like:

Hypothesis: Nonlinear machine-learning models will achieve better predictive performance than a linear regression baseline because drug response may depend on nonlinear relationships and interactions among gene-expression features.

But importantly, you shouldn't be disappointed if this is wrong.

Suppose you get:

Linear Regression R² = 0.42
Random Forest R² = 0.39
XGBoost R² = 0.45
Neural Network R² = 0.31

That's actually interesting.

You now have something to investigate.

12. One thing I would change from your original plan

I would not commit to XGBoost yet.

Your core experiment should be:

Linear Regression → Random Forest → PyTorch MLP

XGBoost becomes an optional fourth model.

Why?

Because you're trying to learn ML, not maximize the number of algorithms in the README.

If you spend half of Day 2 fighting with preprocessing and XGBoost instead of understanding your data and PyTorch, that's counterproductive.

13. Your actual starting point

So I would divide your work into three phases.

Phase 1 — Conceptual understanding

Before coding, make sure you can answer:

What is the biological question?
What is a cell line?
What is gene expression?
What is pharmacogenomic drug sensitivity?
What does your response metric mean?
What exactly is one observation?
What are X and y?
What is regression?
Why is linear regression your baseline?
What makes Random Forest nonlinear?
What is a neural network actually learning?
What is overfitting?
What is data leakage?
Why do we need train/test separation?
What do RMSE, MAE, and R² tell you?

You do not need to master all of these before coding.

You just need enough understanding that you're not blindly running scikit-learn functions.

14. Phase 2 — Figure out the data

Then your first actual coding milestone should NOT be training a model.

It should be:

Create one clean dataset containing gene-expression features and the drug-response target for one drug.

Your first notebook might literally be:

01_data_exploration.ipynb

And your goal is simply:

Download data
↓
Understand files
↓
Inspect columns
↓
Identify cell-line IDs
↓
Identify gene-expression data
↓
Identify drug-response data
↓
Choose drug
↓
Join datasets
↓
Inspect final X and y

At the end, you should be able to print something conceptually like:

Samples: 350
Features: 1000
Target: Drug X response

That's your first major milestone.

15. Phase 3 — Build the simplest possible model

Only after the dataset is correct:

X, y
↓
Train/test split
↓
Preprocessing
↓
Linear Regression
↓
Predictions
↓
RMSE / MAE / R²

Then save those results.

## Jupytry notebooks

- used to record how we chose drug and test feature/threshold/model

## Notes made EDA (exploratory data analysis)

- 'PUTATIVE_TARGET' has 27872 missing values
- the following have two id's
  DRUG_NAME
  Acetalax 2
  Dactinomycin 2
  Docetaxel 2
  Fulvestrant 2
  GSK343 2
  Oxaliplatin 2
  Selumetinib 2
  Ulixertinib 2
  Uprosertib 2

_result of AUC distribution curve_

- spike at 1.0
- means a large number of drug–cell line combinations showed essentially no drug effect — the cell line's viability stayed near 100% across the whole tested dose range, so the area under the curve is at (or very near) its maximum.
- Why this happens biologically: most drugs in a large screening panel like GDSC are only effective against specific cancer types or cell lines with a particular vulnerability (e.g. a targeted therapy only works on cell lines with the mutation it targets). Test that same drug against an unrelated cell line, and it does essentially nothing — hence AUC pinned near 1.0. With thousands of drug × cell-line combinations tested, a lot of them are just "wrong pairing, no effect," which piles up at the ceiling

Why this matters for your model:

It's not a smooth continuous distribution — you have a big mass at the ceiling plus a spread of everything else. This is sometimes called a censored or ceiling effect, and it can make plain regression harder, since the model has to both distinguish "no effect" from "some effect" and predict fine-grained values within the responsive range.
These aren't necessarily errors or outliers — don't be tempted to drop them, they're real biology (resistance).
Something to consider going forward: some pharmacogenomics workflows model this as two problems — a classification step ("does this drug have any effect on this cell line at all?") followed by regression only on the responsive subset — because a single regression model can struggle to fit both the flat ceiling mass and the meaningful variation together.

# things to check

- which drugs have more than 300–400 cell line samples (real, honest count — after merging response and expression data on cell line ID, not before)
- which drugs have the most response variance (std of AUC) — not drugs where nearly every cell line is resistant (AUC ~1.0), since there's nothing to predict there
- of those, which have the most overlapping cell lines in the TPM gene expression file (a drug can look well-tested in GDSC but lose most of its samples if those cell lines aren't in the expression data)
- prefer a targeted therapy with a known biomarker (e.g. an EGFR or BRAF inhibitor) over a broad chemo drug — makes it easier to sanity-check the model's top predictive genes against known biology
- check whether anyone has published results predicting response to that drug from expression data — gives a benchmark to compare against

```python
for drug in drug_counts.head(10).index:
    subset = dose_response[dose_response['DRUG_NAME'] == drug]['AUC']
    print(drug, "std:", subset.std(), "mean:", subset.mean())
```

for every drug listed in the GDSC file we are going to get a count of how many of those cell lines tested against exist in the rna_seq file

# Drug selected: Docetaxel

DRUG_ID = 1819
DRUG_NAME = Docetaxel

AUC is spread well

**note: there is another Docetaxel with the id of 1007 we are not working with that**

- think about combining them later?

# build_dataset

transposing sample to keep with ML consistency of rows=samples, columns=features

dropped 397 genes since consistently missing from 4724 occurances
