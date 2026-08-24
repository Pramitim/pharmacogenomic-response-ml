# Baseline Model Results

## Linear Regression:

**Predicted y test values vs. Actual y test values:**
RMSE: 0.27365430143375075
MAE: 0.2198595584503373
R²: 0.1721344702463099

**Train vs Test Data**
Train R²: 1.0
Test R²: 0.1721344702463099

## Random Forest

**Predicted y test values vs. Actual y test values:**
RMSE: 0.27764103428235126
MAE: 0.2399737479389313
R2: 0.14783724416792132

**Train vs Test Data**
Train R²: 0.897191065577738
Test R²: 0.14783724416792132

## MLP

**Predicted y test values vs. Actual y test values:**
RMSE: 0.4797615171728442
MAE: 0.35757038615724696
R²: -1.5445211232491758

**Train vs Test Data**
Train R²: 0.99954443208043
Test R²: -1.5445211232491758

## Findings

Increasing model complexity did not improve prediction performance when all ~36,000 gene-expression features were used directly. All three models showed evidence of overfitting, with the MLP showing the most severe failure to generalize.

Is the number of genes too large relative to the number of samples?
Is the model learning noise?
Would feature selection help?
Would dimensionality reduction help?
Would regularization improve Linear Regression?
Is there genuinely only a weak relationship between expression and Docetaxel response?
Does the AUC distribution make prediction difficult?
