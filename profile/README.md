<p align="center">
  <img src="https://github.com/user-attachments/assets/37be4bfc-60e5-496f-bd02-abe0adf3979c" width="300">
</p>

# osu!somtum

osu!somtum is an [osu!](https://osu.ppy.sh) private server from Thailand. Services communicate via a shared MySQL database and Redis cache, all hosted on OVH and fronted by Cloudflare.

**Website:** [somtum.fun](https://somtum.fun)

## Service Architecture

```mermaid
flowchart TB
    subgraph Clients["External Clients"]
        osu(["osu! Game Client"])
        web(["Web Browser"])
        dclient(["Discord"])
    end

    subgraph CF["Cloudflare (DNS · CDN · DDoS · SSL)"]
        cf{{"Proxy & Cache"}}
    end

    subgraph OVH["OVH VPS — Podman Containers"]
        ng{{"Caddy — Reverse Proxy & Rate Limiting"}}

        subgraph Public["Public Services"]
            bancho["bancho.py<br/>(Python/FastAPI) :5001"]
            gukarkka["gukarkka<br/>(Next.js 14) :3000"]
            payments["payments-service<br/>(Python/FastAPI) :8001"]
        end

        subgraph Internal["Internal Services"]
            circlecore["Circlecore-somtum<br/>(Python/FastAPI) :8555"]
        end

        subgraph Bots["Bots"]
            bot["Discord-Bot-Somtum<br/>(discord.py)"]
        end

        subgraph Data["Data Stores"]
            mysql[("MySQL 9.4<br/>Shared Database")]
            redis[("Redis 8.2<br/>Cache & Sessions")]
            files[("Static Files<br/>Avatars & Replays")]
        end
    end

    subgraph ExternalAPIs["External APIs"]
        osuapi["osu! API v1/v2"]
        discordwh["Discord Webhooks"]
        truemoney["TrueMoney / PromptPay"]
        stripe["Stripe"]
    end

    %% Client connections
    osu -->|"c.somtum.fun (HTTP, direct)"| ng
    web --> cf --> ng
    dclient <--> bot

    %% Caddy routing
    ng -->|"somtum.fun"| gukarkka
    ng -->|"session / api .somtum.fun"| bancho
    ng -->|"a.somtum.fun"| files

    %% Inter-service HTTP
    gukarkka -->|"player & session data"| bancho
    bancho -->|"replay analysis"| circlecore
    payments -->|"grant_donator"| bancho
    bot -->|"player actions"| bancho
    bot -->|"replay checks"| circlecore

    %% External APIs
    bancho --> osuapi
    bancho --> discordwh
    circlecore --> discordwh
    payments --> truemoney
    payments --> stripe
    bot --> discordwh

    %% Data stores
    bancho --> mysql & redis
    gukarkka --> redis
    payments & bot --> mysql

    %% Styling
    classDef client fill:#e1f5fe,stroke:#01579b,color:#01579b
    classDef proxy fill:#fff3e0,stroke:#e65100,color:#e65100
    classDef public fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef internal fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef data fill:#fce4ec,stroke:#c2185b,color:#880e4f
    classDef external fill:#eceff1,stroke:#546e7a,color:#37474f
    classDef bot fill:#e3f2fd,stroke:#1565c0,color:#1565c0

    class osu,web,dclient client
    class cf,ng proxy
    class bancho,gukarkka,payments public
    class circlecore internal
    class mysql,redis,files data
    class osuapi,discordwh,truemoney,stripe external
    class bot bot
```

## Services

| Service | Language | Purpose |
|---|---|---|
| **bancho.py** | Python/FastAPI | Core game server (osu! protocol, scoring, multiplayer) |
| **gukarkka** | Next.js 14/TypeScript | Web frontend — profiles, leaderboards, beatmap browser |
| **Discord-Bot-Somtum** | Python/discord.py | Community bot — stats, moderation, beatmap workflow |
| **Circlecore-somtum** | Python/FastAPI | Anti-cheat — replay analysis & similarity detection |
| **payments-service** | Python/FastAPI | Donation processing (TrueMoney, PromptPay, Stripe) |

## Communication Patterns

- **Shared Database**: bancho.py, Discord bot, and payments-service all connect to the same MySQL instance
- **Redis**: bancho.py for sessions and cache; gukarkka for caching
- **HTTP REST**: All inter-service communication

### Inter-Service HTTP Calls

| Caller | Target | Purpose |
|---|---|---|
| gukarkka | bancho.py | Player sessions, scores, leaderboards |
| bancho.py | Circlecore-somtum | Queue replay for analysis after score submit |
| payments-service | bancho.py | `POST /internal/grant_donator` — sync donator status |
| Discord-Bot-Somtum | bancho.py | Player management (restrict, whitelist, etc.) |
| Discord-Bot-Somtum | Circlecore-somtum | On-demand replay checks from staff commands |

### Score Submission Flow

1. Client `POST`s encrypted score data to bancho.py via `c.somtum.fun`
2. bancho.py decrypts and validates the score
3. Score and stats written to MySQL
4. Replay queued to Circlecore-somtum for async analysis
5. Circlecore fires Discord webhook alert if suspicious
6. First-place announcements sent via Discord webhook

## Tech Stack

| Category | Technologies |
|---|---|
| **Languages** | Python, TypeScript, SQL |
| **Web Frameworks** | FastAPI, Next.js 14, React 18 |
| **Databases** | MySQL 9.4, Redis 8.2 |
| **Styling** | Tailwind CSS, shadcn/ui |
| **Discord** | discord.py 2.4+ |
| **Anti-cheat** | circleguard 5.4.3 |
| **Containers** | Podman / Podman Compose |
| **Reverse Proxy** | Caddy |
| **External APIs** | osu! API v1/v2, Discord, TrueMoney, PromptPay, Stripe |

## Infrastructure

| Provider | Purpose |
|---|---|
| **OVH** | VPS — hosts all services |
| **Cloudflare** | DNS, CDN, DDoS protection, SSL certificates |
| **Caddy** | Reverse proxy, automatic HTTPS, rate limiting |
| **Podman** | Container runtime & orchestration |

### Domain Routing

| Domain | Service | Notes |
|---|---|---|
| `somtum.fun` | gukarkka :3000 | Behind Cloudflare proxy |
| `session.somtum.fun` / `api.somtum.fun` | bancho.py :5001 | Behind Cloudflare proxy |
| `c.somtum.fun` | bancho.py :5001 | **Direct** — osu! client does not support Cloudflare proxy |
| `a.somtum.fun` / `assets.somtum.fun` | Static files | Avatars, replays |
