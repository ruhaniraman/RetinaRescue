# Retina Rescue

**Explainable AI screening for diabetic retinopathy, built for low-resource clinics.**

Retina Rescue grades diabetic retinopathy (DR) from fundus photographs of both eyes, explains each decision with a heatmap, and helps a specialist review a case in about 30 seconds. It was built for **Smart India Hackathon 2026, problem statement SIH26038 (MathWorks)**. It runs a five-stage MATLAB pipeline behind a FastAPI service and a React web app, with a guided tour and spoken results in English, Hindi and Kannada.

> [!IMPORTANT]
> **Screening aid only.** Results come from automated image analysis. They are not a medical diagnosis and must be reviewed by a qualified eye-care professional. The clinical wording has not yet been reviewed by a clinician (see [Clinical review](#clinical-review)).

---

## Results at a glance

Held-out evaluation of the deployed Stage 3 model (fine-tuned at 384 px on APTOS + IDRiD). The full write-up is in [`validation/REPORT.md`](validation/REPORT.md).

| Test set | Images (referable) | Sensitivity | Specificity | AUC | Kappa (5-class) |
|---|---|---|---|---|---|
| APTOS held-out test | 548 (223) | **96.9%** (93.7–98.5) | **88.3%** (84.4–91.4) | **0.977** | 0.860 |
| IDRiD official test | 103 (64) | 90.6% (81.0–95.6) | 53.8% (38.6–68.4) | 0.922 | — |

*Problem-statement targets: sensitivity > 90%, specificity > 85%. 95% confidence intervals in brackets. Referral threshold 0.096.*

- **The referral decision is strong. The exact stage is weaker:** the 5-way grade is correct 79.4% of the time.
- **Other cameras are the weak point.** IDRiD specificity is 53.8%. A clinic can recalibrate the threshold on its own graded photos with the [calibration tool](#calibrating-for-a-clinic); on IDRiD that raises specificity to 74.4% at 89.1% sensitivity.
- **Confidence is temperature-calibrated** ([`calibration.md`](validation/results/calibration.md)).
- This is an internal technical validation, **not** clinical validation.

---

## The pipeline

| Stage | What it does | How |
|---|---|---|
| **1. Quality** | Rejects blurred, dark or badly framed photos with recapture instructions; optional enhanced view for reviewers | OpenCV port of the MATLAB check (`backend/quality.py`, same verdict on 97.3% of 698 photos); MATLAB `enhanceForReview` |
| **2. Structure** | Retinal vessels, optic disc, fovea and macula zone; optional possible-lesion overlay (MA, HE, EX, SE) | MATLAB: vessel U-Net, disc/fovea localiser, lesion U-Net (`stage2_structure/`) |
| **3. Grading** | Bilateral ICDR grade per eye, calibrated confidence and a referral decision | MATLAB Engine → trained network (`stage_3/`) |
| **4. Explainability** | Grad-CAM heatmap of the referral score, plus PDF report | MATLAB (`stage4_explainability/`) |
| **5. District planning** | Cheapest mix of camera sites, uplink, GPU servers and reviewers for a district; simulate a year of screening | Optimiser + Simulink model `DistrictScreening.slx` (`stage5_simulink/`) |

---

## Features

- **Two-eye assessment** with per-eye grade, calibrated confidence and an overall referral decision, saved to **Exam History**
- **Detailed Report** with four views: *Original*, *AI View* (Grad-CAM), *Enhanced*, *Anatomy* and, when enabled, *Possible lesions*
- **Specialist review** (`/review`): both eyes, heatmaps, grades, referral score against threshold and proliferative-DR probability on one screen, with a review timer, agree/override logging and agreement statistics
- **District Planner** (`/district`): plan resources for a district and confirm the plan in Simulink
- **Low-bandwidth mode:** photos are shrunk to JPEG ≤ 1800 px in the browser, with the upload time on a 2G link shown ([effect on grading](validation/results/compression.md))
- **Listen to the result:** reads the result aloud with on-device voices only (Web Speech API). Hindi and Kannada use fixed, human-written sentences, never machine translation, and stay off until reviewed.
- **Guided welcome tour** in English, Hindi, Kannada and Tamil, where the user completes a real screening step by step
- **One-page PDF report** for both eyes, generated in memory
- **Accounts** with mobile-number verification, `HttpOnly` session cookies, and self-service deletion of exams, health data or the whole account

---

## Architecture

```
                 ┌────────────────────────────┐
  Browser ──────▶│ frontend/  React + Vite    │  :5173
                 └──────┬──────────────┬──────┘
                        │              │
                        ▼              ▼
     ┌──────────────────────┐   ┌────────────────────────────┐
     │ auth-server/         │◀──│ backend/  Python FastAPI   │  :5000
     │ Node/Express :4000   │   │  validates every session   │
     │ accounts, sessions,  │   │  with the auth-server      │
     │ encrypted health data│   └─────────────┬──────────────┘
     │ (SQLite, AES-256-GCM)│                 │ MATLAB Engine for Python
     └──────────────────────┘                 ▼
                                ┌────────────────────────────┐
                                │ MATLAB / Simulink          │
                                │ stage1_quality/ … stage5_  │
                                └────────────────────────────┘
```

The ML backend never sees passwords or the JWT secret. It forwards each request's session to the auth-server (`GET /api/auth/me`) and rejects the request unless that succeeds, so logging out revokes access everywhere.

### Repository layout

```
RetinaRescue/
├── frontend/               React + Vite web app (pages, i18n, speech, tour, district planner)
├── auth-server/            Node/Express: sign-up, SMS verification, sessions, encrypted patient data
├── backend/                FastAPI ML service: server.py, quality.py, report_pdf.py, clinical_text.py
├── stage1_quality/         MATLAB: photo-quality gate (original; the app runs the Python port)
├── stage2_structure/       MATLAB: vessels, optic disc / fovea, lesion networks
├── stage_3/                MATLAB: DR grading model and fine-tuning scripts
├── stage4_explainability/  MATLAB: Grad-CAM and report text
├── stage5_simulink/        MATLAB/Simulink: district resource optimiser and simulation
├── utils/                  Shared MATLAB preprocessing
├── validation/             Evaluation scripts, statistics and result write-ups
├── calibration/            Site threshold-calibration tool
├── docs/                   Demo walkthrough and review-packet generators
└── deploy/                 Caddy, systemd and demo (Vercel + ngrok) deployment
```

---

## Getting started

### Prerequisites

- **Node.js 22.13+** (the auth-server uses the built-in `node:sqlite`, so nothing needs compiling)
- **Python 3.12+**
- **MATLAB** with Deep Learning Toolbox, Image Processing Toolbox and Simulink, plus the [MATLAB Engine for Python](https://www.mathworks.com/help/matlab/matlab_external/install-the-matlab-engine-for-python.html)
- The trained model `stage_3/Stage3_Final_HighSensitivity_Model.mat` (included in the repo, about 41 MB)

### Run locally (three terminals)

**1. Auth server**

```bash
cd auth-server
cp .env.example .env        # set JWT_SECRET (the file shows a generator command)
npm install
npm run dev                 # http://localhost:4000
```

No SMS provider is connected in development, so verification codes are printed in this terminal. For a demo, set `FIXED_OTP=123456` instead (refused in production).

**2. ML backend**

```bash
cd backend
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
# then install the MATLAB Engine for Python (see requirements.txt)
cp env.example .env
python server.py            # http://localhost:5000
```

Wait for `MATLAB Engine ready` in the log. `GET /api/health` reports the MATLAB status.

> **No MATLAB?** Set `DISABLE_MATLAB=true` in `backend/.env`. Only the Stage 1 quality check (Python) and the in-browser district planner keep working; grading, heatmaps, enhancement, anatomy, lesions, PDFs and the Simulink run return a clear `503`. The app never substitutes a made-up result.

**3. Frontend**

```bash
cd frontend
npm install
npm run dev                 # http://localhost:5173
```

Override API addresses in `frontend/.env.local` (see `frontend/.env.example`).

### Using it

1. Sign up with a mobile number, enter the verification code and sign in. A guided tour starts.
2. Fill in the patient profile under **My Health Record**.
3. Upload a fundus photo for each eye. Each is quality-checked, and rejected photos must be replaced.
4. Click **Run AI Assessment**, then open **Detailed Report** for per-eye grades, heatmaps and **Download PDF Report**.

For a scripted six-minute demo covering every problem-statement requirement, see [`docs/DEMO_WALKTHROUGH.md`](docs/DEMO_WALKTHROUGH.md).

---

## Configuration

Key settings (see `auth-server/.env.example`, `backend/env.example` and `frontend/.env.example` for the full list):

| Variable | Where | Purpose |
|---|---|---|
| `JWT_SECRET` | auth-server | Signs sessions |
| `DATA_KEY` | auth-server | AES-256-GCM key for health data. **Back it up and never change it.** |
| `SERVICE_KEY` | auth-server + backend | Lets the backend save results to exam history (same value in both). If unset, results are not saved. |
| `FIXED_OTP` | auth-server | Demo-only fixed verification code |
| `CLIENT_ORIGIN` / `CLIENT_ORIGINS` | auth-server / backend | Frontend origin(s) allowed to call each server |
| `AUTH_SERVER_URL` | backend | Where the backend validates sessions |
| `VITE_AUTH_API_URL`, `VITE_ML_API_URL` | frontend | API addresses |
| `DISABLE_MATLAB` | backend | Run without MATLAB (see above) |
| `REFERRAL_THRESHOLD` | backend | Referral cut-off; change only from a written clinical decision using the calibration tool |
| `STAGE1_ENGINE` | backend | `python` (default) or `matlab` for the quality check |
| `ENABLE_LESION_OVERLAY` / `VITE_ENABLE_LESION_OVERLAY` | backend / frontend | Possible-lesion overlay, off until clinically reviewed |
| `VITE_ALLOW_UNREVIEWED_SPEECH` | frontend | Demo only: read unreviewed Hindi/Kannada/Tamil speech drafts |
| `VITE_ALLOW_NETWORK_VOICES` | frontend | Allow online (non-device) voices for results |

With `NODE_ENV=production` / `APP_ENV=production` the servers refuse to start on weak or placeholder configuration.

---

## Privacy and health data

| Stored (per user, encrypted) | Not stored |
|---|---|
| Patient profile: name, date of birth, gender, blood group, diabetes duration, blood pressure, HbA1c, fasting sugar | Retinal photographs from an assessment (analysed in memory, then discarded) |
| Each assessment: date, per-eye grade and confidence, overall grade, summary | Grad-CAM heatmaps and lesion overlays |
| A PDF report per saved exam (**contains the eye photographs**) | The on-demand **Download PDF Report** file |

- Profiles, exams and saved PDFs are **AES-256-GCM encrypted** before they reach SQLite, so a copied database is unreadable without `DATA_KEY`.
- **Exam results come from the model, not the browser.** Only the backend can write them, using `SERVICE_KEY`.
- Users can delete a single exam, all their health data, or their account. Backups you keep are not touched.
- This is not a compliance certification. Real patient data still needs consent, an audit policy, backups, TLS and legal review (e.g. India's DPDP Act, HIPAA, GDPR).

---

## Calibrating for a clinic

The default threshold (0.096) was chosen for 99% sensitivity on mostly-APTOS validation data, so it over-refers on other cameras. [`calibration/`](calibration/README.md) takes a clinic's own clinician-graded photos, runs the model, and reports the cost of each sensitivity target, with confidence intervals, a hold-out estimate and predictive values at the clinic's prevalence. It recommends a threshold only when given a target, because the target is a clinical decision. Apply the result with `REFERRAL_THRESHOLD`.

## Clinical review

Messages such as "referral is recommended" have **not** been reviewed by a clinician. All patient-facing wording lives in three files (`backend/clinical_text.py`, `frontend/src/clinicalText.js`, `stage4_explainability/report/formatReportText.m`), and tests fail if risky phrasing (reassurance, "required", clinical timings) appears. Generate the review packets locally (they are git-ignored until reviewed):

```bash
python docs/tools/build_clinical_review.py       # → docs/CLINICAL_REVIEW.md
python docs/tools/build_translation_review.py    # → docs/SPOKEN_TRANSLATIONS_FOR_REVIEW.md
python docs/tools/export_stage4_text.py          # after editing the MATLAB report text
```

---

## Testing

```bash
cd frontend    && npm run lint && npm test
cd auth-server && npm test                     # starts the real server on a temporary database
cd backend     && pip install -r requirements-dev.txt && python -m pytest
cd backend     && python -m pytest ../validation ../calibration   # statistics (numpy only)
```

CI ([`.github/workflows/ci.yml`](.github/workflows/ci.yml)) runs everything that does not need MATLAB. The MATLAB tests are in `stage4_explainability/**/test_*.m` and need the APTOS dataset under `data/`.

## Deployment

- **Production:** [`deploy/DEPLOYMENT.md`](deploy/DEPLOYMENT.md) covers HTTPS through Caddy (automatic certificates, security headers, CSP), systemd units, a verification checklist and the decisions still open (clinical validation, regulation, privacy law).
- **Hackathon demo:** [`deploy/DEMO_VERCEL_NGROK.md`](deploy/DEMO_VERCEL_NGROK.md) puts the frontend on Vercel and tunnels the backend from a laptop through ngrok.

---

## Limitations

- **Not clinically validated.** The numbers above are an internal technical evaluation on public datasets.
- **Camera dependence.** Specificity drops sharply on IDRiD; calibrate per site before real use.
- **Stage grading** is less reliable than the referral decision (79.4%).
- **Lesion overlay** marks *possible* lesions in 34% of eyes without retinopathy; Dice vs expert masks is 0.42–0.62 ([details](validation/results/lesions_dl_Stage2_LesionUNet_v2_calibrated.md)). It is off by default.
- **Grad-CAM** is a rough guide to what raised the score, not a lesion detector ([`gradcam.md`](validation/results/gradcam.md)).
- **No SMS provider** is connected yet (`auth-server/sms.js`); add one (e.g. Twilio, MSG91) before going live.
- Health data has a single owner; there is no clinician–patient sharing or audit log yet.
- Model `.mat` files are large binaries in git (consider Git LFS). Datasets are not included because their licences do not allow redistribution.

## License

[MIT](LICENSE)
