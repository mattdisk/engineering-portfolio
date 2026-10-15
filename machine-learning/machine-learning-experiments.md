# Machine Learning Experiments

Empirical studies in supervised learning, unsupervised learning, and reinforcement learning. Each study compares model families under controlled conditions and examines why they behave as they do:
- classifiers on imbalanced tabular data;
- clustering and dimensionality reduction against known labels;
- planning vs model-free control on discrete and discretized environments.

The recurring themes are leakage-aware preprocessing, metric selection, hyperparameter sweeps that change one factor at a time, working within compute budgets, and interpreting results that were unexpected or negative. The algorithms come from standard libraries: scikit-learn, imbalanced-learn, PyTorch, Keras, and bettermdptools. The experiment harnesses, PyTorch training loops, custom metric calculations, and analysis were built around them.

> **Source access:** The implementation code, experimental notebooks, and original technical reports are maintained in a private repository and are available to reviewers upon request.

## Experimental Areas

| Area | Task | Methods | Evaluation focus | Main themes |
|---|---|---|---|---|
| **Supervised learning** | Binary classification on two tabular datasets: corporate bankruptcy (3.2% positive) and synthetic cancer stage | KNN, SVM (RBF and linear), multilayer perceptrons (Keras and PyTorch) | F1 and ROC-AUC under class imbalance; learning, validation, and per-epoch curves | Resampling without leakage, metric choice, compute-budgeted sweeps, a near-chance result |
| **Unsupervised learning** | Structure discovery on the same two datasets | K-Means, Gaussian Mixture Models; PCA, ICA, Gaussian random projection, LLE; t-SNE for visualization | Silhouette, BIC/AIC, posterior entropy, Adjusted Rand Index; downstream MLP F1 | Cluster compactness vs label alignment; comparing feature transformations |
| **Reinforcement learning** | Blackjack-v1 (discrete, stochastic) and CartPole-v1 (continuous, discretized) | Value iteration, policy iteration, SARSA, tabular Q-learning | Planning time, convergence, mean episode return / steps balanced | Granularity of the state representation; learning-rate, exploration, and discount schedules |

The supervised and unsupervised studies share the same two datasets, so they form one tabular track. Dataset characteristics found in one carry over to the other. The reinforcement learning studies are separate, but they follow the same approach: change one factor at a time and measure the cost as well as the outcome.

## Key Findings

- **Supervised: resampling and metric choice both change the conclusions.** On the imbalanced bankruptcy data, wherever SMOTE was used in cross-validation (the SVM sweeps), it ran inside the training folds, so synthetic samples never reached the validation folds. F1 and ROC-AUC favored different hyperparameters for both KNN and SVM.
- **Unsupervised: tight clusters were not the same as label-aligned clusters.** On the bankruptcy data, PCA gave by far the most compact K-Means clustering (silhouette 0.847), but it agreed poorly with the bankruptcy label (ARI 0.106).
- **Reinforcement learning: discretization pushed planning and learning in opposite directions.** On CartPole, finer state grids improved the recorded value and policy iteration results but increased planning time sharply. Under a fixed training budget, the same finer grids reduced SARSA performance. State representation, compute cost, and sample efficiency interact.

## Experimental Design and Findings

All numbers come from saved notebook outputs and figures from the original runs; they were not re-run for this summary. Most experiments used a single random seed, so small differences shouldn't be read as rankings.

### Tabular Datasets

- **Taiwanese bankruptcy data.** 6,819 companies with 95 numeric financial-ratio features and a binary `Bankrupt?` label; 220 companies (3.2%) are positive. Data from the Taiwan Economic Journal, 1999–2009 (see References).
- **Synthetic cancer patient data.** 50,000 synthetic patient records (2015–2024) with demographic, lifestyle, cost, and survival fields. These are **generated data, not real patient records**. For classification, `Cancer_Stage` was binarized as Stage III–IV vs Stage 0–II, about 40/60. Categorical fields were one-hot encoded.

