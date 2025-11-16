You’re basically describing “a personal CIO with a graph brain”. Nice project.

Quick style note: “not only being a passive tool but **which** has some regularly scheduled activities” would flow better as “but **that** has” or “but instead has”.

I’ll structure this as: vision → main capabilities → architecture & data → UI → local/cloud & migration → phased roadmap.

---

## 1. Vision and scope

**pfman = personal / family asset OS** that:

* Keeps a clean, auditable record of all assets over time
* Understands relationships: sectors, countries, currencies, counterparties, correlations
* Runs **analytics + LLM-based reasoning** over that graph
* Is **proactive**: flags risks, suggests rebalancing, proposes new ideas on a schedule

Assume **single primary user + family** to start; multi-user comes later.

---

## 2. Core capabilities

### 2.1 Portfolio model & data entry

* Assets:

  * Listed: stocks, ETFs, mutual funds, bonds
  * Cash & term deposits
  * Alternative: real estate, private equity, startups, collectibles, pensions
  * Synthetic: options, futures, structured products (later phase)
* Structures:

  * Accounts (brokerage, savings, pension, company, joint)
  * Portfolios (logical groupings: “Long-term”, “Kids”, “High risk”)
* Inputs:

  * Manual entry via web UI
  * CSV import (simple but powerful)
  * Optional connectors (Phase 3+): broker APIs, open banking, etc.
* Time dimension:

  * Transactions: buy/sell, dividends, fees, transfers
  * Snapshots: “Portfolio as of YYYY-MM-DD with per-asset values”

### 2.2 Analytics & risk

Initial set:

* Per-asset and portfolio-level:

  * Time-weighted return, money-weighted return
  * Volatility, max drawdown, Sharpe / Sortino
  * Concentration metrics (e.g. top 10 positions %, top sector %, top country %)
* Factor-style exposures (approximate at first):

  * Region, sector, currency, asset class
  * ESG / “brown/green” tagging if wanted
* Risk views:

  * Simple stress tests: “-30% in equities”, “+300bp rates”, “SEK -10% vs USD/EUR”
  * Scenario P&L estimates based on asset betas / elasticities
* Later: VaR/ES, correlation matrix, factor regression, Monte Carlo.

### 2.3 Investment advice & policy comparison

Model “advice” as **target portfolios / policy constraints**:

* Policy objects:

  * Target allocation ranges by asset class, sector, region, currency
  * Risk limits (max volatility, max single-position %, max illiquid assets %)
  * Rules of thumb (min 6 months expenses in cash, etc.)
* Features:

  * “Distance to policy”: show over/underweights vs target ranges
  * Backtesting vs simple benchmark (e.g. global 60/40, MSCI ACWI, etc.)
  * Multiple advice profiles (e.g. “Bank A”, “My own”, “Boglehead 3-fund”)

### 2.4 News & market impact tracking

* Map each asset to:

  * Ticker, ISIN, exchange
  * Company, sector, country
* Daily job:

  * Pull market data & news headlines for:

    * Owned assets
    * Closely related assets (same sector/country/index)
  * Flag:

    * Large price moves (e.g. > 5% in a day)
    * Big events: earnings, rating changes, macro news, regulatory items
* Present:

  * “Impact dashboard”:

    * List events with estimated impact on portfolio (value at risk, narrative summary)
  * Optional LLM summarization:

    * “Explain the last week’s events and their impact on my portfolio in 5 bullet points.”

### 2.5 Scenario analysis & LLM tools

Scenarios as first-class objects:

* Scenario definition:

  * Type: macro shock, sector shock, policy change, FX move, idiosyncratic event
  * Parameter set: e.g. “Equities -25% globally, SEK -10% vs USD, rates +150bp”
  * Narrative: free text describing the story
* Engine:

  * Deterministic “what-if” calculator using:

    * Asset class betas
    * FX exposure
    * Duration for interest-rate sensitive assets
  * Optional Monte Carlo around that scenario
* LLM integration:

  * Given scenario + portfolio graph + current news:

    * Generate a qualitative risk report
    * Suggest defensive / opportunistic actions
  * Allow “ad hoc” prompts:

    * “Given my portfolio and recent events, what are the 3 main tail risks over the next 12 months?”

