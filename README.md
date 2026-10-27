# Evaluating Phishing Detection Under URL Obfuscation

This project implements a research-oriented web application for evaluating how URL obfuscation techniques affect a deep learning-based phishing URL detector.

## Research objective

Evaluate the effect of four URL obfuscation techniques on the performance of a character-level deep learning phishing classifier:

- Character substitution
- URL encoding
- Symbol insertion
- Subdomain modification

The application includes:

- automatic loading and validation of the workspace's `PhiLegit-100K.csv`, including malformed URL and duplicate handling
- optional CSV upload for using a different dataset in the web interface
- registered-domain-grouped, stratified 70/15/15 train/validation/test partitions
- a PyTorch character-level URL model
- validation-only hyperparameter tuning and five-seed training
- VS Code tasks to validate the local dataset or train without using the web interface
- held-out test evaluation using controlled phishing-only URL obfuscations
- SQLite-backed experiment storage
- React + Vite frontend and Flask backend

## Stack

- Frontend: React + TypeScript + Vite
- Styling: Tailwind CSS
- Backend: Flask + Python
- ML: PyTorch + scikit-learn
- Storage: SQLite

## Installation

1. Create and activate a virtual environment.
2. Install backend dependencies:
   ```bash
   python -m pip install -r backend/requirements.txt
   ```
3. Install frontend dependencies:
   ```bash
   npm install
   ```
4. Copy `.env.example` to `.env` and customize as needed.

## Run the app

Start the backend:
```bash
python -m flask --app backend.app run --debug --host 0.0.0.0 --port 5000
```

Start the frontend:
```bash
npm run dev -- --host 0.0.0.0
```

Open the frontend at `http://localhost:5173`.

## Train directly in VS Code

The backend automatically loads `PhiLegit-100K.csv` from the workspace root using the `URL` and `label_legit_1_phish_0` columns (`0` = phishing, `1` = legitimate). You do not need to upload this file through Dataset Management. Keep the CSV in the workspace root; it is already included with this project.

1. Open the project folder in VS Code and select the project's Python environment with **Python: Select Interpreter**.
2. To check the dataset and generated partitions without starting training, run **Terminal → Run Task… → Phishing: Validate local dataset (dry run)**.
3. To train and save the model, run **Terminal → Run Task… → Phishing: Train research model**. Training runs in VS Code's terminal and saves the five-seed model to `models/phishing_char_model.pt`.

The training task uses the same backend training pipeline as the web application. It performs validation-only tuning and reserves the test partition for final evaluation. The task defaults to five consecutive seeds starting at 42, batch size 128, up to 30 epochs per final run, and up to five tuning epochs per candidate. These can also be set from a terminal with `python -m backend.train --seed 42 --batch-size 128 --epochs 30 --tuning-epochs 5`.

Use either the VS Code task or the web interface to train, not both at the same time. The current workspace environment has CPU-only PyTorch (`2.5.1+cpu`), so it will train on the CPU; a CUDA-enabled PyTorch environment is needed to use the NVIDIA GPU. The selected device is shown in Model Training.

## Dataset requirements

For the supplied `PhiLegit-100K.csv`, the URL and label columns are `URL` and `label_legit_1_phish_0`; raw labels use `0` = phishing and `1` = legitimate. Labels are normalized internally to `0` for legitimate and `1` for phishing. The web upload form also supports configurable column names, the inverse numeric encoding, and recognizable text labels such as `Legitimate` and `Phishing`. Unsupported labels and URLs with conflicting labels are rejected rather than silently discarded. Malformed and duplicate URLs are removed and counted in the preprocessing report.

If no dataset is supplied, the application can generate a clearly labeled synthetic demonstration dataset for quick testing, but research usage should use a licensed real dataset.

For the supplied `PhiLegit-100K.csv`, the raw labels are 50,000 each. URL validation removes 2 malformed records, leaving 50,000 legitimate and 49,998 phishing URLs (internal labels 0 and 1 respectively). With the current pinned dependencies, registered-domain grouping produces 70,654 training, 16,243 validation, and 13,101 test records, with 53,052, 11,492, and 10,850 registered domains respectively. Group integrity takes precedence over exact row-level percentages. The app reports the actual split counts and number of registered domains after split generation. No samples are fabricated or oversampled.

## Model choice

The methodology-aligned classifier is a character-level CNN: 32-dimensional character embeddings, convolution layers with 128 filters and kernel sizes 5 and 3, max pooling, global max pooling, a 64-unit dense layer, dropout 0.5, and a sigmoid output. Inputs have a fixed length of 200 characters, preserve case and special characters, map unsupported characters to an unknown token, and remove the protocol prefix. Training uses Adam, binary cross-entropy, batch size 128, up to 30 epochs, and early stopping on validation loss with patience 3. Four candidate configurations are selected using validation F1; the test set is reserved for final evaluation. The selected configuration is trained with five consecutive random seeds, and each run saves its best validation-loss checkpoint. Training automatically uses CUDA when the selected PyTorch environment supports it and otherwise falls back to CPU. The implementation uses PyTorch, as selected for this project, rather than the proposal's Keras/TensorFlow framework; the experimental design and initial model/training settings are retained.

## Research workflow

1. Keep `PhiLegit-100K.csv` in the workspace root. The backend loads it automatically; alternatively, use the **Phishing: Validate local dataset (dry run)** VS Code task to inspect preprocessing and split sizes without training.
2. Check the raw and preprocessed class counts, validation report, and registered-domain partition counts in Dataset Management. Training is rejected if either label is invalid, either class is absent in a partition, there are too few samples, or registered-domain groups cannot support the split.
3. Run **Phishing: Train research model** in VS Code, or select **Tune and train** in the web interface. Four configurations are compared using the validation set; then five seeded CNN runs are trained. The test set is not used for tuning.
4. Select **Run final test-set evaluation** in Experiments. The app evaluates the baseline and four techniques at low, medium, and high intensity. Only phishing test URLs are transformed; legitimate URLs remain unchanged. Failed transformations are counted and excluded from that paired condition rather than replaced.
5. Review accuracy, precision, recall, F1, confusion-matrix averages, F1 decrease relative to baseline, and paired evasion rate (baseline-detected phishing URLs subsequently missed), summarized across seeds.

Evaluation conditions and transformations are deterministic for a given protocol and seed. Obfuscations are syntactic research transformations, are quality-checked as URLs, and are never visited. Experiments require a trained five-seed model matching the currently loaded dataset fingerprint. Persisted experiment records from earlier protocols should not be compared as methodology-v1 results.

## Notes on experimental integrity

- The application records preprocessing and experiment settings.
- Results are computed from real predictions and ground-truth labels.
- The same seed model and held-out test samples are paired across baseline and obfuscation conditions.
- Generated obfuscations are syntactic transformations and should not be treated as verified live phishing URLs.

## Project structure

- `src/` — React frontend
- `backend/` — Flask API and ML modules
- `data/` — uploaded datasets and SQLite database
- `docs/` — research documentation
- `README.md` — installation and usage guide
