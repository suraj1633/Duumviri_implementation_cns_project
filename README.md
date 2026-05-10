# DUUMVIRI

**Reimplemented by:**

- Riya Saran — 2023ucp1704
- Suraj Meena — 2023ucp1633

---

## Goal

To detect trackers on legitimate sites and block them.
It helps user protect their personal data of what they are browsing or the tasks they perform on the site being tracked.

---

## What did others miss ?

Ad blockers like try to block them, but they have two big problems:

1. They block too much → breaks websites (e.g., login stops working)
2. They miss trackers → some trackers are hidden inside scripts that also do useful things

---

## Duumviri's Idea ?

Duumviri uses two AI models that work together:

```
Request → [Tracker Oracle] + [Breakage Detector] → Final Decision
           "Is this tracking?"   "Will blocking break the site?"
```

- If Tracker Oracle says **YES** + Breakage Detector says **NO BREAKAGE** → Block it safely
- If blocking would break the site → Don't block (it's a "mixed" tracker)

---

## Implementation

### script1 — Replicating results on easylist/easy privacy (private data)

```bash
./replicate_filter_list_exp.sh
```

<!-- 📸 INSERT PHOTO HERE: Terminal screenshot showing script1 output with accuracy results (t-thr / f-thr / accu rows) -->
<img width="1439" height="813" alt="Screenshot 2026-05-10 at 2 07 43 PM" src="https://github.com/user-attachments/assets/31a58331-542c-4490-b318-b914bc254515" />


---

### script2 — Detecting non-mixed trackers

it took most time , collected data at runtime

```bash
./detect_non_mixed_tracker.sh [URL]
```

<!-- 📸 INSERT PHOTO HERE: Screenshot of "Detecting Non-mixed Trackers" documentation page showing the command and example output with list of URLs (True/False labels) -->
<img width="1439" height="813" alt="Screenshot 2026-05-10 at 2 08 02 PM" src="https://github.com/user-attachments/assets/7fa3ecd9-334f-4307-852e-3ed37a8506fd" />


<!-- 📸 INSERT PHOTO HERE: Terminal screenshot showing full script2 runtime output — URLs being processed, CSV collection, xgboost model loading, and final predictions -->
<img width="1439" height="813" alt="Screenshot 2026-05-10 at 2 08 11 PM" src="https://github.com/user-attachments/assets/31b4abc7-29e0-4903-88df-c49243312425" />


---

### script3 — Detecting mixed trackers

duumviri collected data for 80 sites — 40 are trackers, 40 are not — but our results show , what duumviri predicted

```bash
./detect_mixed_tracker.sh [URL]
```

<!-- 📸 INSERT PHOTO HERE: TABLE IX + Comparison table (non-mixed vs mixed) showing Dataset, False Positives, Missed trackers, Accuracy -->
<img width="1439" height="813" alt="Screenshot 2026-05-10 at 2 08 21 PM" src="https://github.com/user-attachments/assets/0cf8729a-da6d-44e8-a3fb-ba25a6d9bde4" />

<!-- 📸 INSERT PHOTO HERE: Terminal screenshot showing full script3 runtime output — positive.tsv / negative.tsv labels breakdown and Accuracy Calculation note -->
<img width="1439" height="813" alt="Screenshot 2026-05-10 at 2 09 12 PM" src="https://github.com/user-attachments/assets/b8206bf9-42e5-4d1d-aec8-67cb80d0942d" />


---

## Novelty Idea :-

- add a new feature entropy in the dataset.
- calculate entropy for each page
- then retrain model
- see the predictions

**function to calculate entropy:-**

<!-- 📸 INSERT PHOTO HERE: Screenshot of pagedelta.py in GNU nano 4.8 showing the _entropy(s) function, entropy-based feature calculations (avg_url_entropy, max_url_entropy, avg_param_count, high_entropy_url_ratio, avg_url_depth), and the if debugging block -->
<img width="752" height="461" alt="Screenshot 2026-05-10 at 2 11 39 PM" src="https://github.com/user-attachments/assets/909b5be6-f12f-49eb-8c85-bfdad7f44d93" />


---

**training edited code:-**

<!-- 📸 INSERT PHOTO HERE: Screenshot of eval.py in GNU nano 4.8 showing the train(data_dir) function — loading data, train_test_split, XGBClassifier setup for tracker oracle and functionality oracle, model fitting, prediction, accuracy scoring, and model saving (dump to ml_data/model/) -->
<img width="1447" height="701" alt="Screenshot 2026-05-10 at 2 11 59 PM" src="https://github.com/user-attachments/assets/39c16298-c2a0-4a70-a795-4353af730ea6" />


