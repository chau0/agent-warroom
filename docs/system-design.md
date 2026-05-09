# System Design: Agent Warroom

## Context

Agent Warroom is a multi-agent AI debate platform where indie hackers submit an app idea and receive a structured **Build / Kill / Modify** verdict from five specialized AI agents. The design targets a solo-builder deployment on a single Ubuntu VPS, keeps LLM costs bounded, and produces output actionable enough to change what the user does next.

---

## Solution Architecture

```
User Browser
    │ HTTPS
    ▼
Nginx / Caddy  (reverse proxy, TLS, gzip)
    │ localhost:3000
    ▼
Next.js Node Server
    ├── App Router  (pages & layouts)
    ├── Route Handlers  (API endpoints)
    │       ├── POST /api/warroom/run
    │       └── GET  /api/warroom/:id
    ├── SSE stream  (live debate updates → browser)
    └── Agent Orchestrator
            ├── Round 1: parallel fan-out  (Founder | Engineer | Growth | Skeptic)
            ├── Round 2: parallel cross-critique
            └── Round 3: Judge synthesis
                    │
                    ▼
             LLM Provider Wrapper
             (Claude / GPT / Gemini — server-side only)
                    │
                    ▼
             SQLite  (WAL mode)
             ├── warroom_sessions
             ├── warroom_messages
             └── agent_configs
                    │
                    ▼
             Daily cron backup
```

---

## Main Components

### 1. Frontend — Next.js App Router + React + Tailwind CSS

| Page | Responsibility |
|---|---|
| **Homepage** | Headline, idea textarea, example idea buttons, "Start Warroom" CTA |
| **Debate Room** | Agent cards (Founder / Engineer / Growth / Skeptic / Judge), round indicator (Opinion → Crossfire → Verdict), typing animation, per-agent score badges |
| **Report Page** | Shareable URL at `/report/:id` — final decision badge, score bar, full transcript, export to Markdown button |

