# HeadStart — Project Report

**A Financial Decision Support System**
Version 1.0.0 · Packaged as a single Docker image · Report date: 14 Nov 2026

---

## 1. What the app is

HeadStart is a web application that helps a person **research markets and make better-informed financial decisions** in one place. It does not trade, hold money or give personal buy/sell advice. It collects real market data, shows it clearly, lets the user track their own holdings and watch-lists, warns them when something they care about happens, and offers an **AI analyst** that answers questions using only the data the app has actually retrieved.

A user signs up with an email and password, then works inside a private workspace with these sections:

| Section | What the user can do |
| --- | --- |
| **Overview (Dashboard)** | See a snapshot of the market: indices, top movers, headline summary |
| **Markets** | Browse market summary, index levels and the day's biggest gainers/losers |
| **Assets** | Search any ticker (e.g. AAPL) and open its page: live quote, price history chart, fundamentals, financial statements, related news and an AI-generated analysis |
| **Compare** | Put two or more stocks side by side |
| **Portfolio** | Record transactions and holdings; the app computes value and performance |
| **Watchlists** | Keep named lists of tickers to follow |
| **Alerts** | Set rules such as "AAPL above $200" or "moves more than 3% today"; get in-app notifications (and email if SMTP is configured) |
| **News** | Financial headlines, general or per stock |
| **AI Analyst** | Chat with an assistant: "compare AAPL and MSFT", "how is the market today?" |
| **Settings** | Per-user preferences |

Everything is **private per account**: portfolios, watchlists, alerts and chat history belong to the signed-in user only.

---

## 2. How it is built (architecture)

```
 Browser  ──►  FastAPI server (one container, port 8000)
               ├── serves the React web app   (/ , /login, /static/…)
               ├── REST API                    (/api/v1/…)
               ├── API docs                    (/docs)
               └── health check                (/health)
                         │
          ┌──────────────┼─────────────────────────┐
          ▼              ▼                         ▼
     Database      Market-data providers      AI analyst
  (SQLite / Postgres)  (Yahoo + others)    (LangGraph + Groq)
```

Everything ships in **one Docker image**. The same server delivers the website and the API, so a user needs only Docker Desktop: no Python, no Node, no database install.

### 2.1 Frontend (what the user sees)
- **React** single-page application built with **Vite**, delivered as pre-compiled files.
- Dark "emerald" theme, responsive down to mobile width.
- Pages are loaded on demand (one small file per screen), which keeps first load fast.
- Sign-in token is kept in the browser's session storage; the app renews it automatically before it expires.
- All data comes from the backend's `/api/v1` endpoints. The frontend never talks to Yahoo, Groq or any external service directly.

### 2.2 Backend (the engine)
Built with **Python 3.12 and FastAPI**, organised in layers:

| Layer | Purpose |
| --- | --- |
| **Routers** (`routers/v1/…`) | HTTP endpoints, one module per feature: auth, markets, stocks, compare, portfolios, watchlists, alerts, news, ai, providers |
| **Services** | Business logic: authentication, portfolio maths, watchlists, alert evaluation, login throttling |
| **Providers** | Adapters to outside data sources behind one common interface (see 2.3) |
| **Models / Schemas** | Database tables (SQLAlchemy) and request/response validation (Pydantic) |
| **Core** | Settings, database connection, security, logging |
| **AI** | The research agent and the tools it can call |
| **Jobs** | Background alert evaluation |

**Database tables:** users, revoked tokens, user preferences, watchlists and their stocks, portfolios, holdings, transactions, alerts, notifications, chat sessions and messages, plus news and market caches. Schema changes are managed by **Alembic migrations**. SQLite is the default (zero setup); PostgreSQL is supported through the optional compose file, and a script is included to copy data from SQLite to PostgreSQL.

### 2.3 Market-data layer
- A common `MarketDataProvider` interface defines what any source must offer: quote, history, fundamentals, financials, news, market summary, top movers.
- The main source is **Yahoo Finance** (via `yfinance`), wrapped with **multi-level caching and a circuit breaker**, so repeated requests are fast and a failing source does not hammer or freeze the app.
- Additional adapters exist for **Twelve Data, Finnhub, Marketaux, CoinGecko (crypto), FRED (economic series) and Google News**. They activate when their API keys are supplied.
- **Honesty rule:** if data cannot be fetched, the API returns a clear "unavailable" (HTTP 503) rather than invented or placeholder numbers.

