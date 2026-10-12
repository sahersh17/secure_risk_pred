# Secure Risk Prediction with Hybrid ECC + AES-GCM Encryption

A prototype of a privacy-preserving prediction pipeline: an ML ensemble predicts a risk label, and the result is protected end to end using a **hybrid cryptosystem** (ECDH key exchange, HKDF key derivation, AES-256-GCM authenticated encryption) before being stored in a simulated cloud and read by an authorized party (e.g. a doctor). The notebook also benchmarks the encryption overhead against plain AES.

## Highlights

- **Key exchange:** ECDH on the NIST P-256 curve (`secp256r1`) between a user and a server/doctor; both sides derive the same shared secret
- **Key derivation:** HKDF-SHA256 turns the shared secret into a 256-bit AES key
- **Encryption:** AES-256-GCM, so data is both confidential and tamper-evident (authentication tag verified on decrypt)
- **ML model:** soft-voting ensemble of an MLP (128, 64) and XGBoost, averaging predicted probabilities
- **Workflow simulation:** `user_send_data` → `store_in_cloud` → `retrieve_from_cloud` → `doctor_view_data`
- **Benchmark:** encryption and decryption latency of the hybrid scheme vs. raw AES

## Pipeline

```
 User ──ECDH──► shared secret ──HKDF──► AES-256 key
   │                                        │
   │  prediction (MLP + XGBoost ensemble)   │
   ▼                                        ▼
 plaintext result ──AES-GCM encrypt──► (nonce, ciphertext, tag) ──► Cloud (simulated list)
                                                                        │
 Doctor ◄──AES-GCM decrypt + verify tag◄────────────────────────────────┘
```

1. Generate P-256 key pairs for the user and the server; confirm both ECDH shared secrets match
2. Derive the AES key with HKDF
3. Round-trip test: encrypt and decrypt a sample message
4. Load the dataset, binarize the target, drop constant/ID columns, one-hot encode categoricals, 80/20 train/test split (`random_state=42`)
5. Train the MLP and XGBoost models, average their probabilities, threshold at 0.5
6. Encrypt a prediction, store the `(nonce, ciphertext, tag)` bundle in the simulated cloud, decrypt on the doctor side
7. Time encryption/decryption and plot the comparison against AES alone

## Dataset

[IBM HR Employee Attrition](https://www.kaggle.com/datasets/pavansubhasht/ibm-hr-analytics-attrition-dataset) (`WA_Fn-UseC_-HR-Employee-Attrition.csv`), 1,470 rows. After preprocessing there are 44 features; the training set has 1,176 samples. It is used here as a **stand-in dataset** to exercise the pipeline; the example message in the encryption test ("Patient Depression Score") illustrates the intended healthcare use case, but no medical data is used.

## Results

| Metric | Value |
|---|---|
| Ensemble accuracy (test set) | 85.0% |
| Encryption time (hybrid scheme, single run) | ~2.50 ms |
| Decryption time (single run) | ~0.48 ms |
| AES-GCM only, encrypt (single run) | ~0.44 ms |

## Requirements

- Python 3.9+ (developed on Google Colab, Python 3.12)
- `cryptography`, `pycryptodome`, `scikit-learn`, `xgboost`, `pandas`, `numpy`, `matplotlib`

```bash
pip install cryptography pycryptodome scikit-learn xgboost pandas numpy matplotlib
```

## Usage

1. Open `crypto_jackfruit (1).ipynb` in Google Colab (or Jupyter)
2. Run the cells in order
3. Upload `WA_Fn-UseC_-HR-Employee-Attrition.csv` when prompted (outside Colab, replace the `files.upload()` cell with a local path)

## Limitations and known issues

Worth knowing before presenting or extending this work:

- **Single-run timings are noisy.** Each latency figure comes from one call, so the first-call overhead (imports, cold caches, key schedule setup) inflates the "hybrid" number. Averaging over many iterations (e.g. `timeit`, 1,000+ runs) would give a fair comparison. Also, the hybrid scheme and "AES only" both use AES-GCM for the actual encryption, so the true per-message difference should be small; the key exchange and derivation happen once per session.
- **The plot uses hardcoded values** (`2.5`, `0.44`) rather than the measured variables.
- **ECDH is unauthenticated.** Without signatures or certificates, a man-in-the-middle could substitute keys. Both parties also live in the same notebook, so this simulates the protocol but not a real network exchange.
- **HKDF uses no salt**, and the keys are static, so there is no forward secrecy across sessions.
- **The encrypted "risk score" is a binary class label** (`final_pred[0]`), not a probability or a score.
- **Accuracy alone is a weak metric here.** The dataset is imbalanced (about 16% attrition), so a model that always predicts "No" scores roughly 84%. Precision, recall, F1 and ROC-AUC should be reported.
- **Cloud storage is simulated** with an in-memory Python list.
- Minor: `use_label_encoder` is deprecated in recent XGBoost versions and triggers a warning; remove it.

## Possible extensions

- Benchmark properly over many iterations and across message sizes; compare against RSA and ChaCha20-Poly1305
- Sign the public keys (ECDSA) or use certificates to authenticate the exchange; use a random HKDF salt and per-session ephemeral keys
- Evaluate with imbalance-aware metrics and add class weighting or threshold tuning
- Swap in a real health-related dataset and a real storage backend
- Explore federated learning or homomorphic encryption so the model never sees plaintext

## Project structure

```
.
├── crypto_jackfruit (1).ipynb   # Key exchange, encryption, ML ensemble, benchmarks
└── README.md
```