Exploratory analysis showed the datasets behave very differently. The bankruptcy features form strongly correlated blocks of related financial ratios. The cancer features show near-zero pairwise correlation, and t-SNE shows the stage classes fully intermixed.

### Supervised Learning

**Splits and resampling.** All experiments used a stratified 80/20 train/test split, with standardization fitted on training data only. SMOTE was applied to the bankruptcy data at specific stages, not uniformly:

| Bankruptcy experiment | Resampling |
|---|---|
| KNN learning curves and k sweep; SVM (RBF) learning curve; first NN learning curve | None |
| SVM C sweeps, RBF and linear (5-fold CV) | SMOTE inside an `imblearn` pipeline with the scaler and SVM, so resampling was confined to each training fold |
| NN activation, width, and epoch studies (single 80/20 split of a 4,000-row subsample) | SMOTE on the training portion only; validation data left untouched |
| Final KNN, SVM, and NN models | SMOTE applied to the full training split (inside an `imblearn` pipeline for KNN and SVM); evaluated on the untouched test split |

The KNN k was therefore chosen from curves without resampling and then used in a SMOTE pipeline. No resampling was used for the cancer data.

**Metric choice.** Accuracy isn't informative at 3.2% prevalence. The final bankruptcy SVM had 0.920 test accuracy, below the 0.968 majority-class rate, yet it was the strongest model by F1 and ROC-AUC. ROC-AUC measures how well the model's scores rank positives above negatives across all decision thresholds. F1 measures the predictions at the classifier's actual decision threshold. The two diverged in the sweeps:
- **KNN:** validation F1 peaked at k = 3, with k = 5 close behind, and fell toward zero for large k. Validation ROC-AUC kept improving, to about 0.89 near k ≈ 45. Better ranking across thresholds did not mean better predictions at the threshold actually used. k = 5, near the observed F1 peak, was used for the final model.
- **SVM (RBF):** validation F1 plateaued around C = 1–10, while validation ROC-AUC was highest at C ≤ 0.1 and fell for larger C. C = 1 was used for the final model.

**Neural networks.**
- Keras models on the bankruptcy data: a 6-way activation × initializer comparison (ReLU / LeakyReLU / tanh × He / Xavier) with early stopping, a hidden-width sweep, and per-epoch train/validation tracking.
- Hand-written PyTorch training loops on the cancer data: `BCEWithLogitsLoss`, explicit Kaiming/Xavier initialization, and the same activation, width, and epoch studies.

**Compute-budgeted SVM experiments.** On 40,000 cancer training rows, sweeping kernel SVMs with probability calibration was too expensive. Three measures kept it tractable:
- The learning-curve loop has a 300 s wall-clock budget and stops early when validation AUC plateaus. It stopped after 4 of 8 training fractions.
- The C sweeps ran on a 20-component TruncatedSVD projection of a 20% subsample.
- ROC-AUC was computed from `decision_function` scores instead of `probability=True`, which avoids the extra internal cross-validation that Platt scaling requires.

**Recorded test results** (F1 for the positive class):

| Dataset (test set) | Model | F1 | ROC-AUC |
|---|---|---|---|
| Bankruptcy (1,364 rows, 44 positive) | KNN, k = 5, SMOTE | 0.282 | 0.844 |
| | SVM, RBF, C = 1, SMOTE | **0.331** | **0.919** |
| Cancer (10,000 rows) | KNN, k = 5 | 0.352 | 0.496 |
| | SVM, RBF, C = 10 | 0.351 | 0.500 |
| | MLP, [256, 256] | 0.364 | 0.492 |

With only 44 positive test companies, the bankruptcy F1 values are noisy estimates. The bankruptcy neural network is left out of the table. Its final evaluation code passes already-standardized test features through a wrapper that standardizes them again, so the recorded score doesn't reflect the model as trained.

**The cancer task.** Every model family stayed at chance-level ROC-AUC, consistent with the flat correlations and mixed t-SNE seen in exploration. This means no signal was recovered by these pipelines, not that the data contains none. Four things limit the conclusion:
- the models were limited to distance-, margin-, and MLP-based classifiers, with no feature engineering;
- the PyTorch architecture comparisons used the test split as their validation set;
- the final SVM was trained on a 20% subsample;
- the pipeline kept the dataset's `Target_Severity_Score` field as a feature, which the unsupervised pipeline drops.

