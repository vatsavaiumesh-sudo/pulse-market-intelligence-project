# Pulse — Self-Healing Market Intelligence Terminal

Built for **Into the Scrape-Verse** (WeMakeDevs × Bright Data), August 17–23, 2026.

Pulse scrapes public crypto prices and financial news with **custom Bright Data Scraper Studio** collectors, stores structured rows in Supabase, generates a short market insight with **CrewAI + NVIDIA NIM**, and **repairs extraction in place** on the same Collector ID when a layout change breaks the scrape — no handwritten CSS selector patches.

Public pages only. No logins, paywalls, or private data. Secrets stay in `.env` (never committed).

---

## What Pulse Does

- Scrapes **CoinGecko** exchange listings (price, 24h change, volume) and **CoinDesk** headlines via two custom Scraper Studio collectors
- Detects broken/empty extraction with a real `brightdata scraper run`, then heals with a real `brightdata scraper heal` on the same `c_*` ID
- Stores flattened rows in Supabase (`market_data`, `news`)
- Generates plain-language insight that cites **actual tickers and headlines** from those tables
- Serves a React dashboard: watchlist, chart, insight card, news feed, and a **Trigger Self-Heal** control (`failed → healing → repaired`)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Scraper | Bright Data Scraper Studio + Bright Data CLI |
| Backend | Python 3.12, FastAPI, Uvicorn |
| Database | Supabase (Postgres) |
| AI | CrewAI + NVIDIA NIM (LiteLLM) |
| Frontend | React 18, Vite, Recharts |
| Hosting | Render (backend), Vercel (frontend) |
| Coding agent | Cursor |

---

## Repo layout

```
pulse/
  backend/          FastAPI, ingest scripts, schema.sql
  frontend/         React + Vite dashboard
  scraper/          Studio CLI notes + collector folders
  sample-output/    Example scrape JSON for judges
  docs/             Self-heal audit + demo shot list
```

`.env` files stay local. Collector IDs in README are public Studio IDs, not API keys.

---

## How Bright Data Scraper Studio Was Used

We did **not** submit a Bright Data library-only scraper. Two **custom** collectors were created in Scraper Studio AI Agent mode and driven from the CLI (and from FastAPI for heal).

| Scraper | Public target | Collector ID | Sample JSON |
|---|---|---|---|
| Prices | https://www.coingecko.com/en | `c_mswww62b2iig1j1hcj` | [`sample-output/example_scraped_data.json`](sample-output/example_scraped_data.json) |
| News | https://www.coindesk.com/ | `c_msx6tyya20kx5jxsy1` | [`sample-output/example_news_data.json`](sample-output/example_news_data.json) |

Create (AI Agent mode) — prices:

```text
brightdata scraper create "https://www.coingecko.com/en" "From the public CoinGecko markets table, extract cryptocurrencies with asset name, ticker, price, 24h percent change, and 24h volume." --name pulse-coingecko-prices
```

Create — news:

```text
brightdata scraper create "https://www.coindesk.com/" "Extract headline, source, published date, and article URL for each news article on this public homepage." --name pulse-coindesk-news
```

Official repair loop (**same Collector ID**, no new scraper, no API contract change):

**run → inspect → heal → approve → re-run**

CLI details: [`scraper/README.md`](scraper/README.md).

### Self-healing in the product

The dashboard button talks to FastAPI, which shells out to the real Bright Data CLI. `repaired` is set **only** when Bright Data’s envelope has `status: "done"` — no timers, no fake progress.

| Endpoint | What it actually does |
|---|---|
| `POST /api/self-heal/detect` | Live `brightdata scraper run`; inspect JSON for empty/broken output (`failed` or `healthy`) |
| `POST /api/trigger-self-heal` | Live `brightdata scraper heal` on `c_mswww62b2iig1j1hcj` |
| `GET /api/self-heal/status` | Poll job until heal finishes or errors |
| `GET /api/self-heal/preflight` | CLI installed + collector configured |

