# PhishTackle

**ML-powered phishing URL detector with a desktop GUI.**

PhishTackle trains a class-balanced Random Forest on 11,055 labelled URL records and exposes it through a small wxPython app: paste a URL, click **Scan URL**, and get a *Legitimate* or *Phishing* prediction in real time.

> This is a machine-learning prediction based on URL structure, not a guarantee of safety. See [Limitations](#limitations).

---

## Highlights

- Class-balanced **Random Forest** trained on **11,055** labelled records with **10 binary features**
- Hyperparameters tuned with a **3-fold grid search** (optimising phishing-class F1)
- **90.5% accuracy** and **0.952 ROC-AUC** on a stratified 20% holdout (2,211 rows)
- Top predictors by feature importance: **SSL state (~70%)**, **subdomain count (~11%)**, **prefix/suffix in the domain (~10.5%)**
- **wxPython** desktop app that validates a URL and extracts **8 of the 10 features** at prediction time
- URL validation via the Public Suffix List (`tldextract`), so incomplete hostnames such as `https://google` are rejected rather than scored

## Results

Evaluated once on a stratified 20% holdout (`random_state=42`; 8,844 train / 2,211 test).

| Metric | Value |
|---|---|
| Accuracy | 90.5% |
| ROC-AUC | 0.952 |
| Phishing precision / recall / F1 | 0.909 / 0.871 / 0.890 |
| Legitimate precision / recall / F1 | 0.901 / 0.931 / 0.916 |

Confusion matrix (rows = actual, columns = predicted):

|  | Pred. legitimate | Pred. phishing |
|---|---|---|
| **Actual legitimate** | 1,146 | 85 |
| **Actual phishing** | 126 | 854 |

Best hyperparameters from the grid search: `n_estimators=200`, `max_depth=10`, `min_samples_split=2`, `min_samples_leaf=1` (best 3-fold CV phishing F1 = 0.885).

### Feature importance

| Feature | Importance |
|---|---|
| `SSLfinal_State` | 70.5% |
| `Having_Sub_Domain` | 11.1% |
| `Prefix_Suffix` | 10.5% |
| `Domain_Age` | 2.8% |
| `Having_IP` | 1.3% |
| `URL_Length` | 1.1% |
| `Shortening_Service` | 1.1% |
| `Double_Slash_Redirecting` | 0.8% |
| `Having_At_Symbol` | 0.8% |
| `Domain_End` | 0.0% (constant in the training data) |

## Features

The model uses 10 binary features, in this exact order:

| # | Feature | How the app computes it from a URL |
|---|---|---|
| 1 | `Having_IP` | Hostname is an IP address |
| 2 | `URL_Length` | URL longer than 54 characters |
| 3 | `Shortening_Service` | Host is a known shortener (bit.ly, tinyurl.com, t.co, goo.gl, ow.ly, is.gd, buff.ly, adf.ly, bit.do, cutt.ly) |
| 4 | `Having_At_Symbol` | URL contains `@` |
| 5 | `Double_Slash_Redirecting` | A second `//` appears after the scheme |
| 6 | `Prefix_Suffix` | Hostname contains `-` |
| 7 | `Having_Sub_Domain` | More than two host labels (ignoring a leading `www`) |
| 8 | `SSLfinal_State` | URL scheme is `https` (**scheme check only; the certificate is not validated**) |
| 9 | `Domain_Age` | Not available from a URL alone; needs WHOIS data. Fixed at 0 |
| 10 | `Domain_End` | Not available from a URL alone; needs WHOIS data. Fixed at 0 |

Features 1–8 are extracted live; features 9–10 require WHOIS/domain-registration data and are set to 0 in the GUI.

## Project structure

```
.
├── phistackle_url_validation.ipynb   # data loading, tuning, evaluation, feature extraction, GUI
├── training.xlsx                     # labelled training data (11,055 rows)
├── feature_train.csv                 # 1,200-row feature file
├── feature_test.csv                  # 400-row feature file (separate domain-shift check)
├── test1.xlsx                        # additional test data
└── requirements.txt
```

## Getting started

**Requirements:** Python 3.9+ and a desktop environment (the GUI uses wxPython).

```bash
pip install -r requirements.txt
jupyter notebook phistackle_url_validation.ipynb
```

Keep the notebook and the data files in the same folder, then run all cells. The final cell trains and tunes the model, shows the confusion matrix and ROC curve, and then opens the PhishTackle window.

Dependencies: `pandas`, `numpy`, `scikit-learn`, `matplotlib`, `seaborn`, `openpyxl`, `wxPython`, `tldextract`.

> On Linux, `wxPython` may need a prebuilt wheel for your distro; see the [wxPython downloads page](https://extras.wxpython.org/wxPython4/extras/linux/gtk3/).

## How it works

1. **Load and split.** `training.xlsx` is split 80/20, stratified by label.
2. **Tune.** `GridSearchCV` (3-fold, scoring = F1 on the phishing class) searches `n_estimators` ∈ {200, 300}, `max_depth` ∈ {10, None}, `min_samples_split` ∈ {2, 5}, using `class_weight="balanced"`.
3. **Evaluate.** The best model is scored once on the holdout set (accuracy, classification report, confusion matrix, ROC-AUC).
4. **Predict.** The GUI validates the URL's domain structure, extracts the features, and runs the same trained model.

## Limitations

- **Duplicate-heavy dataset.** `training.xlsx` contains only 331 unique rows (261 unique feature vectors) among its 11,055 records, so many holdout rows have an identical twin in the training set. The 90.5% figure is therefore optimistic: it is close to the ceiling achievable by these 10 binary features on this data (about 90.4% by majority label per feature vector). When the split is made so that no feature vector appears in both train and test, accuracy drops to roughly 88% on average (range ≈ 83–92% over 10 random splits, fixed hyperparameters).
- **Two features are inert in the app.** `Domain_Age` and `Domain_End` are fixed at 0, and `Domain_End` is constant in the training data.
- **SSL state is only the URL scheme.** An `https://` URL scores as "SSL present"; certificates are not checked. Since this is the dominant feature (~70%), the prediction leans heavily on whether the URL starts with `https`, which many phishing sites also use.
- **Structure-only validation.** The domain check confirms a recognised public suffix, not that a domain is registered, reachable, or safe.
- **`feature_test.csv` is not used for model selection.** Its feature distribution differs from `training.xlsx` (e.g. `Domain_End` varies there but is constant in training).
- **Known issue: feature encoding.** In the training data, the value `1` appears to mean the *benign* state for most structural features (e.g. every row with `Prefix_Suffix = 1` is labelled legitimate), whereas the GUI sets these features to `1` when the suspicious pattern is *present*. This mismatch likely distorts predictions for those features and should be reconciled before relying on the app.

## Roadmap

- Fix the feature-encoding mismatch between the dataset and the GUI extractor
- Add WHOIS lookup for domain age/expiry and real SSL certificate validation
- Deduplicate the dataset and re-evaluate with a leakage-free split
- Evaluate on a fresh, independently collected URL set

## Tech stack

Python · scikit-learn · pandas · NumPy · wxPython · tldextract · matplotlib · seaborn
