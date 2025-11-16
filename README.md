# pfman

**Personal / Family Asset Operating System**

A proactive portfolio management system that keeps a clean, auditable record of all assets over time, understands their relationships, and uses analytics + LLM-based reasoning to provide insights and suggestions.

## Core Capabilities

- **Portfolio Modeling**: Track listed securities, cash, alternatives (real estate, private equity, collectibles), across multiple accounts and portfolios
- **Analytics & Risk**: Time-weighted returns, volatility, concentration metrics, stress tests, and scenario analysis
- **Investment Policy**: Define target allocations and risk limits, track deviations, generate rebalancing suggestions
- **Proactive Agent**: Scheduled activities (daily price updates, weekly risk summaries, monthly rebalance checks) with LLM-generated insights
- **News & Impact Tracking**: Monitor market events and their impact on your holdings

## Architecture

- **Frontend**: Web SPA (React/Vue/Svelte with TypeScript)
- **Backend**: FastAPI (Python) or NestJS/Express (TypeScript)
- **Data**: Neo4j graph database for relationships and metadata
- **Analytics**: Python-based analytics services (pandas/NumPy)
- **LLM Integration**: Analysis-agent microservice for AI-powered insights

## Local Development

The system uses Docker Compose for local deployment:

```bash
docker-compose up
```

Services include:
- `pfman-api` - Backend API
- `pfman-web` - Frontend interface
- `neo4j` - Graph database
- `analysis-agent` - LLM-based analysis service (optional)

## Roadmap

**Phase 0** - Skeleton: Basic setup with core data model
**Phase 1** - Manual portfolio tool with CRUD operations and basic analytics
**Phase 2** - Policies, risk analysis, and scenario modeling
**Phase 3** - News integration, LLM insights, proactive recommendations
**Phase 4** - Cloud deployment and broker/bank connectors

## Philosophy

This is not a passive tracking tool but an active "personal CIO with a graph brain" that regularly analyzes your portfolio, flags risks, and proposes ideas based on your investment policy and market conditions.