Heal is demonstrated on **localhost** in the demo video. Render’s free Python image typically lacks Node.js / the Bright Data CLI, so shipping heal on that host would be dishonest. Full audit: [`docs/self-heal-explanation.md`](docs/self-heal-explanation.md). Recording steps: [`docs/demo-script.md`](docs/demo-script.md).

If detect returns `healthy`, we do **not** claim the site is broken on camera.

---

## Setup Instructions

### Prerequisites

- Python 3.12, Node.js 18+
- Bright Data CLI (`npm i -g @brightdata/cli` or `npx @brightdata/cli`) and `brightdata login`
- Bright Data, Supabase, and NVIDIA NIM ([build.nvidia.com](https://build.nvidia.com)) accounts

### Backend

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
```

Fill `backend/.env` (never commit it): `SUPABASE_URL`, `SUPABASE_KEY` (service role for ingest), `BRIGHT_DATA_API_KEY`, `NVIDIA_NIM_API_KEY`, `PRICE_COLLECTOR_ID`, `PRICE_SCRAPER_URL`. Full list: [`backend/.env.example`](backend/.env.example).

In the Supabase SQL Editor, run [`backend/schema.sql`](backend/schema.sql), then:

```powershell
python scripts\ingest_sample.py --replace
python scripts\ingest_news_sample.py --replace
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

Health: http://127.0.0.1:8000/health

### Frontend

```powershell
cd frontend
copy .env.example .env
npm install
npm run dev
```

`VITE_API_URL` defaults to `http://127.0.0.1:8000`.

### Self-heal (local)

```powershell
Invoke-RestMethod http://127.0.0.1:8000/api/self-heal/preflight
Invoke-RestMethod -Method POST http://127.0.0.1:8000/api/self-heal/detect
Invoke-RestMethod -Method POST http://127.0.0.1:8000/api/trigger-self-heal
Invoke-RestMethod http://127.0.0.1:8000/api/self-heal/status
```

Heal can take several minutes.

---

## API Endpoints

| Endpoint | Description |
|---|---|
| `GET /health` | Health check |
| `GET /api/prices` | Flattened CoinGecko listings from `market_data` |
| `GET /api/news` | CoinDesk articles from `news` |
| `GET /api/insight` | CrewAI + NVIDIA NIM insight from those rows |
| `GET /api/activity` | Real scrape/heal events for the Logs page |
| `GET /api/self-heal/preflight` | CLI / collector ready check |
| `POST /api/self-heal/detect` | Real scrape-run failure detection |
| `POST /api/trigger-self-heal` | Real Bright Data heal |
| `GET /api/self-heal/status` | Heal job status |

---

## Sample Output

- Prices: [`sample-output/example_scraped_data.json`](sample-output/example_scraped_data.json) — coins with `markets[]` (exchange, ticker, price, 24h change, volume)
- News: [`sample-output/example_news_data.json`](sample-output/example_news_data.json) — CoinDesk `headline`, `source`, `published_at`, `url`

---

## AI Tools Disclosure

AI-assisted development is allowed under hackathon rules. This project used:

| Use | Tool |
|---|---|
| Custom scraper generation | Bright Data Scraper Studio AI Agent mode |
| Application scaffolding and pairing | Cursor, under human direction |
| In-product market insight | CrewAI + NVIDIA NIM (`meta/llama-3.1-8b-instruct`) |

All scraper runs, schema, ingest, APIs, and the self-heal CLI path were reviewed, executed, and verified by the team. We can explain architecture and technical decisions. No API keys are hardcoded.

---

## Demo Video

_Add the public URL before submission._

The video should show: live dashboard (prices, news, insight), how Scraper Studio collectors were built, and a real self-heal sequence on **localhost** (`failed` or disclosed `healthy` → `healing` → `repaired`). Shot list: [`docs/demo-script.md`](docs/demo-script.md).

---

## Team

Built for Into the Scrape-Verse, August 2026.

| Person | Role |
|---|---|
| Devendra | Backend |
| Hardeep | UI/UX |
| Umesh chandra | Team lead + AI infra |
| Mahesh | DBMS |

---

## License

MIT — see [LICENSE](LICENSE).
