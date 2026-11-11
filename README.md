# OU Parking Occupancy Prototype

A full-stack prototype that shows University of Oklahoma students how full campus parking garages are before they drive there.

**All occupancy data is simulated.** This app does not connect to OU, PGS, cameras, gates, or any real parking system yet.

Live dashboard → garage detail pages with an entry/exit feed, occupancy history charts, and a simulated hour-of-day forecast.

## Features

- **Dashboard** — grid of OU garages with live occupancy, status badges (Available / Filling Up / Full), and progress bars
- **Live updates** — the UI polls the API every 3 seconds
- **Simulation** — start/stop fake entry and exit events (all garages or one garage)
- **Garage detail** — large occupancy display, recent event feed, and per-garage sim controls
- **History** — 24-hour and 7-day occupancy charts from stored snapshots
- **Forecast** — typical busy hours from historical hour-of-day averages (clearly labeled simulated; not machine learning)

## Tech stack

| Layer | Tools |
|---|---|
| Frontend | React, Vite, React Router, Tailwind CSS, Recharts |
| Backend | Node.js, Express |
| Database | SQLite (`better-sqlite3`) |

The repo is a split stack: `/client` on port `5173`, `/server` on port `4000`. In development, Vite proxies `/api/*` to Express.

## Architecture

```
Browser (localhost:5173)
  React UI
  fetch('/api/...')
        |
        |  Vite proxy (dev)
        v
Express API (localhost:4000)
  occupancy, simulation, snapshots, forecast
        |
        |  SQL (prepared statements)
        v
SQLite  server/data/parking.sqlite
  garages | events | occupancy_snapshots
```

- The frontend never talks to the database.
- Occupancy is **derived from entry/exit events**, not stored as a number: `occupancy = entries − exits`.
- That matches how gate/camera systems work. A real PGS feed could write the same `events` rows later without changing occupancy logic or the UI.
- Historical charts use `occupancy_snapshots` recorded every 5 minutes, plus synthetic 7-day history seeded on first run so demos are not empty.

## Getting started

Requires [Node.js](https://nodejs.org/).

**Terminal 1 — API**

```bash
cd server
npm install
npm run dev
```

API: http://localhost:4000

**Terminal 2 — UI**

```bash
cd client
npm install
npm run dev
```

App: http://localhost:5173

Open the dashboard, click **Start Simulation**, and occupancy should start changing within a few seconds.

## Project structure

```
OU/
├── server/                 Node + Express + SQLite
│   └── src/
│       ├── index.js        App entry, CORS, routes
│       ├── db.js           Schema + connection
│       ├── seed.js         OU garages + synthetic history
│       ├── occupancy.js    Occupancy, available, status
│       ├── simulation.js   Fake entry/exit generator
│       ├── snapshots.js    5-minute history copies
│       ├── forecast.js     Hour-of-day averages
│       └── routes/
│           └── garages.js  REST endpoints
└── client/                 React + Vite
    └── src/
        ├── api.js          Fetch helpers
        ├── pages/          Dashboard, garage detail, history chart
        └── components/     Cards, badges, event feed, forecast
```

## Seeded garages

| Facility | Capacity |
|---|---|
| Elm Avenue Parking Facility | 577 |
| Jenkins Avenue Parking Facility | 1217 |
| Asp Avenue Parking Facility | 739 |
| Timberdell Road Parking Facility | 990 |

Capacities are placeholder figures based on published OU information and may not match current official counts.

## Occupancy rules

| Field | Rule |
|---|---|
| Occupancy | `max(0, entries − exits)` |
| Available | `max(0, capacity − occupancy)` |
| Status | **Full** if occupancy ≥ capacity; **Filling Up** if available ≤ 20% of capacity; otherwise **Available** |

Occupancy can exceed capacity internally (cars can still enter when a garage is marked full). Display occupancy is only clamped at 0.

## API

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/health` | Health check |
| GET | `/api/garages` | All garages with occupancy and sim status |
| GET | `/api/garages/:id` | One garage |
| POST | `/api/garages/:id/events` | Record an event `{ "event_type": "entry" \| "exit" }` |
| GET | `/api/garages/:id/events?limit=20` | Recent events, newest first |
| POST | `/api/garages/simulate/start` | Start simulation for all garages |
| POST | `/api/garages/simulate/stop` | Stop all simulations |
| POST | `/api/garages/:id/simulate/start` | Start simulation for one garage |
| POST | `/api/garages/:id/simulate/stop` | Stop simulation for one garage |
| GET | `/api/garages/:id/history?range=24h` | Snapshots (`24h` or `7d`) |
| GET | `/api/garages/:id/forecast?day=today` | Simulated hour-of-day forecast |
| POST | `/api/admin/snapshot` | Record a snapshot now |

## Data model

**garages** — campus catalog (id, name, capacity, location)

**events** — source of truth for live occupancy (`entry` or `exit` per garage)

**occupancy_snapshots** — point-in-time occupancy for charts and forecasts
