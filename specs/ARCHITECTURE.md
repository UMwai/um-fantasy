# Fantasy Sports Analytics - System Architecture

## Overview

A data-driven fantasy sports analytics platform with AI-powered projections, automated lineup optimization, and multi-sport support.

## System Diagram

```
                            Fantasy Sports Analytics Platform
+-----------------------------------------------------------------------------------+
|                                                                                   |
|   +------------------+          +------------------+          +---------------+   |
|   |                  |   REST   |                  |   SQL    |               |   |
|   |  React Frontend  | <------> |  FastAPI Backend | <------> |  PostgreSQL   |   |
|   |  (Vite + TS)     |   JSON   |  (Python)        |          |  Database     |   |
|   |                  |          |                  |          |               |   |
|   +------------------+          +------------------+          +---------------+   |
|         :3000                         :8000                                       |
|                                          |                                        |
|                         +----------------+----------------+                       |
|                         |                |                |                       |
|                         v                v                v                       |
|   +------------------+    +------------------+    +------------------+           |
|   |                  |    |                  |    |                  |           |
|   |  ML Pipeline     |    |  Data Ingestion  |    |  Notification    |           |
|   |  (scikit-learn)  |    |  (Celery)        |    |  (Firebase)      |           |
|   |                  |    |                  |    |                  |           |
|   +------------------+    +------------------+    +------------------+           |
|                                  |                                               |
|                    +-------------+-------------+                                  |
|                    |             |             |                                  |
|                    v             v             v                                  |
|            +----------+   +----------+   +----------+                            |
|            | ESPN API |   | Yahoo API|   | NFL API  |                            |
|            +----------+   +----------+   +----------+                            |
|                                                                                   |
+-----------------------------------------------------------------------------------+
```

## Component Architecture

### 1. Frontend (React + TypeScript)

```
frontend/
+-- src/
|   +-- components/
|   |   +-- Dashboard/           # Main dashboard
|   |   +-- PlayerCard/          # Player info display
|   |   +-- MatchupChart/        # Matchup visualizations
|   |   +-- LineupOptimizer/     # Optimization UI
|   |   +-- WaiverAlerts/        # Alert notifications
|   +-- pages/
|   |   +-- Home.tsx             # Landing page
|   |   +-- Players.tsx          # Player search/browse
|   |   +-- Matchups.tsx         # Weekly matchups
|   |   +-- MyTeam.tsx           # User's roster
|   |   +-- Optimize.tsx         # Lineup optimizer
|   |   +-- Trades.tsx           # Trade analyzer
|   +-- services/
|   |   +-- api.ts               # API client
|   |   +-- auth.ts              # Authentication
|   +-- hooks/
|   |   +-- usePlayer.ts         # Player data hooks
|   |   +-- useProjections.ts    # Projection hooks
|   +-- store/
|       +-- playerStore.ts       # Zustand state
|       +-- userStore.ts         # User state
+-- package.json
+-- vite.config.ts
```

**Key Technologies**:
- React 18 with hooks
- TypeScript for type safety
- Vite for fast builds
- Tailwind CSS for styling
- Recharts for visualizations
- Zustand for state management
- TanStack Query for data fetching

### 2. Backend (FastAPI + Python)

```
backend/
+-- app/
|   +-- main.py                  # Application entry
|   +-- api/
|   |   +-- routes/
|   |   |   +-- players.py       # Player endpoints
|   |   |   +-- projections.py   # Projection endpoints
|   |   |   +-- matchups.py      # Matchup endpoints
|   |   |   +-- lineup.py        # Lineup optimizer
|   |   |   +-- trades.py        # Trade analyzer
|   |   |   +-- auth.py          # Authentication
|   |   +-- deps.py              # Dependencies
|   +-- models/
|   |   +-- player.py            # Player models
|   |   +-- team.py              # Team models
|   |   +-- projection.py        # Projection models
|   |   +-- user.py              # User models
|   +-- services/
|   |   +-- projections.py       # Projection service
|   |   +-- optimizer.py         # Lineup optimization
|   |   +-- trade_analyzer.py    # Trade analysis
|   |   +-- data_fetcher.py      # External API client
|   +-- ml/
|   |   +-- models/              # Trained models
|   |   +-- pipeline.py          # ML pipeline
|   |   +-- features.py          # Feature engineering
|   |   +-- train.py             # Training scripts
|   +-- core/
|       +-- config.py            # Configuration
|       +-- security.py          # Auth utilities
+-- requirements.txt
+-- Dockerfile
```

