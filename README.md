# Smart Attendance Monitoring & Academic Performance Prediction System

Face-recognition attendance (ArcFace) + machine-learning performance prediction
and early risk flagging, in one platform for educational institutions.

## Quick start (Windows PowerShell)

```powershell
python -m venv .venv
.venv\Scripts\activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python app.py
```

Open **http://127.0.0.1:5000**. That's it — on first start the app creates its
database, loads the **built-in demo school**, trains the prediction model and
generates predictions, so every screen has data immediately.

| Login | Password | Role |
|---|---|---|
| `admin` | `admin123` | admin |
| `teacher_a` / `teacher_b` / `teacher_c` | `teacher123` | teacher |

**Change these** (sidebar → *Change password*, admin → *Users*). A sidebar badge
reads **DEMO DATA** while demo records are loaded.

For a production-style run use `python serve.py` (Waitress; Windows-friendly).


## Deploy on Render (portfolio/demo mode)

This repository includes a `render.yaml` Blueprint and uses the included Dockerfile.
To deploy:

1. Push this project folder to a GitHub repository.
2. In Render, choose **New → Blueprint**, connect that repository, and apply the `render.yaml` configuration.
3. Wait for the Docker build and first boot. The first boot may take longer because InsightFace downloads its `buffalo_l` model pack and the demo dataset/model are initialized.
4. Open the Render URL and sign in with the demo credentials below. Immediately change the demo passwords before sharing the URL publicly.

**Demo login:** `admin` / `admin123`; teachers: `teacher_a`, `teacher_b`, `teacher_c` / `teacher123`. These are intentionally convenient demo credentials, not safe for a public production system. Do not put real student data on this public demo deployment.

### Important Render limitations

- The free service has an ephemeral filesystem. The SQLite database, captured face embeddings, generated model files and registry are stored under `/tmp/smartattend` and can be lost on a restart or redeploy. This setup is for showcasing the workflow, not retaining real institutional records.
- Free web services may spin down when idle, so the first request after inactivity can be slow.
- The existing live-camera module opens a webcam attached to the **server**. A Render server cannot directly open a visitor's laptop webcam. Therefore, live face registration/attendance from a remote browser needs a separate browser-camera implementation (for example, browser `getUserMedia` sending frames to a protected endpoint) or a camera agent running on the institution's local machine. The hosted demo can still showcase dashboard, attendance records, marks, reports and predictions.
- The model is trained from synthetic demo data. Prediction scores are not evidence of real-world academic accuracy.
- Do not upload real biometric or student records to a public demo. A real deployment needs consent, appropriate institutional approval, retention controls, durable storage, and a security review.

### Keeping data on a paid Render service

For a deployment that supports a persistent disk, attach a disk mounted at `/var/data`, then change the storage environment variables to paths on that mount, for example `DATABASE_PATH=/var/data/attendance.db`, `FACES_DIR=/var/data/faces`, `FACE_GALLERY_PATH=/var/data/models/face_gallery.npz`, `ML_DATASET_PATH=/var/data/academic_dataset.csv`, and `MODEL_REGISTRY_DIR=/var/data/models/registry`. Also set `CURRENT_VERSION_POINTER` and `BASELINE_STATS_PATH` under `/var/data/models/registry`. This preserves files but does not make the server-side webcam accessible from a remote browser, and it does not replace a security review.

## What's built in (all synthetic, reproducible)

45 students in sections A/B/C · 8 global subjects · ~5 weeks of attendance
history · 3–4 exams per subject · a trained, versioned model · a prediction for
every student with complete data (≈ 19 Low / 17 Medium / 6 High risk).
The last 3 students are "new admissions" with no history, so you can see the
system report *exactly what's missing* instead of guessing.
Face embeddings are **not** seeded — capture them from a real webcam.

Reset to a clean demo: `python database\seed.py --reset`
Start empty instead: set `AUTO_SEED_DEMO=0` (see `.env.example`).

## Workflow

```
Syllabus Management ─ subjects registered once, globally
Student registration → Face capture → Face recognition → Automatic attendance
Academic Data ─ Subject + free-typed Exam Name + Total Mark → marks grid
Features (Attendance %, Marks Average %, Previous GPA) → ML prediction → Risk
Dashboard · Reports (PNG / PDF / CSV)
```

* **Face capture:** *Face Registration* → pick a student → *Start Capture*. The
  camera runs in a background worker; samples are saved automatically and the
  recognition gallery rebuilds itself when capture completes.
* **Marks:** *Academic Data* → subject, exam name, total mark → *Load Sheet* →
  type obtained marks (percentage updates live) → *Save*. Re-opening the same
  subject + exam edits in place. CSV import/export supported.
* **Prediction:** the model trains on a synthetic baseline dataset; real
  students' *current* numbers are fed to it for prediction. (There's no
  reliable "true final outcome" label in free-form marks data to train on —
  a deliberate, documented trade-off.)

## Architecture (v2)

See **docs/ARCHITECTURE.md** for the full picture and an honest list of what is
implemented vs. documented-but-not-built. In short:

* **Camera worker** — one background thread owns the webcam; HTTP just serves its latest frame.
* **Job queue** — training and bulk prediction run in the background with live progress.
* **Vector index** — FAISS if installed, otherwise NumPy; exact cosine search.
* **Migrations** — versioned SQL files, SQLite in WAL mode.
* **Security** — CSRF tokens, role-based access, login throttling, audit log, secure cookies, no hard-coded secret.
* **Ops** — structured logs (`logs/app.log`), `/health`, env-driven config, CI, Docker files.

## Configuration

Environment variables or a `.env` file (copy `.env.example`): `APP_ENV`, `HOST`,
`PORT`, `SECRET_KEY`, `AUTO_SEED_DEMO`, `INITIAL_ADMIN_PASSWORD`,
`LOGIN_MAX_ATTEMPTS`, `JOB_WORKERS`, `VECTOR_BACKEND` (`auto|faiss|numpy`), `LOG_LEVEL`.
Optional FAISS: `pip install -r requirements-optional.txt`.

## Tests

```powershell
python -m pip install -r requirements-dev.txt
python -m pytest tests -q
```

104 tests, including one end-to-end test with the **real ArcFace model**
(register two faces → recognise each → reject a stranger) that skips itself if
the model weights can't be downloaded.

## Camera troubleshooting

* `python tools\test_camera.py` — raw camera check (isolates driver/permission problems).
* `python tools\test_face_detection.py` — live detection scores and frame brightness.
* First run downloads the ArcFace `buffalo_l` pack (~280 MB) to `~/.insightface`.
* Windows: Settings → Privacy & security → Camera → allow desktop apps; close Zoom/Teams.
* Try another `CAMERA_INDEX` in `config.py` if you have several cameras.

## Privacy

Only 512-d face embeddings are stored (`data/faces/<id>/*.npy`) — never face
images — but they are still biometric data: protect that folder, don't commit it
(it's in `.gitignore`). A real deployment needs student consent, institutional
authorisation, access control, and a retention/deletion policy.

## Known limitations

* No liveness / anti-spoofing: a photo held to the camera could be recognised.
* Prediction accuracy is validated only against synthetic data.
* One camera, one machine; camera and job state are per-process, so run a single server process.
* Face accuracy depends on lighting and camera quality.
