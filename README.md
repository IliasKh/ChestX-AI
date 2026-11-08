# 🫁 ChestX AI — `web_ui`

The Streamlit application of the project: a clinical-style workspace that runs the trained **ResNet-50** on a chest X-ray, shows the prediction with its **Grad-CAM** explanation, and helps draft a preliminary radiology report.

> ⚠️ **Decision support only.** Results must be reviewed and signed by a licensed radiologist. This is an academic project and is not a medical device.

🇫🇷 Version française : [`README_fr.md`](README_fr.md)

---

## Table of Contents

- [Features](#features)
- [Demo](#demo)
- [Project layout](#project-layout)
- [Requirements](#requirements)
- [One-time setup](#one-time-setup)
- [Run](#run)
- [Configuration](#configuration)
- [Using the app](#using-the-app)
- [How it works](#how-it-works)
- [Troubleshooting](#troubleshooting)
- [Developer notes](#developer-notes)
- [Customizing and extending](#customizing-and-extending)

---

## Features

- **Four-class prediction**: COVID-19, Normal, Pneumonia, Tuberculosis, with a confidence score
- **Differential probabilities**: all four classes shown, sorted, so uncertainty is visible
- **Grad-CAM heatmap** overlaid on the original radiograph, side by side with the original image
- **Findings strip**: readable indicators (consolidation, ground-glass opacity, cavitation/TB, clear lung fields) derived from the class probabilities
- **Severity coloring** (`ok` / `warn` / `alert`) driving the colors on screen
- **Model card** displayed in the workflow: architecture, input size, normalization, Grad-CAM layer, device (GPU/CPU), inference time
- **Patient and study form**: ID, accession, name, age, sex, modality/view, priority, clinical history, risk factors
- **Human-in-the-loop actions**: *Generate radiology report*, *Confirm*, *Flag & route* — nothing is published automatically
- **Automatic CPU fallback** when no GPU is available

## Demo

A demonstration video is available in the [`video_demo/`](../video_demo/) folder at the project root.
Sample chest X-rays to try the app are in [`Image_test/`](../Image_test/).

## Screenshots


## Project layout

The app expects this structure (the checkpoint is located by going one directory up from `web_ui/` and looking in `model/`):

```
<project-root>/
├── Image_test/                     ← sample chest X-rays to try the app
├── video_demo/                     ← demonstration video
├── model/
│   └── best_model_final.pth        ← trained checkpoint
    └── Pipeline_Complet        
├── web_guide/                      ← reference design (unchanged)
└── web_ui/                         ← THIS folder
    ├── app.py                      Streamlit app (UI)
    ├── inference.py                model + transforms + Grad-CAM
    ├── requirements.txt
    └── README.md
```

To use a different checkpoint location, set the `CHESTX_WEIGHTS` environment variable (see [Configuration](#configuration)).

## Requirements

- Python **3.10+**
- The trained checkpoint `best_model_final.pth`
- Packages (see `requirements.txt`): `streamlit`, `torch`, `torchvision`, `Pillow`, `opencv-python`, `numpy`
- A GPU is optional: the app falls back to CPU automatically

## One-time setup

Open a terminal in `<project-root>` (the folder containing `model/` and `web_ui/`).

**Windows (PowerShell)**

```powershell
# 1. Create a virtual environment (recommended)
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 2. Install dependencies
pip install -r web_ui\requirements.txt
```

**Linux / macOS**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r web_ui/requirements.txt
```

> If `pip install torch` is slow, or you want a specific CUDA build, use the selector at <https://pytorch.org/get-started/locally/>.
> Example for CUDA 12.1:
> `pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121`

> If PowerShell blocks the activation script, run once:
> `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

## Run

**Windows (PowerShell)**

```powershell
streamlit run web_ui\app.py
```

**Linux / macOS**

```bash
streamlit run web_ui/app.py
```

Your browser opens at <http://localhost:8501>. The first inference takes a few seconds (model load + Grad-CAM warm-up); subsequent inferences are fast.

To use another port:

```bash
streamlit run web_ui/app.py --server.port 8502
```

## Configuration

| Variable | Purpose | Default |
|---|---|---|
| `CHESTX_WEIGHTS` | Full path to the `.pth` checkpoint | `<project-root>/model/best_model_final.pth` |

**Windows (PowerShell)**

```powershell
$env:CHESTX_WEIGHTS = "C:\full\path\to\best_model_final.pth"
streamlit run web_ui\app.py
```

**Linux / macOS**

```bash
export CHESTX_WEIGHTS=/full/path/to/best_model_final.pth
streamlit run web_ui/app.py
```

## Using the app

1. **Patient & study** (left panel): enter the patient ID, accession number, name, age, sex, modality/view and priority.
2. **Clinical history**: add the indication (symptoms, duration, context) and select any risk factors (e.g. smoker, recent travel).
3. **Image input**: upload a chest X-ray (PNG, JPG or DICOM).
4. Click **Run analysis**.
5. Read the results:
   - **Center**: original X-ray and Grad-CAM overlay, with the findings strip above
   - **Right panel (AI analysis)**: main prediction, confidence, ICD-10 code, differential probabilities and the model card
6. Use the action buttons:
   - **Generate radiology report (PDF)**: draft report to be reviewed and signed by a radiologist
   - **Confirm**: validate the AI suggestion
   - **Flag & route**: send the case for review

A permanent banner reminds the user that the tool is advisory only.

## How it works

```
Uploaded image
   │
   ▼
Preprocessing ── RGB → resize 224×224 → ImageNet mean/std normalization
   │
   ▼
ResNet-50 (ImageNet backbone + custom head)
   │      Linear(2048→512) → BatchNorm → ReLU → Dropout(0.5) → Linear(512→4)
   ▼
Softmax ── probabilities over [COVID, Normal, Pneumonia, Tuberculosis]
   │
   ├──► Grad-CAM on layer4[-1].conv3 ── heatmap resized to 224×224 and overlaid
   │
   ▼
Prediction object ── main class, confidence, ICD-10 code, severity,
                     differential probabilities, findings, heatmap
   │
   ▼
Streamlit UI ── diagnosis card, probabilities, findings strip, model card
```

1. **Loads** `model/best_model_final.pth` into a ResNet-50 with the custom head from the training notebook.
2. **Predicts** softmax probabilities over `[COVID, Normal, Pneumonia, Tuberculosis]` — the exact class order produced by `ImageFolder` (alphabetical), which is what the checkpoint was trained on.
3. **Generates a Grad-CAM** heatmap from `layer4[-1].conv3` (the same target layer used in the notebook) and overlays it on the original radiograph.
4. **Renders** the result in the clinical workspace UI: diagnosis card, differential probabilities, findings strip and model card.

### Reference performance (test set, 1,057 images)

| Accuracy | Macro F1 | Macro ROC-AUC |
|---|---|---|
| 98.01% | 0.978 | 0.9992 |

These figures come from the project's own test split; they are not a guarantee of performance on images from other hospitals or scanners.

## Troubleshooting

**"Model weights not loaded" banner at the top**
- The app couldn't find or load `model/best_model_final.pth`. Check the path shown in the banner. If your `.pth` lives elsewhere, set `CHESTX_WEIGHTS` before `streamlit run` (see [Configuration](#configuration)).

**`Missing key(s) in state_dict` / `Unexpected key(s)`**
- The architecture in `inference.py:build_model()` doesn't match what was trained. The current build matches the notebook exactly. If you later change the head, change it in both places (notebook and `inference.py`).

**`Error(s) in loading state_dict ... size mismatch for fc.4`**
- The number of classes changed. `inference.py` has `NUM_CLASSES = 4` to match `["COVID", "Normal", "Pneumonia", "Tuberculosis"]`. Update both if you retrain with different classes.

**CUDA out of memory / no GPU**
- The app auto-falls back to CPU. Inference takes ~1–3 s per image on CPU vs ~50 ms on GPU. No code changes needed.

**Predictions look wrong / inverted**
- Most likely the class order is off. Verify with one image of each class. The notebook used `datasets.ImageFolder`, which sorts alphabetically → `['COVID', 'Normal', 'Pneumonia', 'Tuberculosis']`. If you trained with a different order, edit `CLASSES` in `inference.py`.

**`streamlit` is not recognized as a command**
- The virtual environment is not activated, or dependencies are not installed. Activate the venv and run `pip install -r web_ui/requirements.txt` again. Alternatively: `python -m streamlit run web_ui/app.py`.

**Port 8501 already in use**
- Another Streamlit instance is running. Close it or start the app on another port with `--server.port 8502`.

**A DICOM file fails to load**
- Try exporting the image to PNG/JPG first. If you need native DICOM support, install `pydicom` and add it to `requirements.txt`.

**Slow first analysis**
- Expected: the model is loaded and Grad-CAM is warmed up on the first run. Later analyses reuse them.

## Developer notes

- **`inference.py`** is the only file that touches PyTorch. The UI never imports `torch` directly, except for the GPU/CPU label in the model card.
- **`Prediction` dataclass** is the contract between inference and UI. Add fields there if you want to surface new information in the workspace.
- **Preprocessing must match training**: same 224×224 resize, RGB conversion and ImageNet normalization. Any change to the training transforms must be mirrored in `inference.py`.
- **PDF report generation** is currently a placeholder. Add `reportlab` to `requirements.txt` and generate real bytes in the `st.download_button` call.

## Customizing and extending

| I want to… | Change this |
|---|---|
| Use a different checkpoint | Set `CHESTX_WEIGHTS`, or replace `model/best_model_final.pth` |
| Change the classes | `CLASSES` and `NUM_CLASSES` in `inference.py`, then retrain and keep the same order |
| Change the classification head | `build_model()` in `inference.py` **and** the training notebook |
| Show a new piece of information in the UI | Add a field to the `Prediction` dataclass, fill it in `inference.py`, display it in `app.py` |
| Generate a real PDF report | Add `reportlab` and build the PDF in `app.py` (`st.download_button`) |
| Deploy to the cloud | Host the app (e.g. Streamlit Community Cloud, Docker) and provide the checkpoint via `CHESTX_WEIGHTS` or a download step |

## Disclaimer

ChestX AI is an academic project. Its outputs are preliminary suggestions to be reviewed by a qualified radiologist and must never replace clinical judgment.
