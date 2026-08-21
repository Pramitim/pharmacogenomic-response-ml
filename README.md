# ML-Based Cancer Drug Sensitivity Prediction

This project investigates whether increasingly complex machine-learning models can predict **cancer-cell drug sensitivity from gene-expression profiles**.

The project uses publicly available pharmacogenomic data to build a regression task where:

- **Input (`X`)** = gene-expression profile of a cancer cell line
- **Target (`y`)** = that cell line's response to one specific drug
- **Observation** = one cancer cell line tested against the selected drug

The models are compared as an increasing progression in complexity:

**Linear Regression → Random Forest → PyTorch MLP**

The goal is not to assume that more complex models will perform better, but to test whether they actually improve generalization on unseen cell lines.

---

## Research Question & Hypothesis

### Research Question

> **Can increasingly nonlinear models predict sensitivity to a particular cancer drug from gene-expression data better than linear regression?**

### Hypothesis

Nonlinear models may achieve better predictive performance than linear regression because drug response can depend on **nonlinear relationships and interactions among genes**.

However, the opposite result would also be meaningful. If a simpler linear model performs as well as or better than the nonlinear models, this would suggest that additional model complexity does not necessarily improve prediction for this dataset.

---

# Project Structure

```text
project/
│
├── data/
│   └── processed/
│       ├── rnaseq_merged_rsem_tpm_20260323.csv
│       └── GDSC2_fitted_dose_response_27Oct23.csv
│
├── notebooks/
│   ├── 01_data_exploration.ipynb
│   ├── 02_build_dataset.ipynb
│   └── 03_modeling.ipynb
│
└── README.md
```

### Notebooks

#### `01_data_exploration.ipynb`

Used for understanding and exploring the datasets, performing initial cleaning, examining drug-response distributions, and selecting the drug used in the experiment.

This notebook records the selection of **Docetaxel (Drug ID = 1819)**.

#### `02_build_dataset.ipynb`

Used to:

- Match cell lines between the TPM and GDSC datasets
- Keep only cell lines present in both datasets
- Transpose the TPM data into an ML-compatible format
- Construct the final feature matrix `X`
- Construct the target vector `y`
- Handle missing data and dimensionality

#### `03_modeling.ipynb`

Used for the ML experiment:

- Train/test splitting
- Feature standardization
- Model training
- Model comparison
- Evaluation using regression metrics

The models are evaluated in increasing order of nonlinearity:

```text
Linear Regression
        ↓
Random Forest
        ↓
PyTorch MLP
```

---

# Data Sources

## 1. RNA-Seq TPM Gene Expression

`data/processed/rnaseq_merged_rsem_tpm_20260323.csv`

The RNA-seq dataset contains gene-expression measurements for cancer cell models.

- Each `SIDM...` identifier represents a specific cancer cell model.
- Each **column** represents a cell model.
- Each **row** represents a gene.
- Each gene/cell-model intersection contains the measured expression level in **TPM (Transcripts Per Million)**.

A value of `0` means the processed expression value is zero; it does \*\*not necessarily mean that the gene is completely absent from the cell.

Because the data is TPM-normalized, the expression values across all genes in a sample sum to approximately **1,000,000**.

---

## 2. GDSC Drug-Response Data

`data/processed/GDSC2_fitted_dose_response_27Oct23.csv`

This dataset contains drug-response measurements for cancer cell lines tested against different drugs.

The dataset contains response measurements including:

- **AUC**
- **LN_IC50**
- **RMSE**

For this project, **AUC** was selected as the target variable.

---

# Drug Selection

A single drug was selected rather than attempting to predict responses to multiple drugs.

This created a cleaner experimental question:

```text
Gene expression of Cell Line A
                ↓
          ML model
                ↓
Predicted response to Drug X
```

The main consideration was the **overlap between the GDSC response data and the TPM expression data**.

A cell line was only considered a usable sample if it had:

1. A drug-response measurement for the selected drug
2. A corresponding gene-expression profile in the TPM dataset

## Selection Process

For every drug in the GDSC dataset, the number of cell lines with both drug-response data and TPM expression data was determined.

Candidate drugs were then evaluated based on:

### 1. Number of overlapping cell lines

Drugs with approximately **400+ usable cell lines** were prioritized.

The important number was not the total number of cell lines tested in GDSC, but the number remaining **after matching with the TPM dataset**.

### 2. Variation in AUC

The distribution and standard deviation of AUC values were examined.

Drugs with very little variation were avoided because there would be little response variation for the model to learn.

### 3. Biological relevance

Targeted therapies with known biomarkers or biological mechanisms were preferred where possible, as this provides a basis for interpreting important predictive genes.

### 4. Existing research

Existing studies using gene-expression data to predict response to candidate drugs were considered as potential benchmarks for the final model.

