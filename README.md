# Property Revenue Dashboard: Debugging Challenge

> My solution to a take-home **debugging exercise**: a multi-tenant property-management revenue dashboard
> (FastAPI + PostgreSQL + Redis + React) went live with bugs that leaked data between clients and produced
> wrong revenue totals. I found and fixed the root causes.

![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?logo=redis&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)

The full brief is in [`ASSIGNMENT.md`](ASSIGNMENT.md). In short, clients reported:

- **Client A:** *"The revenue numbers for March don't match our internal records."*
- **Client B:** *"When we refresh the page, we sometimes see revenue numbers that belong to another company."*
- **Finance:** *"Some totals are off by a few cents."*

The rule was to **fix the existing code, not rebuild the system**.

## Bugs found and fixed

| # | Symptom | Root cause | Fix | Commit |
| --- | --- | --- | --- | --- |
| 1 | Backend couldn't reach the database in the Docker setup | The async engine was built from unused `supabase_db_*` settings instead of the `DATABASE_URL` provided by docker-compose. `QueuePool` is also incompatible with SQLAlchemy's async engine, and `get_session()` was declared `async`, which breaks `async with pool.get_session()` | Build the `postgresql+asyncpg://` URL from `DATABASE_URL`, use the async engine's default pool, make `get_session()` synchronous | `f47ccdc` |
| 2 | 🔒 **Client B sees another company's revenue** (data leak) | The Redis cache key was `revenue:{property_id}`. Different tenants own properties with the same IDs, so whichever tenant filled the cache first had its data served to the other for 5 minutes | Scope the key per tenant: `revenue:{tenant_id}:{property_id}` | `cc2bd5f` |
| 3 | Totals off by a few cents | Amounts are stored with 3 decimals (`NUMERIC(10,3)`) for sub-cent tracking, but the summed total was returned without being rounded to cents | `quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)` | `dbc10cf` |
| 4 | **Client A's monthly totals don't match** | Reservations were bucketed into months by their UTC `check_in_date`. A Paris booking checking in at 00:30 on 1 March is stored as 23:30 on 29 February UTC, so it was counted in February instead of March | Convert `check_in_date` to the **property's own time zone** (`properties.timezone`) before filtering by month | `b36d7ea` |

### Takeaways

- **Multi-tenant isolation must hold at every layer.** The SQL queries filtered by `tenant_id`, but the cache
  in front of them didn't, and that was enough to leak data.
- **Money needs explicit rounding rules.** Use `Decimal` and quantize to the currency's precision.
- **"Which month?" depends on the time zone.** Business dates should be computed in the property's local
  time zone, not the server's.

## Running the project

### Requirements

- Docker and Docker Compose

```bash
git clone https://github.com/Reathe/New_devs_App
cd New_devs_App
docker compose up --build
```

| Service | URL |
| --- | --- |
| Frontend (React dashboard) | http://localhost:3000 |
| Backend API docs (Swagger) | http://localhost:8000/docs |
| PostgreSQL | `localhost:5433` (seeded from `database/schema.sql` + `database/seed.sql`) |
| Redis | `localhost:6380` |

### Test accounts

| Client | Email | Password |
| --- | --- | --- |
| Sunset Properties (Client A) | `sunset@propertyflow.com` | `client_a_2024` |
| Ocean Rentals (Client B) | `ocean@propertyflow.com` | `client_b_2024` |

### Without Docker

```bash
make uv-install   # install backend dependencies with uv
make back         # FastAPI on :8000 (needs PostgreSQL & Redis, see backend/app/config.py)
make front        # Vite dev server
```

## Tech stack

- **Backend:** FastAPI, SQLAlchemy 2 (async) + asyncpg, Redis, multi-tenant context resolution
- **Frontend:** React 18, TypeScript, Vite, TanStack Query, Tailwind CSS
- **Infrastructure:** Docker Compose (PostgreSQL 15, Redis, nginx-served frontend)

## Credits

The base application and assignment were provided by the hiring company
([original repository](https://github.com/Base360-AI/New_devs_App)). The investigation and fixes are my own
(see the commits by **Reathe**).