### Unsupervised Learning

**Setup.**
- For bankruptcy, the majority class was undersampled to 1:5 (220 positive, 1,100 negative), then split 80/20 and standardized on the training split.
- Cancer clustering used 500–2,000-row samples because of runtime.
- Labels were not used to fit the clustering or reduction models. They were used only for undersampling, stratified splitting, external evaluation (ARI), and coloring visualizations.

**Choosing dimensions.** Each reduction method was swept over component counts using a criterion suited to it:
- PCA: explained variance.
- ICA: mean absolute excess kurtosis of the components.
- Random projection: reconstruction MSE through the pseudo-inverse of the projection matrix, computed in the notebook.
- LLE: trustworthiness.

The selected dimensions were PCA 6, ICA 13, RP 30, and LLE 25.

**Clustering quality vs label alignment.** K-Means (K = 2–10) and GMMs were rerun on each representation of the undersampled bankruptcy training split (about 1,056 rows). GMM posterior entropy and BIC were also recorded for every configuration.

| Representation | K-Means best silhouette | K-Means best ARI | GMM best ARI | MLP macro F1 (5 runs) |
|---|---|---|---|---|
| Original (95 features) | 0.129 | 0.260 | **0.330** | 0.753 |
| PCA (6) | **0.847** | 0.106 | 0.159 | 0.730 |
| ICA (13) | 0.548 | 0.022 | 0.152 | **0.786** |
| Random projection (30) | 0.196 | **0.407** | 0.188 | 0.739 |
| LLE (25) | 0.400 | 0.131 | 0.095 | 0.455 |

PCA produced by far the tightest clustering but low agreement with the bankruptcy label. A data-independent random projection gave the highest K-Means ARI, and GMMs on the raw features aligned with the label better than on any reduced representation. "Best ARI" uses the labels to pick K, so it is an optimistic diagnostic, not a model-selection procedure.

**Feature transformations for downstream classification.** A fixed scikit-learn MLP ([256, 256], ReLU, early stopping) was trained on each representation, 5 runs each, and scored by macro F1 on the 264-row undersampled test split:
- ICA features scored slightly above the original features (0.786 vs 0.753), a small margin relative to the test size.
- PCA and RP were slightly lower but had lower run-to-run variance.
- LLE collapsed to 0.455 with zero variance, consistent with an embedding that doesn't preserve the separating structure.

These macro-F1 scores come from a rebalanced test split and are **not comparable** to the supervised bankruptcy F1 values.

**Cancer data.** K-Means silhouette on the raw features never exceeded about 0.001 for K = 2–10. On PCA and RP representations, ARI against the stage labels stayed at about 0. The GMM comparison on reduced cancer representations didn't finish (the notebook cell ended in an error), so no results from it are reported.

### Reinforcement Learning

Algorithms, environment wrappers, and the evaluation harness come from `bettermdptools` on Gymnasium. The experiment code adds:
- sweep functions that override one environment or algorithm parameter at a time;
- wall-clock timing;
- per-run `.npz` result archives, freed from memory between runs so long sweeps fit in RAM, then reloaded for plotting.

**Blackjack: planning vs model-free learning.**
- Value iteration and policy iteration converged to nearly identical state-value maps within a handful of iterations.
- SARSA was trained for 200,000 episodes under six schedules, each varying one of: ε-decay, α-decay, or the discount factor.
- Mean test returns over 1,000 hands were VI −0.063, PI −0.045, and SARSA between −0.074 and −0.025. All evaluated policies produced negative mean returns under these conditions.
- Differences of this size are comparable to the sampling error of a 1,000-hand estimate (roughly ±0.03), so no ranking of the methods is drawn from them.

**CartPole: state-space granularity.** CartPole's continuous state was discretized with 5, 10, or 20 bins per variable for position, velocity, and angular velocity. The wrapper's angle-binning parameter was held fixed.