**Key Technologies**:
- FastAPI for REST API
- SQLAlchemy for ORM
- Pydantic for validation
- Celery for background jobs
- Redis for caching/queues

### 3. Database Schema

```sql
-- Players table
CREATE TABLE players (
    id SERIAL PRIMARY KEY,
    external_id VARCHAR(50) UNIQUE,          -- ESPN/Yahoo ID
    name VARCHAR(200) NOT NULL,
    position VARCHAR(10) NOT NULL,           -- QB, RB, WR, TE, K, DEF
    team_id INTEGER REFERENCES teams(id),
    status VARCHAR(20) DEFAULT 'active',     -- active, injured, suspended
    photo_url TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Teams table
CREATE TABLE teams (
    id SERIAL PRIMARY KEY,
    external_id VARCHAR(10) UNIQUE,
    name VARCHAR(100) NOT NULL,
    abbreviation VARCHAR(5) NOT NULL,
    conference VARCHAR(10),
    division VARCHAR(20),
    logo_url TEXT
);

-- Weekly stats
CREATE TABLE player_stats (
    id SERIAL PRIMARY KEY,
    player_id INTEGER REFERENCES players(id),
    season INTEGER NOT NULL,
    week INTEGER NOT NULL,
    pass_yards INTEGER DEFAULT 0,
    pass_tds INTEGER DEFAULT 0,
    rush_yards INTEGER DEFAULT 0,
    rush_tds INTEGER DEFAULT 0,
    receptions INTEGER DEFAULT 0,
    rec_yards INTEGER DEFAULT 0,
    rec_tds INTEGER DEFAULT 0,
    fantasy_points DECIMAL(6,2),
    scoring_type VARCHAR(20) DEFAULT 'ppr',  -- standard, half_ppr, ppr
    UNIQUE(player_id, season, week, scoring_type)
);

-- Projections
CREATE TABLE projections (
    id SERIAL PRIMARY KEY,
    player_id INTEGER REFERENCES players(id),
    season INTEGER NOT NULL,
    week INTEGER NOT NULL,
    projected_points DECIMAL(6,2) NOT NULL,
    floor DECIMAL(6,2),                      -- Low projection
    ceiling DECIMAL(6,2),                    -- High projection
    boom_probability DECIMAL(4,3),           -- P(points > 1.5x projection)
    bust_probability DECIMAL(4,3),           -- P(points < 0.5x projection)
    confidence DECIMAL(4,3),                 -- Model confidence
    source VARCHAR(50) DEFAULT 'internal',   -- internal, espn, yahoo
    scoring_type VARCHAR(20) DEFAULT 'ppr',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(player_id, season, week, source, scoring_type)
);

-- Matchup difficulty
CREATE TABLE matchup_ratings (
    id SERIAL PRIMARY KEY,
    team_id INTEGER REFERENCES teams(id),
    season INTEGER NOT NULL,
    week INTEGER NOT NULL,
    position VARCHAR(10) NOT NULL,           -- QB, RB, WR, TE
    vs_team_id INTEGER REFERENCES teams(id),
    difficulty_rating DECIMAL(4,2),          -- 1-10 scale
    points_allowed_avg DECIMAL(6,2),
    rank INTEGER,                            -- 1-32 rank
    UNIQUE(team_id, season, week, position)
);

-- User rosters
CREATE TABLE user_rosters (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    league_id VARCHAR(100),
    platform VARCHAR(20),                    -- espn, yahoo
    player_id INTEGER REFERENCES players(id),
    roster_position VARCHAR(10),             -- QB, RB1, RB2, WR1, etc.
    is_starter BOOLEAN DEFAULT FALSE,
    season INTEGER,
    UNIQUE(user_id, league_id, player_id, season)
);

-- Users
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    name VARCHAR(200),
    notification_preferences JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Alerts
CREATE TABLE alerts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    player_id INTEGER REFERENCES players(id),
    alert_type VARCHAR(50),                  -- injury, waiver, news
    message TEXT,
    is_read BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 4. ML Pipeline Architecture

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  Feature Store   | --> |  Model Training  | --> |  Model Registry  |
|  (PostgreSQL)    |     |  (scikit-learn)  |     |  (MLflow)        |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
         |                       |                        |
         v                       v                        v
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  Historical      |     |  XGBoost         |     |  Model Serving   |
|  Stats + Context |     |  Gradient Boost  |     |  (FastAPI)       |
|                  |     |  Random Forest   |     |                  |
+------------------+     +------------------+     +------------------+
```