### 2. API Layer — Next.js Route Handlers

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/warroom/run` | POST | Create session, trigger orchestrator, stream results via SSE |
| `/api/warroom/:id` | GET | Return saved session with full transcript and verdict |

All LLM calls happen server-side. API keys never reach the browser.

### 3. Agent Orchestrator

Controls the fixed 3-round debate flow:

- **Round 1** — `Promise.all` fan-out to all four debate agents. Each agent receives only the user idea + scoring rubric. No cross-visibility; prevents groupthink.
- **Round 2** — `Promise.all` fan-out. Each agent receives the Round 1 full transcript and must critique, agree, disagree, or revise its position.
- **Round 3** — Judge agent receives the full debate transcript and returns the structured final verdict.

If one agent call fails, the orchestrator saves a partial result, retries once, then continues rather than failing the entire session.

### 4. Agent Layer

Five agents with fixed roles:

| Agent | Focus |
|---|---|
| Founder | Pain urgency, monetization, founder fit, MVP speed |
| Engineer | Technical feasibility, architecture risk, time-to-MVP |
| Growth | Acquisition channels, distribution, virality, pricing |
| Skeptic | Demand risk, competition, unclear buyer, weak willingness-to-pay |
| Judge | Synthesizes debate → final decision, score, next action |

Agent identity and role prompts are stored in the `agent_configs` database table, making them tunable without a code deploy.

All agents return structured JSON. The Judge returns an extended contract with `biggest_risk`, `next_action`, `mvp_scope`, and `first_user_strategy`.

### 5. LLM Provider Wrapper

A thin abstraction over Claude / GPT / Gemini:

- Enforces max input/output token budgets per call
- Requires JSON-mode output; triggers a single structured retry on malformed responses
- Logs per call: provider, model, token usage, latency, error reason

### 6. Database — SQLite (WAL mode)

Three tables:

| Table | Purpose |
|---|---|
| `warroom_sessions` | One row per debate; tracks status, final decision, final score |
| `warroom_messages` | One row per agent per round; stores JSON output in `score_json` |
| `agent_configs` | Agent role prompts; editable at runtime |

- WAL mode for better concurrent reads on a live web app
- UUIDs generated in the application layer, stored as `text`
- Database file lives outside the repo at `/opt/agent-warroom/data/warroom.sqlite`
- ORM: Drizzle ORM (lightweight, SQLite-native, typed migrations)

### 7. Infrastructure

| Component | Choice |
|---|---|
| VPS | Ubuntu — single long-lived Node.js process |
| Reverse Proxy | Nginx or Caddy — HTTPS, domain routing, gzip/brotli |
| Process Manager | systemd or PM2 — survives SSH disconnect and server reboot |
| Backup | Daily cron — copies SQLite file to `/opt/agent-warroom/backups/` |

---

## Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Frontend | Next.js 14+ App Router + React + Tailwind CSS | One codebase for UI and API; fast MVP iteration |
| Backend | Next.js Route Handlers (Node.js) | No separate service; co-located with frontend |
| Database | SQLite (WAL mode) | Zero infra cost, trivial backup, sufficient for MVP traffic |
| ORM | Drizzle ORM | Lightweight, typed, SQLite-native, simple migrations |
| LLM | Claude API (primary) + wrapper for GPT/Gemini | Quality output; abstraction keeps provider swappable |
| Streaming | Server-Sent Events (SSE) | Simpler than WebSockets for one-directional server→client streaming |
| Process Manager | systemd or PM2 | Keeps server alive across reboots |
| Reverse Proxy | Nginx or Caddy | HTTPS termination, routing, static asset caching |
| Deployment | `git pull` + `npm run build` + `systemctl restart` | Zero CI/CD dependency for solo builder |
| Backup | cron + shell script | Daily SQLite file copy; no managed service needed |

---

## Main Design Decisions

### 1. SQLite over a managed database
SQLite is sufficient for a solo-builder MVP. Backup is a single file copy, there is no database server to maintain, and cost is zero. WAL mode handles the read/write concurrency of a small web app. Migration to Postgres is straightforward via Drizzle if traffic demands it later.

### 2. Next.js monorepo — no separate backend service
A single Next.js codebase handles both the React UI and the API route handlers. Eliminates deployment coordination between two services. Appropriate until team size or traffic justifies splitting.

### 3. Server-Sent Events over WebSockets
SSE is one-directional (server → client), which matches the debate streaming pattern exactly. Native in Next.js Route Handlers. No persistent bidirectional socket needed for the MVP.

### 4. Parallel Round 1 execution with agent isolation
All four debate agents run in parallel and see only the user's idea. This prevents groupthink and cuts Round 1 wall-clock time from ~4× serial to ~1× the slowest agent.

### 5. JSON output contracts for all agents
Structured JSON from every LLM call makes UI rendering reliable and avoids fragile text parsing. Malformed output triggers one retry; after that the agent is marked failed and the session continues with partial results.

### 6. Agent prompts in the database
Role prompts live in `agent_configs`, not hardcoded in source. Allows tuning agent behavior — tone, rubric, focus areas — without a code deploy or server restart.

### 7. Server-side LLM calls only
All API keys are server environment variables. The client browser never contacts LLM providers. This is a non-negotiable security boundary.

### 8. Cost bounding by design
- Idea input capped at a character limit (e.g. 1 000 chars)
- Transcript trimmed to a token budget before Round 2 and Judge calls
- Cheaper/faster models can be used for critique rounds; the Judge uses the highest-quality model
- Daily free-tier debate limit enforced by session count per anonymous ID or logged-in user

---

## Data Flow

```
1.  User submits idea → POST /api/warroom/run
2.  Server creates session row in SQLite  (status = 'running')
3.  Orchestrator fans out Round 1 to 4 agents in parallel
4.  Each agent result → warroom_messages row  (round = 1)
5.  Results streamed to browser via SSE as they arrive
6.  Round 2 fan-out with full Round 1 transcript
7.  Critique results → warroom_messages rows  (round = 2) + streamed
8.  Judge receives full transcript → structured verdict
9.  Session updated  (status = 'done', final_decision, final_score, final_summary)
10. Frontend renders verdict; /report/:sessionId is publicly shareable
```

---

## Deployment Shape

```
/opt/agent-warroom/
├── app/                   ← Next.js build output
├── .env.production
├── data/
│   └── warroom.sqlite
├── backups/
│   └── warroom-YYYY-MM-DD.sqlite
└── scripts/
    ├── deploy.sh
    └── backup.sh
```

**Deploy flow:**
```
git pull origin main
npm install
npm run build
npm run db:migrate
systemctl restart agent-warroom
```

---

## Verification Checklist

- [ ] Run one full idea through local dev server; all three rounds complete and verdict renders
- [ ] Confirm `warroom_sessions` and `warroom_messages` rows written to SQLite after each round
- [ ] Open `/report/:id` in an incognito window; transcript and verdict visible without re-running
- [ ] Simulate one agent timeout; app shows partial results rather than crashing the session
- [ ] Check server logs for per-call token usage and latency entries
- [ ] Run backup script manually; SQLite file copies to `backups/` correctly