| Bins per variable | VI / PI planning time | VI / PI mean steps (100 episodes) | SARSA mean steps (20k training episodes, 1,000 test) |
|---|---|---|---|
| 5 | 0.11 s / 0.15 s | 16.8 / 15.5 | **344.5** |
| 10 | 1.48 s / 2.20 s | 30.8 / 27.7 | 333.4 |
| 20 | 23.6 s / 46.3 s | **86.9 / 83.9** | 301.0 |

- **Planning** improved with finer grids, at a steeply growing computational cost. Policy iteration needed fewer iterations than value iteration but more wall-clock time, because each iteration includes full policy evaluation.
- **SARSA**, with a fixed budget of 20,000 episodes, did best on the coarsest grid, and its learned Q-values grew much faster there. This fits a sample-efficiency explanation: a finer grid spreads the same experience over many more rarely visited states.

This is an observation under these training conditions, not a general property of either method. The model-free SARSA policies also balanced the pole far longer than the planners on the same grids. That suggests the transition model the planner builds from the discretized environment is a coarse approximation of the true dynamics, though this interpretation was not tested separately.

**CartPole: learning-rate, exploration, and discount schedules.** On the coarsest grid, each variant changed one parameter from the baseline (γ = 0.95, α and ε annealed from 0.8 to 0.01):

| Variant | Change | Mean steps balanced |
|---|---|---|
| Fast α decay | α₀ = 1.0, anneal over first 30% of training | **452.5** |
| Long horizon | γ = 0.99 | 347.4 |
| Baseline | — | 344.5 |
| Slow α decay | α₀ = 1.0, anneal over first 90% | 322.8 |
| Slow ε decay | ε₀ = 1.0, anneal over first 90% | 193.0 |
| Fast ε decay | ε₀ = 1.0, anneal over first 30% | 163.7 |
| Short horizon | γ = 0.90 | 154.7 |

The same sweep on the 20-bin grid ranked the top variants differently: the long horizon came first there (303.5), ahead of fast α decay (244.6). The bottom of the ranking was the same on both grids: short horizons and fast exploration decay. With single-seed runs, that qualitative pattern is better supported than any exact ordering.

Tabular Q-learning was also trained on the coarse grid for 20,000 episodes. Only its learning curve was retained: smoothed reward climbed to roughly 250–300.

## Methodology and Limitations

Each study started from expected behavior, drawn from exploratory analysis or theory, then varied one factor while holding the others fixed. Several results contradicted those expectations, most notably the SARSA discretization result, and they are reported as observed. Some things limit how far the results can be pushed:

- **Sample size and seeds.** Most runs used a single fixed seed, and a few subsamples were unseeded. The bankruptcy test set has 44 positive cases, and the unsupervised comparisons use about 1,000 rows. Small differences are not treated as meaningful.
- **Evaluation conditions differ between studies.** The supervised study uses SMOTE and a natural-distribution test set; the unsupervised study uses undersampling and a rebalanced test set. F1 is binary in one and macro-averaged in the other. CartPole planners and SARSA were evaluated over 100 and 1,000 episodes respectively. Results are compared only within matching conditions.
- **Known evaluation issues are disclosed, not corrected.** The original code is preserved as it ran. The bankruptcy neural-network double standardization and the cancer-pipeline issues (test-split validation, the retained `Target_Severity_Score`) are described next to the affected results.
- **Interactive workflow.** The experiments were developed in notebooks with per-area environment snapshots (Python 3.11). A few cells depend on earlier session state.

## Private Source Organization

The following directories exist only in the private repository and are **not** part of this README-only public repository.