**ML Features**:

| Category | Features |
|----------|----------|
| Player | Recent performance, season avg, career avg, age, snap % |
| Team | Offensive line ranking, pace of play, red zone efficiency |
| Matchup | Opponent rank vs position, implied team total, spread |
| Situational | Home/away, weather, rest days, injuries to teammates |
| Advanced | Target share, air yards, snap trends, route efficiency |

**Model Ensemble**:
- XGBoost for gradient boosting
- Random Forest for stability
- Linear regression as baseline
- Weighted average based on recent accuracy

### 5. Lineup Optimizer

**Algorithm**: Integer Linear Programming (using PuLP)

```python
# Optimization objective
maximize: sum(projected_points[player] * selected[player])

# Constraints
subject to:
    sum(salary[player] * selected[player]) <= salary_cap
    sum(selected[player] at position) == roster_requirements[position]
    sum(selected[player]) == total_roster_size
    selected[player] in {0, 1}  # Binary decision
```

**Optimizer Features**:
- Multi-lineup generation (diverse outputs)
- Exposure limits (avoid over-correlating)
- Stack detection (QB + receivers)
- Lock/exclude players
- Custom scoring rules

### 6. Data Ingestion Pipeline

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  Schedule        | --> |  Fetch Data      | --> |  Transform       |
|  (Celery Beat)   |     |  (API Clients)   |     |  (Pandas)        |
|                  |     |                  |     |                  |
+------------------+     +------------------+     +------------------+
                                                          |
                    +-------------------------------------+
                    |
                    v
         +------------------+     +------------------+
         |                  |     |                  |
         |  Load to DB      |     |  Trigger ML      |
         |  (SQLAlchemy)    |     |  (Celery Task)   |
         |                  |     |                  |
         +------------------+     +------------------+
```

**Data Sources**:

| Source | Data Type | Frequency |
|--------|-----------|-----------|
| ESPN Fantasy API | Projections, scores | Hourly |
| Yahoo Fantasy API | Projections, scores | Hourly |
| NFL Stats API | Official stats | Post-game |
| Injury Reports | Player status | Real-time |
| Weather API | Game conditions | Daily |
| Vegas Lines | Spreads, totals | Multiple daily |

---

## API Design

### Player Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/players | List players with filters |
| GET | /api/players/{id} | Get player details |
| GET | /api/players/{id}/stats | Get player stats history |
| GET | /api/players/{id}/projections | Get player projections |
| GET | /api/players/trending | Get trending players |

### Projection Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/projections/weekly | Week's projections |
| GET | /api/projections/ros | Rest of season rankings |
| GET | /api/projections/{player_id} | Single player projection |

### Lineup Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /api/lineup/optimize | Generate optimal lineup |
| POST | /api/lineup/compare | Compare two lineups |
| GET | /api/lineup/saved | User's saved lineups |

### Trade Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /api/trades/analyze | Analyze trade fairness |
| GET | /api/trades/suggestions | Suggested trades |

---

## Notification System

**Push Notification Flow**:
```
1. News event detected (injury, trade, etc.)
2. Event published to message queue (Redis)
3. Worker processes event
4. Filter users with relevant players
5. Send notifications via Firebase Cloud Messaging
6. Log notification in database
```

**Alert Types**:
- Injury updates (immediate)
- Waiver wire pickups (immediate)
- Game day reminders (scheduled)
- Trade suggestions (daily)
- Weekly projections update (Wednesday)

---

## Security

### Authentication
- JWT tokens for API access
- OAuth 2.0 for ESPN/Yahoo login
- Password hashing with bcrypt

### Authorization
- Role-based access control
- Free vs Premium tiers
- Rate limiting per user

### Data Protection
- HTTPS everywhere
- Database encryption at rest
- No sensitive PII stored

---

## Scalability Plan

### Phase 1 (100 users)
- Single server deployment
- PostgreSQL on same host
- Redis for caching

### Phase 2 (1,000 users)
- Separate database server
- Redis cluster
- CDN for static assets

### Phase 3 (10,000 users)
- Kubernetes deployment
- Read replicas for database
- Microservices architecture
- ML model serving cluster

---

## Monitoring

### Health Checks
- `/health` endpoint for all services
- Database connectivity check
- External API status

### Metrics
- API response times (P50, P95, P99)
- Projection accuracy scores
- User engagement metrics
- ML model performance

### Alerting
- Error rate threshold alerts
- API downtime alerts
- Data freshness alerts

---

**Last Updated**: 2024-12-30
**Architecture Version**: 1.0