### 2.6 Proactive agent & scheduling

Scheduled activities (configurable):

* Daily:

  * Update prices, FX rates, basic news headlines
  * Check breach of critical thresholds (margin, position size, stop-loss)
* Weekly:

  * Risk summary email/report
  * “Top 3 things to pay attention to”
* Monthly/quarterly:

  * Full rebalance check vs policy
  * “New ideas” report (LLM-generated based on constraints & universe)
* Interaction style:

  * Dashboard notifications + email (or local notification initially)
  * “Tasks” list (“Consider trimming XYZ”, “Rebalance EM from 8% to 5%”)

---

## 3. Architecture overview

### 3.1 High-level

* **Frontend**: Web SPA

  * React / Vue / Svelte; TypeScript
* **Backend API**:

  * REST + WebSocket (for live updates later)
  * Python (FastAPI) or TypeScript (NestJS/Express) – Python has advantage for quant libs
* **Data layers**:

  * Primary: **Neo4j** for relationships + metadata
  * Optionally (recommended medium term): Postgres/TimescaleDB for ledger & time series
* **Analytics services**:

  * Stateless services using pandas/NumPy (Python) or a dedicated analytics microservice
* **LLM / external tools**:

  * Separate “analysis-agent” service calling LLM API, news APIs, market data APIs.
* **Scheduling**:

  * Local: APScheduler / cron jobs in backend
  * Cloud: managed scheduler (Cloud Run + Cloud Scheduler / ECS scheduled tasks / etc.)

### 3.2 Why not only Neo4j?

Graph DB is perfect for:

* Asset relationships (sector, country, issuer, index membership)
* Dependency graphs (ETF → underlying holdings; structured product → underlyings)
* Propagating shocks through the network

But less ergonomic for:

* Ledger-like transaction records
* Time-series analytics (pricing, performance, risk)

So:

* **Phase 1** (to keep it simple): use **Neo4j only**, but design model so that:

  * Time series are stored as nodes/relationships with date properties
  * You can later sync critical parts to a relational store without breaking semantics
* **Phase 2+**: introduce Postgres or TimescaleDB for:

  * Transactions
  * Daily price & performance series
  * Materialized analytics tables
  * Keep Neo4j for the “knowledge graph” layer

---

## 4. Neo4j data model (Phase 1)

### 4.1 Node types

* `User`
* `Account` (broker / bank / pension)
* `Portfolio` (logical grouping)
* `Asset`

  * Properties: type, ticker, ISIN, name, currency, listing, etc.
* `HoldingSnapshot`

  * Properties: date, quantity, value_ccy, price_ccy
* `Transaction`

  * Properties: date, type (buy/sell/dividend/fee/transfer), quantity, price, fees, tax
* `PricePoint`

  * Properties: date, price, currency, volume (optional)
* `Market`
* `Sector`
* `Country`
* `Issuer` (company/fund provider/etc.)
* `Policy` / `TargetAllocation`
* `Scenario` and `ScenarioRun`
* `NewsItem`
* `Alert`
* `Tag` (flexible tagging of anything)

### 4.2 Relationships

Examples:

* `(:User)-[:OWNS]->(:Account)`
* `(:Account)-[:CONTAINS]->(:Portfolio)`
* `(:Portfolio)-[:HOLDS]->(:Asset)`
* `(:HoldingSnapshot)-[:OF_ASSET]->(:Asset)`
* `(:HoldingSnapshot)-[:IN_PORTFOLIO]->(:Portfolio)`
* `(:HoldingSnapshot)-[:AT_TIME]->(:TimePoint)` (if you use explicit time nodes)
* `(:Transaction)-[:ON_ASSET]->(:Asset)`
* `(:Transaction)-[:IN_ACCOUNT]->(:Account)`
* `(:Asset)-[:ISSUED_BY]->(:Issuer)`
* `(:Asset)-[:LISTED_ON]->(:Market)`
* `(:Asset)-[:IN_SECTOR]->(:Sector)`
* `(:Asset)-[:IN_COUNTRY]->(:Country)`
* `(:Asset)-[:TRACKS_INDEX]->(:Asset)` (index ETF → index)
* `(:Asset)-[:CORRELATED_WITH {rho: …}]->(:Asset)` (optional, can be computed)
* `(:Policy)-[:TARGETS]->(:Portfolio)`
* `(:Policy)-[:HAS_TARGET]->(:TargetAllocation {class:'Equity', min:0.4, max:0.6, …})`
* `(:ScenarioRun)-[:APPLIES_TO]->(:Portfolio)`
* `(:ScenarioRun)-[:RESULT_OF]->(:Scenario)`
* `(:NewsItem)-[:MENTIONS]->(:Asset|:Issuer|:Sector)`
* `(:Alert)-[:ABOUT]->(:Portfolio|:Asset)`

