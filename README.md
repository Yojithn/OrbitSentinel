# OrbitSentinel

**Detection is solved, decisions aren't.** Free collision warnings exist, but satellite operators get raw
numbers with no advice, and there is no rule for who moves when two active satellites are on a collision
course. OrbitSentinel predicts close approaches from live orbital data, recommends an avoidance burn,
negotiates who should move, and shows what a single breakup would do to the rest of orbit.

![OrbitSentinel demo](frontend/docs/demo.gif)

## Features

- **Live catalog on a 3D globe**: every tracked on-orbit object (about 32,500) from Space-Track.org
  general perturbations data in OMM format, propagated with SGP4 in the browser.
- **Conjunction screening**: a 7-day SGP4 screen of the watch-list against the whole catalog, with miss
  distance, relative speed, RTN offsets, uncertainty ellipse and a replay of the encounter.
- **Collision probability**: worst-case (maximum) Pc over covariance scale, because public data has no
  covariance (Alfano 2005, the same idea as CelesTrak SOCRATES' MAX_PROB).
- **Avoidance options**: prograde and retrograde burns with delta-v, propellant (rocket equation), new
  miss distance and new Pc, plus "No action"; a burn is recommended only above the NASA CARA red level.
- **AI Negotiator**: when both objects can maneuver, one agent per operator and a mediator negotiate who
  moves, scored on propellant reserve, maneuver cost, mission impact and delta-v, with adjustable weights.
- **Sentinel brief and Ask Sentinel**: a plain-English risk summary and a chat that only answers from the
  computed data; numbers are validated, with a deterministic fallback, so it never invents values.
- **Rewind 2021**: real 2021 Starlink near-miss cases replayed to show how much faster and cheaper the
  decision would have been.
- **Cascade Lab**: fragments a chosen object and projects the extra conjunctions over the following years.
- **Mobile alert view** for the on-call operator.

## Architecture

| Layer | Stack |
| --- | --- |
| Frontend | React, TypeScript, Vite, three.js, satellite.js |
| API | Python 3.11+, FastAPI, Pydantic v2 (`backend/app/main.py`, `backend/app/routers/`) |
| Data | Space-Track.org GP (or CelesTrak GP as a fallback), cached locally or in S3 (`backend/app/data/`) |
| Physics | sgp4 + numpy screening, Pc, maneuver planning, cascade model (`backend/app/physics/`) |
| AI | Grounded LLM text with a free-provider chain and cached fallback (`backend/app/ai/`) |
| Deploy | AWS Lambda (Mangum) + API Gateway, S3, EventBridge |

The frontend types in `frontend/src/api/types.ts` are the API contract; `backend/app/models.py` mirrors them.

## Run locally

### Backend

```sh
cd backend
python -m pip install -r requirements.txt
cp .env.example .env        # add SPACETRACK_USERNAME and SPACETRACK_PASSWORD (free account)
python -m app.data.refresh  # download the live catalog (at most once every 2 hours)
python -m uvicorn app.main:app --port 8000
```

The first screening runs in the background at startup and takes about a minute per watched object.
Never commit `.env`; it is git-ignored.

### Frontend

Use Node.js 22.12 or later.

```sh
cd frontend
npm ci
echo "VITE_USE_MOCKS=false" > .env.local
echo "VITE_API_BASE_URL=http://localhost:8000" >> .env.local
npm run dev
```

Without `.env.local` the frontend runs on built-in sample data and needs no backend.

### Tests

```sh
cd backend && python -m pytest -q
cd frontend && npm run lint && npm test && npm run build
```

## Limitations

- Public GP data has no covariance, so Pc is a worst-case bound, not an operator-grade estimate.
- "Maneuverable" means an active payload of a known propulsive fleet (Starlink, OneWeb, Kuiper, ISS,
  Tianhe) or an ID listed in `CATALOG_MANEUVERABLE_IDS`; it is an assumption, not operator-verified.
- Space-Track has no operational status, so other payloads are shown as status unknown.
- The negotiation is a decision aid. No commands are sent to any spacecraft; a human operator approves
  every maneuver.

More detail: [frontend guide](frontend/README.md), [engineering decisions](frontend/DECISIONS.md),
[data pipeline](backend/app/data/README.md), [asset credits](frontend/ASSET_CREDITS.md).

## Contributors

Built for the Anvathon hackathon by Yojith N, Utkarsh Keshri, Yash Mathur and Upendra Singh Gaur.