```text
machine-learning-experiments/
├── supervised-learning/
│   ├── data/                # bankruptcy + synthetic cancer CSVs
│   ├── models/              # empty; notebooks write trained models here when re-run
│   ├── notebooks/           # EDA, KNN, SVM, NN (one per dataset)
│   ├── plots/               # EDA, learning/validation/epoch curves
│   └── requirements.txt
├── unsupervised-learning/
│   ├── data/                # same two CSVs
│   ├── notebooks/           # EDA, clustering + dimensionality reduction analyses
│   ├── plots/               # clustering, reduction metrics, t-SNE projections
│   └── requirements.txt
├── reinforcement-learning/
│   ├── notebooks/           # Blackjack VI/PI and SARSA; CartPole VI/PI, SARSA, Q-learning
│   ├── plots/               # convergence, value maps, learning and evaluation curves
│   └── requirements.txt
├── reports/
│   ├── supervised-learning.pdf
│   ├── unsupervised-learning.pdf
│   └── reinforcement-learning.pdf
├── README.md
└── .gitignore
```

## Technical Stack

Python 3.11 · NumPy · pandas · scikit-learn · imbalanced-learn · PyTorch · TensorFlow / Keras · SciPy · Gymnasium · bettermdptools · Matplotlib / seaborn · Jupyter

## References and Attributions

**Datasets**
- Taiwanese Bankruptcy Prediction. UCI Machine Learning Repository, https://archive.ics.uci.edu/dataset/572/taiwanese+bankruptcy+prediction. Data from the Taiwan Economic Journal, 1999–2009. Associated study: D. Liang, C.-C. Lu, C.-F. Tsai, G.-A. Shih, "Financial ratios and corporate governance indicators in bankruptcy prediction: A comprehensive study," *European Journal of Operational Research* 252(2), 2016. The UCI listing states a CC BY 4.0 license. The local copy matches that dataset's structure; the exact mirror it was downloaded from was not recorded.
- Global cancer patients, 2015–2024. A publicly distributed **synthetic** dataset (`global_cancer_patients_2015_2024.csv`, 50,000 records). Its original publisher and license were not verified for this summary. Neither dataset was created as part of this work.

**Libraries**
- scikit-learn: F. Pedregosa et al., "Scikit-learn: Machine Learning in Python," *JMLR* 12, 2011. https://scikit-learn.org
- imbalanced-learn: G. Lemaître, F. Nogueira, C. K. Aridas, *JMLR* 18(17), 2017. https://imbalanced-learn.org
- PyTorch: https://pytorch.org · TensorFlow / Keras: https://www.tensorflow.org, https://keras.io
- Gymnasium: M. Towers et al., "Gymnasium: A Standard Interface for Reinforcement Learning Environments," arXiv:2407.17032, 2024. https://github.com/Farama-Foundation/Gymnasium
- bettermdptools: https://github.com/jlm429/bettermdptools

**Methods**
- N. V. Chawla, K. W. Bowyer, L. O. Hall, W. P. Kegelmeyer, "SMOTE: Synthetic Minority Over-sampling Technique," *JAIR* 16, 2002.
- R. S. Sutton, A. G. Barto, *Reinforcement Learning: An Introduction*, 2nd ed., MIT Press, 2018. http://incompleteideas.net/book/the-book-2nd.html
- G. A. Rummery, M. Niranjan, "On-line Q-learning using connectionist systems," Cambridge University Engineering Department, Tech. Rep. CUED/F-INFENG/TR 166, 1994 (SARSA).
- C. J. C. H. Watkins, P. Dayan, "Q-learning," *Machine Learning* 8, 1992.
- A. Hyvärinen, E. Oja, "Independent component analysis: algorithms and applications," *Neural Networks* 13(4–5), 2000.
- E. Bingham, H. Mannila, "Random projection in dimensionality reduction: applications to image and text data," *KDD*, 2001.
- S. T. Roweis, L. K. Saul, "Nonlinear dimensionality reduction by locally linear embedding," *Science* 290(5500), 2000.
- L. van der Maaten, G. Hinton, "Visualizing Data using t-SNE," *JMLR* 9, 2008.
- P. J. Rousseeuw, "Silhouettes: a graphical aid to the interpretation and validation of cluster analysis," *Journal of Computational and Applied Mathematics* 20, 1987.
- L. Hubert, P. Arabie, "Comparing partitions," *Journal of Classification* 2, 1985.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