### 4.3 Time modeling choices

Option A (simpler):

* `HoldingSnapshot` nodes with a `date` property
* Query by date ranges using property filters

Option B (more “graphy”):

* `(:Day {date:'YYYY-MM-DD'})`
* `(:HoldingSnapshot)-[:ON_DAY]->(:Day)`
* Easier to group by date, run period operations, etc.

I’d start with **A** for speed, keep B in mind if you later want heavy graph-based time navigation.

---

## 5. Analytics & risk engine design

Create a dedicated **analytics module** in the backend:

* Input:

  * Portfolio ID, date / date range
  * Optional: scenario parameters, policy constraints
* Steps:

  1. Fetch holdings & price series from DB
  2. Normalize to base currency
  3. Compute daily portfolio value series
  4. Derive metrics (returns, volatility, drawdowns, etc.)
  5. Map exposures via Neo4j (asset → issuer → sector/country/currency)
  6. Return a structured result (JSON) for the UI + store summary nodes in Neo4j for caching
* Design principle:

  * Analytics functions are **pure**: no side effects except optional caching
  * Keeps it easy to test and to move into separate microservice later

---

## 6. LLM & external integration

### 6.1 “analysis-agent” microservice

* Service that exposes high-level endpoints:

  * `/summarize-weekly-events`
  * `/evaluate-policy-breaches`
  * `/propose-rebalancing`
  * `/scenario-report`
* Internal workflow:

  1. Backend produces a **compact, structured JSON** of the current situation:

     * Top N holdings with weights
     * Recent changes
     * Key risk metrics
     * Policy rules
     * Short list of notable news items
  2. Sends this + a well-crafted prompt to the LLM API
  3. LLM returns:

     * Bullet-pointed analysis
     * List of suggested actions with rationale and confidence
  4. Service stores results as:

     * `(:Insight)` and `(:SuggestedAction)` nodes in Neo4j.

### 6.2 Important design choices

* **LLM is advisory, not authoritative**:

  * Always keep quantitative checks separate
* **Prompt templates** stored in DB or config:

  * Easy to evolve, test, and switch
* **Clear user controls**:

  * Risk preferences
  * Investment universe allowed (e.g. “only ETFs”, “no single-stock picks”)
  * Max trade frequency

---

## 7. Web interface design

Main screens:

1. **Dashboard**

   * Net worth over time chart
   * Current allocation by asset class / region / currency
   * Today’s P&L and biggest movers
   * Open alerts & suggested actions

2. **Holdings & accounts**

   * Table: asset, quantity, value, % of portfolio, gain/loss
   * Filters by account, portfolio, tag
   * Drill-down to asset detail:

     * Price chart
     * News section
     * Risk & exposure breakdown

3. **Transactions**

   * Ledger view per account
   * Import/Export CSV
   * Simple “add transaction” form

4. **Risk & analytics**

   * Charts for volatility, drawdowns, diversification
   * Concentration metrics
   * Stress tests / scenario sliders

5. **Advice & policy**

   * Define/edit policy
   * Visualization of deviations:

     * “You’re 12% over your max EM equity allocation”
   * Rebalancing proposals (list of trades)

6. **News & events**

   * Timeline of relevant news items
   * Portfolio-weighted impact indicators
   * “Explain the last week” LLM button

7. **Scenarios**

   * List of saved scenarios
   * Run scenario on selected portfolio + view quantitative + narrative results

