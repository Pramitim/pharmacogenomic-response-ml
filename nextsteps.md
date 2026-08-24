# Next Steps

## Current Status

The baseline modeling phase is complete.

- **Samples:** 651 cancer cell lines
- **Features:** ~36,000 gene-expression features
- **Target:** Docetaxel AUC
- **Models:** Linear Regression, Random Forest, PyTorch MLP

### Baseline Results

| Model             | Train R² | Test R² |  RMSE |   MAE |
| ----------------- | -------: | ------: | ----: | ----: |
| Linear Regression |    1.000 |   0.172 | 0.274 | 0.220 |
| Random Forest     |    0.897 |   0.148 | 0.278 | 0.240 |
| MLP               |    1.000 |  -1.545 | 0.480 | 0.358 |

The models showed substantial overfitting, particularly the MLP. Increasing model complexity did not improve test-set performance when using all ~36,000 genes directly.

---

## Next Steps — Essential Plan

### 1. Apply PCA

Reduce the ~36,000 gene-expression features into a smaller number of principal components.

Test:

- 50 components
- 100 components
- 250 components

**Important:** Fit PCA only on the training data to prevent data leakage.

### 2. Retrain the Models

Using the PCA-reduced features, retrain:

- Linear Regression
- Random Forest
- PyTorch MLP

Compare their performance against the original baseline.

### 3. Test Regularization

Run **Ridge Regression** to determine whether regularization improves the Linear Regression baseline and reduces overfitting.

- compare several alpha values
- looking for whether Ridge regression can lower the train R² while increasing the test R².

### 4. Evaluate

For every experiment, record:

- **RMSE** — lower is better
- **MAE** — lower is better
- **R²** — higher is better
- **Train vs. Test R²** — used to assess overfitting

### 5. Compare Results

Determine whether:

1. Reducing the number of features improves generalization.
2. Regularization improves performance.
3. Nonlinear models outperform the linear baseline after addressing the high-dimensionality problem.

# RESULTS

### PCA Results

PCA was evaluated using 50, 100, and 200 components. Reducing the number of components progressively decreased both training and test R².

The raw-feature Linear Regression model achieved a test R² of 0.172, while the 100-component PCA model achieved 0.156. Although PCA substantially reduced training performance and therefore reduced the model's ability to fit the training data, it did not improve test-set performance.

This suggests that the dimensionality reduction removed information that was useful for predicting Docetaxel response, rather than simply removing noise. PCA therefore did not improve the baseline Linear Regression model under the configurations tested.

50 components:
RMSE: 0.2825733751088381
MAE: 0.2324002782222643
Train R²: 0.40972123411738925
Test R²: 0.11729066399129029

100 components:
RMSE: 0.27947584520194013
MAE: 0.2312365687308631
Train R²: 0.5039227021437598
Test R²: 0.13653686866389914

200 components:
RMSE: 0.2751118952520807
MAE: 0.22884055719111468
Train R²: 0.6372996299104322
Test R²: 0.1632918879303885

therefore, reducing the dimensionality with PCA does not improve the linear model

### Ridge regression results

1. Overfitting/under-regularized:

- alpha between 0.01 and 1000
- Train R² ≈ 1.00
- Test R² ≈ 0.172

2. moderate regularization:

- alpha bettwen 5000-25000
- training R² decreases (model is not memorizing training data)
- test R² improves
- best R² and RMSE result is when alpha is 15000
- MAE is worsened

3. Underfitting/excessive regularization

- after alpha is 25,000
- model can not identify patterns

Moderate Ridge regularization improved test R² and RMSE relative to unregularized Linear Regression, while MAE increased. This suggests that regularization modestly improved overall variance explained and reduced larger prediction errors, but did not improve average absolute prediction error.

---