---

**accuracy results:-**

<!-- 📸 INSERT PHOTO HERE: Terminal screenshot showing the full accuracy grid output (t-thr / f-thr / accu combinations from 0.40 to 0.90) and the final result: "Duumviri achieves the highest accuracy of 0.9661" -->
<img width="1447" height="807" alt="Screenshot 2026-05-10 at 2 12 14 PM" src="https://github.com/user-attachments/assets/e3b8e765-dc44-49ae-b7e5-095efcd4fd03" />


---

### Motivation

The existing feature set in `pagedelta.py` covers visual similarity, DOM changes, and PageGraph structural features — but does not capture any signal from the structure or content of the request URLs themselves. Tracker URLs tend to contain long random-looking identifiers, session tokens, hashed values, and tracking parameters, making them measurably higher in **Shannon entropy** compared to functional resource URLs, which tend to be clean and predictable.

---

### Step 1 — Define the Entropy Feature

Shannon entropy quantifies the randomness of a string:

> H(s) = − Σ (count(c)/|s|) × log₂(count(c)/|s|)

**Example:**
- `cdnjs.cloudflare.com/ajax/libs/jquery/3.6.0/jquery.min.js` → **low entropy** (predictable, clean path)
- `google-analytics.com/collect?cid=1257168463.1778065524&tid=UA-104921371` → **high entropy** (random identifiers, session tokens)

---

### Step 2 — Add Entropy Code to `pagedelta.py`

Added the `_entropy()` function and **5 new derived features** inside `build_network_features()`, immediately after the existing network feature computation block:

| Feature | Description |
|---|---|
| `avg_url_entropy` | Mean Shannon entropy across all blocked request URLs |
| `max_url_entropy` | Highest entropy among all blocked URLs |
| `avg_param_count` | Average number of query parameters per URL |
| `high_entropy_url_ratio` | Fraction of URLs with entropy above **3.5** |
| `avg_url_depth` | Average number of path segments per URL |

> The threshold of **3.5** was chosen empirically — tracker URLs consistently exceed this value while CDN and static resource URLs fall below it.

<!-- 📸 INSERT PHOTO HERE: Screenshot of pagedelta.py showing the _entropy(s) function and the 5 derived features -->

---

### Step 3 — Add a `train()` Function to `eval.py`

The original artifact had **no retraining code** — it only loaded pre-trained binaries. We added a `train()` function that:

1. Loads the augmented CSV data
2. Splits it **80/20** into training and test sets
3. Trains two new XGBoost classifiers (300 estimators, learning rate 0.05)
4. Saves the resulting models to disk

> For the **functionality oracle**, the label was inverted (`~y_train`) because it predicts whether blocking is functionally harmful — the complement of the tracker label.

<!-- 📸 INSERT PHOTO HERE: Screenshot of eval.py showing the train() function -->

---

### Step 4 — Run Retraining

Before retraining, `pagedelta` had to regenerate all CSV feature files so the **5 new entropy columns** were present in the dataset:

```bash
python eval.py train ./data_dir
```

Total pipeline runtime: **~3–4 hours**
- Pagedelta regeneration (bulk of time) — reprocessed each site's pickle data and recomputed all features including new entropy columns
- XGBoost training itself — comparatively fast

---

### Step 5 — Results After Retraining

<!-- 📸 INSERT PHOTO HERE: Terminal screenshot showing full accuracy grid and "Duumviri achieves the highest accuracy of 0.9661" -->

| Metric | Baseline (original) | Retrained (with entropy) |
|---|---|---|
| Peak accuracy | 0.9656 | **0.9661** |
| Best t-thr | 0.50 | 0.50 |
| Best f-thr | 0.90 | 0.90 |
| Training time | N/A (pre-trained) | ≈ 3–4 hours |
| New features added | 0 | **5** |

The improvement is most visible at optimal threshold settings (`t-thr = 0.50`, `f-thr = 0.90`) where the entropy features provide additional signal for tracker requests that produce only subtle visual or structural differences.

---

### Why the Entropy Feature Helps

The original feature set relies heavily on **visual and structural comparisons** between normal and blocked page states. A tracker that injects only a small analytics beacon produces very little visual difference when blocked — making it hard to detect from rendering comparison alone.

URL entropy captures a **complementary property**: the URL of such a beacon will contain session identifiers and user hashes that make it structurally distinct from any CDN resource URL. The combination of both signal types makes the model more robust.



## ThankYou