8. **Settings**

   * Risk profile
   * Scheduling preferences
   * Data sources & API keys
   * Backup/export

---

## 8. Local deployment & future cloud hosting

### 8.1 Local first

Use **docker-compose** from day one:

* Services:

  * `pfman-api` (backend)
  * `pfman-web` (frontend)
  * `neo4j`
  * `analysis-agent` (optional at first, can be stubbed)
* Benefits:

  * Same topology locally and in cloud
  * Easy to spin up / tear down
* Data:

  * Neo4j volume mounted to host for persistence & backup

### 8.2 Cloud later

When moving to cloud, reuse containers:

* Options:

  * GCP: Cloud Run for stateless services, Neo4j Aura or Neo4j in GKE
  * AWS: ECS/Fargate + Neo4j on EC2 or managed Aura
  * Azure: Container Apps + Neo4j Aura
* Steps for migration:

  1. Harden config:

     * Secrets via environment variables / secret manager
     * Separate test/prod environments
  2. Move Neo4j:

     * For Aura: export local DB (`neo4j-admin dump`) and import into cloud instance
     * For self-hosted cloud Neo4j: restore from backup or use dump/load
  3. Point backend to new DB via config
  4. Add HTTPS, auth, rate limits
  5. Add logging/monitoring (OpenTelemetry, structured logs)

---

## 9. Migration planning & extensibility

Design for **smooth evolution**:

1. **Schema versioning**

   * Keep a `schema_version` and migration scripts (e.g. Python scripts for Neo4j)
   * Define migration steps in code, not just ad hoc changes

2. **Adding Postgres/Timescale (Phase 2)**

   * Introduce a `Storage` abstraction in backend:

     * `GraphRepository` for Neo4j
     * `TimeSeriesRepository` for relational
   * Start by mirroring:

     * Keep Neo4j as the “source” and replicate key series into Postgres
   * Over time, move heavy analytics to relational store

3. **Broker / bank connectors**

   * Design connector interfaces now:

     * Methods like `fetch_transactions(since)`, `fetch_positions()`
   * Implement “manual CSV connector” first
   * Later add specific APIs (Interactive Brokers, Nordnet, etc.)

4. **Auth & multi-user**

   * Initially: single-user with simple auth
   * Keep user ID in all top-level entities from day one
   * Later: add OAuth (Google, Apple, etc.) and proper multi-tenant isolation

---

## 10. Phased roadmap (concrete)

### Phase 0 – Skeleton (1–2 weeks of focused work)

* Set up repo, docker-compose, baseline:

  * Neo4j
  * Simple backend with healthcheck
  * Simple frontend “Hello pfman”
* Define core data model in Neo4j and wire minimal CRUD:

  * Assets, Accounts, Portfolios, HoldingSnapshots, Transactions

### Phase 1 – Solid manual portfolio tool

* Full UI for:

  * Entering accounts, assets, portfolios
  * Recording transactions
  * Viewing current holdings & performance
* Basic risk & analytics:

  * Portfolio value over time
  * Simple concentration metrics
* Basic backup/restore (Neo4j dump)

### Phase 2 – Policies, risk, and scenarios

* Add Policy model and UI
* Implement:

  * Policy deviation views
  * Simple stress tests
* Add Scenario & ScenarioRun nodes and engine
* Introduce simple scheduling (daily/weekly cron jobs locally) for:

  * Recompute metrics
  * Generate reports

### Phase 3 – News, LLM, proactive insights

* Integrate a market data + news API (or stub with manual input to start)
* Add NewsItem nodes and link them to assets
* Implement `analysis-agent`:

  * Weekly summary
  * Scenario narrative
  * Rebalancing suggestions
* UI for:

  * Viewing AI-generated insights
  * Accepting/ignoring suggested actions

### Phase 4 – Cloud & connectors

* Deploy to a chosen cloud as a private instance
* Add at least one real connector (e.g., a broker or bank API you actually use)
* Harden security, auth, monitoring

---

If you want, next step could be: I sketch a concrete **repo layout + initial Neo4j schema (Cypher)** + a minimal `docker-compose.yml` that matches this plan.
