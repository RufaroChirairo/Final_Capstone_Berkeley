# Predicting E. coli Contamination in River Water

My capstone project uses water-quality monitoring data from the UN GEMStat database to test whether routine water-quality measurements can predict *E. coli* exceedances in river water. Six classification models were compared, three were tuned with cross-validated grid search, and the best model was analyzed to see which measurements matter most.

* Author: Rufaro Chirairo
* Notebook: 

---

## Problem Statement

*E. coli* testing is slow, expensive, and requires lab work. This project asks: can routine physical and chemical water-quality measurements identify samples likely to exceed the *E. coli* threshold?

Samples are labeled as positive (`1`) when *E. coli* is ≥ 100 per 100 mL and negative (`0`) when it is below 100, a level the World Health Organization considers high or very high risk.

**Primary metric: recall.** A false negative leaves people exposed to unsafe water with no warning, so the models are judged mainly on how many true exceedances they catch. Precision, F1, accuracy, and ROC-AUC are reported alongside recall because recall alone can be inflated by flagging everything.

## Data

- Source: [GEMStat](https://gemstat.org/), the Global Freshwater Quality Database maintained by UNEP, downloaded in five batches due to download size limits.
- Geographic scope: river stations across Mexico.
- Raw size: 1,222,396 rows across five CSV files (semicolon-delimited, long format: one row per measurement).
- After removing `Poor`/`Suspect` quality rows and JCU turbidity: 1,135,919 rows.
- After reshaping and keeping only events with an *E. coli* measurement: 46,467 sampling events.
- After removing events missing more than 25% of their features: **45,760 sampling events** used for modeling.
- Features (18): Depth, BOD, COD, NH3N, NO2N, NO3N, O2-Dis, TDS, TEMP, TKN, TN, TOC, TON, TP, TRP, TSS, TURB, pH.
- Target: ECOLI, converted to a binary label. Class balance: 62% positive, 38% negative.

## Methodology

1. Combine the five data files into one dataframe.
2. Filter out rows with `Poor` or `Suspect` data quality ratings, and drop the 137 turbidity measurements reported in JCU rather than NTU (0.27% of turbidity rows).
3. Reshape (pivot) so each row is one sampling event (station, date, time, depth) with one column per parameter. Repeat measurements within an event are averaged.
4. Keep only events with an *E. coli* measurement.
5. Explore the data: mean vs. median comparisons, histograms, missingness, correlations, and the largest *E. coli*, FECALCOLI, turbidity, and TDS values (kept, since extreme values are not necessarily errors).
6. Clean: drop events missing more than 25% of their features, then drop identifier columns (station number, date, time).
7. Drop `FECALCOLI` (a near-duplicate of the *E. coli* signal) and `EC` (correlation of 1.00 with TDS, with more missing values).
8. Label the target using the 100 per 100 mL threshold.
9. Split 75% train / 25% test, stratified on the target. Preprocessing happens inside a Pipeline (median imputation, then `RobustScaler`) so nothing from the test set leaks into training.
10. Train six models with default settings, then tune three of them with `GridSearchCV` (3-fold cross-validation, scored on recall).

### Models compared

| Model | Tuned with GridSearchCV |
|---|---|
| Logistic Regression | No |
| Decision Tree | Yes |
| Random Forest | Yes |
| Gradient Boosting | Yes |
| Support Vector Classifier | No |
| K-Nearest Neighbors | No |

A fixed `random_state=42` is used for every model with built-in randomness so results are reproducible.

## Results

The baseline of always predicting "contaminated" gives **62.03% accuracy**. All results below are on the held-out test set.

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC |
|---|---|---|---|---|---|
| Logistic Regression | 0.714 | 0.705 | 0.927 | 0.801 | 0.745 |
| Decision Tree | 0.677 | 0.740 | 0.739 | 0.740 | 0.658 |
| Random Forest | 0.768 | 0.781 | 0.870 | 0.823 | 0.832 |
| Gradient Boosting | 0.741 | 0.750 | 0.874 | 0.807 | 0.796 |
| SVC | 0.700 | 0.725 | 0.832 | 0.775 | 0.741 |
| KNN | 0.717 | 0.752 | 0.810 | 0.780 | 0.753 |
| **Random Forest (tuned)** | **0.749** | **0.749** | **0.894** | **0.815** | **0.808** |
| Decision Tree (tuned) | 0.722 | 0.742 | 0.846 | 0.791 | 0.758 |
| Gradient Boosting (tuned) | 0.735 | 0.739 | 0.886 | 0.806 | 0.785 |

### Key takeaways

- Every model beat the 62.03% accuracy baseline, using only routine water-quality measurements and no fecal coliform data.
- The tuned Random Forest is the recommended model. It catches 89.4% of exceedances with 74.9% precision and a ROC-AUC of 0.808. Logistic Regression has the highest recall (0.927), but with the lowest precision and weaker AUC, so the tuned Random Forest offers a better balance while keeping recall close to the best.
- Tuning for recall raised recall for all three tuned models. For the Random Forest it came at a cost: precision fell from 0.781 to 0.749 and AUC from 0.832 to 0.808. The untuned Random Forest is the strongest overall ranker.
- Top features in the tuned Random Forest: Depth (0.203), TSS (0.084), TP (0.083), TRP (0.078), and O2-Dis (0.063). Depth's high importance likely reflects differences in sampling protocol between sites, since nearly all samples were taken at 0.3 m, so it should not be read as depth itself driving contamination.

## Limitations

- Samples from the same station can appear in both the training and test sets, which may inflate performance.
- All data come from Mexican river stations, so results may not transfer to other regions.
- Tuning on recall alone lowered precision and AUC.
- Missing values were filled with the median, which can blur real differences between sites.

## Future Work

- Split train/test by station so no station appears in both.
- Test on data from other regions or time periods.
