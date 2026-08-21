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
R²: 0.897191065577738
R²: 0.14783724416792132

## Findings

Both models are overfitting heavily. The models are struggling to generalize from the available cell lines to unseen cell lines?"

Is the number of genes too large relative to the number of samples?
Is the model learning noise?
Would feature selection help?
Would dimensionality reduction help?
Would regularization improve Linear Regression?
Is there genuinely only a weak relationship between expression and Docetaxel response?
Does the AUC distribution make prediction difficult?