---

# Selected Drug: Docetaxel

The selected drug was:

```text
Drug name: Docetaxel
Drug ID: 1819
```

Docetaxel was selected because it provided a suitable combination of **overlapping cell lines and AUC variation** for the ML experiment.

The AUC distribution showed sufficient spread to provide meaningful variation in the target variable.

> **Important:** GDSC contains another Docetaxel entry with **Drug ID = 1007**. This project uses **Docetaxel (ID = 1819)** only.

The two Docetaxel entries could potentially be investigated as a future extension, but they were kept separate for this experiment.

---

# Target Variable: AUC

**AUC (Area Under the Dose-Response Curve)** was used as the continuous target variable representing drug sensitivity.

AUC summarizes the entire dose-response curve rather than relying on a single measurement such as IC50.

For this project:

- **Lower AUC → greater drug sensitivity**
- **Higher AUC → greater drug resistance**
- AUC values are generally normalized to approximately **0–1**

AUC was preferred because it captures the overall response across the tested concentration range and does not require the dose-response curve to reach exactly 50% inhibition.

---

# Exploratory Data Analysis

Initial exploratory analysis was performed before constructing the ML dataset.

### Missing Data

`PUTATIVE_TARGET` contained **27,872 missing values**.

Several drug names appeared with multiple IDs in the GDSC dataset, including:

- Acetalax
- Dactinomycin
- Docetaxel
- Fulvestrant
- GSK343
- Oxaliplatin
- Selumetinib
- Ulixertinib
- Uprosertib

Therefore, **drug name alone was not used to identify a drug**. The corresponding `DRUG_ID` was used to distinguish entries.

### AUC Distribution

The overall AUC distribution showed a large spike around **1.0**.

This indicates that many drug–cell-line combinations showed little or no measurable drug effect, resulting in AUC values near the maximum.

This is biologically plausible. Many drugs are only effective against particular cancer types or cell lines with specific molecular vulnerabilities. When a drug is tested against a cell line without the relevant vulnerability, the response can remain close to the resistance ceiling.

This creates a **ceiling effect** in the response distribution.

These high-AUC observations were not automatically treated as errors or outliers because they represent genuine biological resistance.

For the selected Docetaxel dataset, the AUC distribution had sufficient variation to proceed with regression modeling.

---

# Building the ML Dataset

The raw datasets were structured differently, so they first had to be transformed into a common ML format.

## Step 1: Select the Drug

```text
Docetaxel
Drug ID = 1819
```

↓

## Step 2: Identify Cell Lines Tested With the Drug

All GDSC cell lines with a response measurement for Docetaxel were identified.

↓

## Step 3: Match Cell Lines With the TPM Dataset

The corresponding cell-line identifiers were matched against the TPM expression dataset.

↓

## Step 4: Keep Only Overlapping Cell Lines

Only cell lines present in **both datasets** were retained.

This produced the actual set of usable samples.

↓

## Step 5: Transpose the TPM Data

The original TPM data was structured approximately as:

```text
             Cell A   Cell B   Cell C
Gene 1         ...      ...      ...
Gene 2         ...      ...      ...
Gene 3         ...      ...      ...
```

For ML, the data was transposed to:

```text
             Gene 1   Gene 2   Gene 3   ...   Gene N
Cell A         ...      ...      ...            ...
Cell B         ...      ...      ...            ...
Cell C         ...      ...      ...            ...
```

This follows the standard ML structure:

> **Rows = samples, columns = features**

↓

## Step 6: Build `X` and `y`

The final dataset was structured as:

```text
                 Gene 1   Gene 2   Gene 3   ...   Gene N
Cell Line A        ...      ...      ...            ...
Cell Line B        ...      ...      ...            ...
Cell Line C        ...      ...      ...            ...
```

This becomes:

```text
X = gene-expression features
```

The corresponding GDSC responses were:

```text
Cell Line A → AUC
Cell Line B → AUC
Cell Line C → AUC
```

This becomes:

```text
y = AUC drug-response target
```

Therefore:

```text
X: Gene-expression profile
          ↓
      ML model
          ↓
y: Predicted Docetaxel AUC
```

---

# Handling Dimensionality

Gene-expression data contains a very large number of potential features relative to the number of cell-line samples.

This creates a high-dimensional learning problem:

```text
Hundreds of cell lines
        ×
Thousands of genes
```

Having many more features than observations increases the risk of **overfitting**.

During dataset construction, genes with consistently missing measurements were removed. In the current preprocessing, **397 genes were removed** because they were consistently missing across the relevant observations.

Dimensionality handling is therefore an important consideration when interpreting model performance.

---

# ML Pipeline

The complete modeling pipeline follows:

```text
                    Gene Expression
                           │
                           ▼
                  Matched Cell Lines
                           │
                           ▼
                     Final X / y
                           │
                           ▼
                  Train / Test Split
                           │
                           ▼
                    Standardization
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
       Linear Regression  Random Forest  PyTorch MLP
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                    Model Evaluation
                           │
                           ▼
                  Compare Performance
```

## Train/Test Split

The dataset is divided into training and test sets.

```text
All cell lines
      │
      ├──────────────┐
      ▼              ▼
   Training         Test
     data           data
      │              │
      ▼              │
 Train models        │
                     │
                     ▼
                Final evaluation
```

The test set is kept separate from model training and preprocessing decisions so that it represents genuinely unseen cell lines.

This is important for determining whether the model can **generalize to cell lines it has not seen before**.

---

# Standardization

Gene-expression features can have very different numerical ranges.

For example:

```text
Gene A → 0–5
Gene B → 0–1,000
Gene C → 0–20
```

Standardization puts features onto comparable scales.

Importantly, the standardization parameters are learned from the **training data only** and then applied to the test data.

This prevents information from the test set from leaking into the training process.

---

# Model Comparison

The models represent an increasing level of flexibility.

## 1. Linear Regression

**Question:**

> Can a linear relationship between gene expression and drug response explain the observed variation in AUC?

This provides the baseline.

---

## 2. Random Forest

**Question:**

> Does allowing nonlinear relationships and interactions between features improve prediction?

Random Forest provides a nonlinear tree-based comparison to the linear baseline.

---

## 3. PyTorch MLP

**Question:**

> Can a neural network learn nonlinear patterns in gene-expression data that the simpler models cannot capture?

The MLP provides the deep-learning component of the experiment.

---

### Why Compare These Models?

The goal is not simply to find the most complicated model.

Instead, the models allow the following question to be tested:

```text
Does increasing model complexity
            ↓
capture more useful biological patterns
            ↓
and improve prediction on unseen cell lines?
```

A more complex model performing worse would be an informative result rather than a failure.

---

# Model Evaluation

Because the task is **regression**, model performance is evaluated using:

### RMSE

**Root Mean Squared Error**

Measures the average magnitude of prediction errors while giving greater weight to larger errors.

**Lower is better.**

### MAE

**Mean Absolute Error**

Measures the average absolute difference between predicted and actual AUC.

**Lower is better.**

### R²

**Coefficient of Determination**

Measures how much of the variation in the target is explained by the model.

**Higher is better.**

The final comparison will focus on whether the nonlinear models actually improve **generalization performance** compared with the linear regression baseline.

---

# Key Experimental Considerations

## High Dimensionality

The number of gene-expression features can be much larger than the number of cell-line samples.

This creates a substantial risk of overfitting and makes feature selection, dimensionality handling, and regularization important considerations.

## Data Leakage

Preprocessing steps such as standardization must be fitted using training data only.

The test set should remain untouched until final evaluation.

## Ceiling Effect in AUC

The large number of AUC values near 1.0 represents substantial drug resistance in the broader GDSC dataset.

This can make regression more difficult because the model must distinguish between:

```text
No/very little drug response
          vs.
Meaningful drug response
          vs.
Different levels of sensitivity
```

A potential future extension would be to investigate a **two-stage approach**:

```text
Stage 1: Does the drug have a meaningful effect?
                  ↓
Stage 2: How strong is the response?
```

However, the current project focuses on the simpler continuous AUC regression problem.

---

# Experimental Progression

The project follows a deliberately simple progression:

```text
1. Understand and explore the data
             ↓
2. Select a biologically and statistically suitable drug
             ↓
3. Match drug-response and gene-expression data
             ↓
4. Build a clean X/y dataset
             ↓
5. Establish a Linear Regression baseline
             ↓
6. Test Random Forest
             ↓
7. Test PyTorch MLP
             ↓
8. Compare generalization performance
             ↓
9. Interpret whether increased nonlinearity improved prediction
```

The central comparison is therefore:

> **Does increasing model complexity provide a meaningful improvement in predicting cancer-cell response to Docetaxel from gene-expression data?**

---

# Future Extensions

If the initial experiment produces meaningful results, potential extensions include:

- Comparing additional drugs
- Investigating the second Docetaxel entry (`DRUG_ID = 1007`)
- Adding XGBoost as another nonlinear baseline
- Testing different dimensionality-reduction or feature-selection approaches
- Investigating which genes contribute most strongly to predictions
- Comparing model performance across different drugs or cancer types
- Exploring a two-stage classification + regression approach for the AUC ceiling effect
- Comparing model-selected features with known biological pathways and drug-response biomarkers

These extensions would allow the project to move from a single-drug proof of concept toward a broader pharmacogenomic ML analysis.
