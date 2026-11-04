# Remediscent

A web app that helps a family manage the medicines in their home. It tracks every family member's medicines, stock and expiry dates, warns about dangerous drug combinations and allergies, and predicts when each medicine will run out so you can reorder in time.

Remediscent is a household prototype, not a medical device. It does not diagnose or give medical advice.

## Features

- **Family inventory:** one account per household, with medicines and allergies recorded for each family member.
- **Expiry and stock alerts:** each medicine is marked as expiring soon, expired or low on stock.
- **Interaction and allergy checks:** every pair of a member's medicines is checked against a knowledge base of risky combinations (for example, warfarin with aspirin), and each medicine is checked against that member's allergies.
- **Reorder prediction:** predicts the run-out date and the date to reorder, from how much of each medicine is actually used.
- **Analytics:** flags unusual consumption (missed or double doses), scores adherence, and groups medicines that are running out into fewer pharmacy trips.
- **Voice entry:** say a medicine's details and the form fills itself, ready for you to review.
- **Dashboard and notifications** for all of the above.

## How it works

**Reorder prediction.** The app builds a 30-day record of how many units were used each day. A Random Forest model learns from the usage 1, 2, 3 and 7 days earlier and predicts tomorrow's usage. Current stock divided by that daily usage gives the run-out date, and the reorder date is 5 days before it. With fewer than 5 days of history, the app uses a simple 7-day average instead, because a model trained on so little data is not reliable. Each prediction records which method was used, with its MAE and R² scores.

**Safety checks.** Medicine names and generic names are compared in lower case, so "Aspirin" and "aspirin" match. Interactions are sorted by severity: HIGH, MODERATE, then LOW.

**Analytics.** A day is flagged as unusual when its usage is 2 or more standard deviations away from the 14-day average (a z-score of 2). Adherence is doses taken divided by doses expected over the last 30 days.

## Tech stack

| Layer | Tools |
| --- | --- |
| Backend | Python, Flask, Flask-Login, Flask-SQLAlchemy |
| Machine learning | scikit-learn, NumPy |
| Frontend | HTML, Bootstrap 5, Jinja2 templates, JavaScript, Web Speech API |
| Database | SQLite locally, PostgreSQL in production |
| Deployment | Gunicorn, Docker, Render or Railway |

## Run it locally

1. Clone the repository and open its folder:

   ```bash
   git clone https://github.com/ragineechun/remediscent.git
   cd remediscent
   ```

2. Create and activate a virtual environment, then install the dependencies:

   ```bash
   python -m venv venv
   venv\Scripts\activate        # Windows
   # source venv/bin/activate   # macOS / Linux
   pip install -r requirements.txt
   ```

3. Copy the example settings file and set your own `SECRET_KEY` in it:

   ```bash
   copy .env.example .env       # Windows
   # cp .env.example .env       # macOS / Linux
   ```

4. Optional: add demo data (a household with three medicines, including a risky combination):

   ```bash
   flask --app app seed-demo
   ```

5. Start the app:

   ```bash
   python app.py
   ```

   Open http://localhost:5000. If you added demo data, log in with **demo@remediscent.app** and password **demo1234**.

## Use the app

1. Open **Members** and add the people in your household, with any allergies.
2. Open **Medicines** and click **Add Medicine**, or use **Voice Entry** and say something like "Paracetamol 500 milligram, quantity 20, expires 2026-12-31".
3. Each time someone takes a dose, open the medicine and click **Log Dose**.
4. Click **Predict Reorder Date** on a medicine to see its run-out and reorder dates.
5. Check the **Dashboard** for safety alerts, the refill plan and adherence.

## Deploy

### Render

1. Push the repository to GitHub.
2. On [Render](https://render.com), choose **New** > **Web Service** and connect the repository.
3. Set the build command to `pip install -r requirements.txt` and the start command to `gunicorn app:app --bind 0.0.0.0:$PORT`.
4. Create a PostgreSQL database on Render and copy its internal database URL.
5. Add these environment variables: `SECRET_KEY` (a long random string), `DATABASE_URL` (the URL from step 4) and `FLASK_DEBUG` set to `0`.
6. Deploy. The tables are created on first start.

### Railway

1. Push the repository to GitHub.
2. Create a new project and choose **Deploy from GitHub**.
3. Add the **PostgreSQL** plugin. Railway sets `DATABASE_URL` for you.
4. Add a `SECRET_KEY` variable. Railway detects the `Procfile` and starts the app.

### Docker

```bash
docker build -t remediscent .
docker run -p 8000:8000 -e SECRET_KEY=change-me -e DATABASE_URL=sqlite:///data.db remediscent
```

## API

| Method | Endpoint | Returns |
| --- | --- | --- |
| GET | `/api/predict/<medicine_id>` | Run-out and reorder prediction for a medicine (JSON) |
| GET | `/api/interactions/<member_id>` | Interaction warnings for a family member (JSON) |
| GET | `/health` | Health check |

## Troubleshooting

| Message or problem | What to do |
| --- | --- |
| "Not enough consumption history." | Log doses first. The prediction needs some usage recorded in the last 7 days. |
| The microphone button is disabled on Voice Entry | Your browser does not support speech recognition. Use Chrome or Edge. |
| Voice entry fills the wrong values | Speak in the format shown on the page, then correct the fields before saving. Nothing is saved until you click **Save Medicine**. |
| No interaction warning appears | Interactions are only checked between medicines of the same family member, and only for pairs in the knowledge base. |

## Limitations

- The interaction knowledge base has 15 known risky pairs. A real product would use a licensed drug database.
- Pill identification from photos and reading labels with OCR are planned but not built yet.
- Predictions improve with more history; the first weeks rely on the moving-average fallback.
- Prototype for household use only. It is not a medical device.