### 2.4 AI analyst
- Uses **Groq** (model `openai/gpt-oss-120b`) orchestrated with **LangGraph**.
- Flow: the question is classified → the relevant **tools** run (stock price, fundamentals, news, compare, market summary, the user's portfolio or watchlist) → the retrieved facts and citations are packed into a context → the model writes the answer from *only* that context.
- The system prompt forbids inventing numbers, sources, timestamps or predictions, requires it to say when data is missing, and forbids personalised buy/sell instructions.
- Vague questions get a fixed capability answer with no model call; the context is capped in size; rate limits are retried once and then reported to the user clearly. Chat history is saved per user.

### 2.5 Alerts
Supported rule types: **price above**, **price below**, **percent change** and **volume spike**. Alerts are evaluated against live quotes, produce notifications, and have de-duplication so a price hovering around its threshold does not spam. An alert re-arms once its condition clears. Email delivery works when SMTP details are configured; otherwise the app says so plainly instead of pretending to send.

### 2.6 Security
- Passwords hashed with **bcrypt**; minimum 12 characters at sign-up.
- **JWT access tokens** (60-minute life) with unique IDs, a revocation list for logout, a refresh endpoint and a hard 12-hour session ceiling.
- **Login throttling**: 5 failed attempts per email + client address, then a 15-minute lockout.
- Every data endpoint requires a valid token; unauthenticated calls get 401.
- Production mode refuses to start without an explicit secret key, PostgreSQL, HTTPS origins and debug off.
- The container runs as a **non-root user** and has a built-in health check.

---

## 3. How it is delivered

| File | Role |
| --- | --- |
| `headstart-1.0.0.tar.gz` | The complete app as a Docker image (~157 MB compressed) |
| `start-headstart.bat` | One-click launcher |
| `docker-compose.windows.yml` | Optional PostgreSQL variant |
| `README-WINDOWS.md`, `IMAGE-INFO.txt` | Instructions and image details |

**What the launcher does when double-clicked**
1. Checks Docker; if it is not running, starts Docker Desktop and waits for the engine.
2. Loads the image from the tarball on first run.
3. Finds a free port (8000, else 8001, 8002 …).
4. Starts the container with its data stored in a persistent Docker volume (`headstart-data`), so accounts and portfolios survive restarts.
5. Waits until `/health` answers, then opens `http://localhost:<port>/login`.

---

## 4. Work done in this engagement

### 4.1 Reviewed and ran the packaged app
Loaded and ran the Docker image, verified the health check, extracted the application code, and tested the site in a browser.

### 4.2 Fixed: "Backend not connected" on `localhost`
- **Symptom:** worked at `127.0.0.1:8000`, failed at `localhost:8000`.
- **Cause:** the compiled frontend had its API address hard-coded as `http://127.0.0.1:8000/api/v1`. From a page opened as `localhost`, that is a different origin, so the browser's CORS check blocked the call (the server's allowed list had no port-8000 origin). Under `127.0.0.1` the page and API shared an origin, so it passed.
- **Fix:** the frontend now calls the relative path `/api/v1`, i.e. whatever address the page was opened from. `localhost`, `127.0.0.1` and any port now work.
- **Cache safety:** changed files were renamed with new names and the HTML page is served with `Cache-Control: no-cache`, so browsers cannot mix old and new files.

### 4.3 Launcher made fully "just works"
Auto-starts Docker Desktop, waits for readiness, picks a free port, waits for the app, and opens straight to `/login`.

### 4.4 Branding
The plain "H" badge was replaced by the supplied **HeadStart — Financial Decision Support System** logo on the sign-in page, sign-up page, mobile layout and sidebar (shown on a white card so it stays readable on the dark theme).

### 4.5 Verification performed
- Health endpoint answers on `localhost`, `127.0.0.1` and `[::1]`.
- Created an account on `localhost:8000`, landed on the dashboard, sidebar logo displayed.
- Simulated a fresh install (removed the local image, ran the `.bat`): tarball loaded, container started healthy, app served the new build.
- The temporary test account was removed afterwards.

---

## 5. Known limitations and notes

- **Live data depends on the network:** Yahoo Finance must be reachable from the container; otherwise quotes show "Unavailable".
- **AI free tier:** the bundled Groq key allows roughly 6–10 questions per minute; beyond that the user sees a rate-limit message.
- **Showcase key:** the Groq key is intentionally embedded for demonstration. It is visible to anyone with the files, so rotate it after the showcase and do not publish the package publicly.
- **Email alerts** need SMTP settings; without them in-app notifications still work.
- **Frontend source:** only the compiled frontend is in the package. The fixes above were applied to the built files; to edit the UI as source code, the original React project is needed.
- **Informational only:** market data may be delayed; the app is not investment advice.
