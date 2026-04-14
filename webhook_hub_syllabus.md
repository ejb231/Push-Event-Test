# GitHub Webhook Hub — Learning Syllabus & Architectural Guide

> **Document Type:** Living Syllabus · Architectural Reference · Self-Study Curriculum
> **Authored For:** A developer transitioning into professional backend engineering
> **Starting Point:** Solid fundamental Python. Prior project experience was guided/scaffolded — this is the first project where *you* own every architectural decision.
> **Stack:** Python · FastAPI · PostgreSQL · SQLAlchemy · Cloudflare Tunnel
> **Pedagogy:** Zero copy-paste. Concept-first. Socratic checkpoints.

> **Start here →** Read Section 1 to get the full mental model, then work through Sections 2–5 in order.

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Development Environment Setup](#2-development-environment-setup)
3. [Phase 1 — The Catch & Log](#3-phase-1--the-catch--log)
4. [Phase 2 — The Vault](#4-phase-2--the-vault)
5. [Phase 3 — The Secure Router](#5-phase-3--the-secure-router)
6. [Next Steps — Presenting This Project](#6-next-steps--presenting-this-project)

---

## 1. System Architecture Overview

### The Request-Response Lifecycle — End to End

Before you write a single line of code, you need a mental model of the entire data flow. Here is exactly what happens, in order, from a developer pressing Enter on `git push` to your server finishing its work:

**Step 1 — The Trigger (GitHub's Side)**

A developer pushes a commit. GitHub's internal event system detects this and looks up any **webhook configurations** registered on the repository. A webhook is nothing more than a URL the repository owner has told GitHub: *"When something happens, POST the details here."* GitHub serializes the event into a JSON payload, attaches metadata headers (event type, a delivery GUID, and optionally an HMAC signature), and fires an HTTP POST request to the registered URL.

**Step 2 — The Tunnel (Cloudflare Tunnel's Role)**

Your FastAPI server is running on `localhost`. GitHub cannot reach `localhost` — it is not a publicly routable address. Cloudflare Tunnel (via the `cloudflared` daemon) creates a **reverse tunnel**: it establishes an outbound-only connection from your machine to Cloudflare's edge network, which then routes incoming traffic back through that connection to your local port. Unlike throwaway tunneling tools, Cloudflare Tunnel gives you a stable URL tied to your Cloudflare account and is the same technology used in production infrastructure — so you're learning a real deployment tool, not a toy.

**Step 3 — The Arrival (Your Server)**

The POST request arrives at your FastAPI application. FastAPI is an **ASGI** (Asynchronous Server Gateway Interface) framework. This means your application runs on an async event loop (via Uvicorn), capable of handling concurrent I/O-bound operations without blocking. Your route handler receives the request, extracts the raw body and headers, and now has a decision to make: *Is this request legitimate?*

**Step 4 — The Verification (Phase 3)**

Before trusting the payload, you recompute the HMAC-SHA256 digest of the raw request body using a shared secret. You compare your computed digest against the signature GitHub sent in the `X-Hub-Signature-256` header. If they don't match, the request is either tampered with or forged. You reject it with a `403 Forbidden`. This is not optional in production — it is the **authentication layer** for incoming webhooks.

**Step 5 — The Persistence (Phase 2)**

The validated payload is written to PostgreSQL via an ORM layer. You store not just the raw JSON, but structured metadata: the event type, the delivery ID, the timestamp, the repository name. This transforms your application from a stateless echo server into a **queryable event ledger**.

**Step 6 — The Routing (Phase 3)**

After persisting the event, your server may need to notify downstream systems — a Slack channel, a CI pipeline trigger, an internal dashboard. These are dispatched as **background tasks** so the response to GitHub is not delayed. GitHub expects a `2xx` response within 10 seconds or it will mark the delivery as failed and begin retrying.

**Step 7 — The Response**

Your server returns `200 OK` (or `202 Accepted`) to GitHub. The entire cycle — tunnel → receive → verify → persist → acknowledge — should complete in under a second for most payloads.

```
  Developer         GitHub           Cloudflare Tunnel     FastAPI Server      PostgreSQL     Downstream
     |                 |                    |                    |                  |              |
     |--- git push --->|                    |                    |                  |              |
     |                 |-- serialize event --|                    |                  |              |
     |                 |-- compute HMAC -----|                    |                  |              |
     |                 |                    |                    |                  |              |
     |                 |--- POST /webhook -->|                    |                  |              |
     |                 |    (JSON+headers)   |--- forward req --->|                  |              |
     |                 |                    |                    |                  |              |
     |                 |                    |                    |-- verify HMAC     |              |
     |                 |                    |                    |                  |              |
     |                 |                    |          [if signature invalid]       |              |
     |                 |<---------- 403 Forbidden ---------------|                  |              |
     |                 |                    |                    |                  |              |
     |                 |                    |          [if signature valid]         |              |
     |                 |                    |                    |--- INSERT ------->|              |
     |                 |                    |                    |--- background ----|----------->  |
     |                 |<---------- 200 OK ------------------------|                |              |
     |                 |                    |                    |                  |              |
```

### Key Architectural Principle: Fail Fast, Respond Faster

Your webhook endpoint is not a general-purpose API. It is a **receiver under a contract**. GitHub will retry failed deliveries with exponential backoff, but it also has timeout thresholds. Your architectural mandate is: validate quickly, persist reliably, respond immediately, and defer all heavy work to background tasks. This is a fundamentally different design posture than a CRUD API where the client waits for the result.

---

## 2. Development Environment Setup

### Project Structure — Why It Matters

If you've worked on any Python project that grew past a few hundred lines, you've felt the pain of everything living in one file — imports tangling, functions doing too many things, no clear place to put new logic. This is the problem **layered architecture** solves. It is not academic overhead. It is a survival strategy: each directory maps to a single **responsibility boundary**, so when something breaks, you know exactly which layer to look at.

This will likely be the first project where you design the structure yourself from the ground up — nobody is scaffolding it for you. That's the point. You need to understand *why* each directory exists, not just copy a template.

Here is the structural pattern you should target. Do not create this all at once — grow into it as each phase demands new modules:

```
project-root/
├── app/
│   ├── __init__.py
│   ├── main.py              # Application entry point, FastAPI instance
│   ├── config.py             # Settings, environment variables
│   ├── models/               # ORM models (database table definitions)
│   │   ├── __init__.py
│   │   └── ...
│   ├── schemas/              # Pydantic schemas (request/response shapes)
│   │   ├── __init__.py
│   │   └── ...
│   ├── routes/               # Route handlers (thin — delegate to services)
│   │   ├── __init__.py
│   │   └── ...
│   ├── services/             # Business logic (validation, processing)
│   │   ├── __init__.py
│   │   └── ...
│   ├── db/                   # Database session, engine, migrations
│   │   ├── __init__.py
│   │   └── ...
│   └── security/             # HMAC verification, auth utilities
│       ├── __init__.py
│       └── ...
├── alembic/                  # Migration scripts (generated)
├── tests/
│   └── ...
├── alembic.ini
├── pyproject.toml            # Dependencies, project metadata
├── .env                      # Secrets (NEVER committed)
└── .gitignore
```

### Architectural Reasoning Behind Each Layer

| Directory | Responsibility | Why Separate? |
|-----------|---------------|---------------|
| `routes/` | Accept HTTP requests, return HTTP responses | Routes should be **thin**. They parse input and call services. If your route handler is longer than ~15 lines, logic is leaking into the wrong layer. |
| `services/` | Business logic: validation, transformation, orchestration | This is where your HMAC check lives, where you decide what to do with a payload. Testable in isolation without needing HTTP. |
| `models/` | Database table definitions via the ORM | Decouples your data schema from your API schema. Your database table and your API response shape are almost never identical. |
| `schemas/` | Pydantic models for request/response validation | FastAPI uses these to auto-generate OpenAPI docs and to validate incoming data at the boundary. |
| `db/` | Connection pooling, session lifecycle | Database connections are expensive resources. This layer manages the pool, provides sessions to handlers via dependency injection, and ensures cleanup. |
| `security/` | Cryptographic verification, secrets management | Security code is audited separately. Isolating it makes review and testing straightforward. |

### Environment Management

Every Python project must live in its own **virtual environment** — an isolated copy of the Python interpreter with its own installed packages. Without one, installing a dependency for this project can break a completely unrelated project on your system. If you haven't been doing this consistently, start now. It is non-negotiable in professional work.

**Why virtual environments exist:** Python has a single global `site-packages` directory. If Project A needs `sqlalchemy==2.0` and Project B needs `sqlalchemy==1.4`, they can't coexist globally. A virtual environment gives each project its own `site-packages`. You create one with `python -m venv .venv`, activate it, and now `pip install` only affects that isolated environment.

- Use `pyproject.toml` as your single source of truth for dependencies (not `requirements.txt` — that is a legacy pattern). Tools like `uv`, `pdm`, or `poetry` all support this. `uv` is currently the fastest and simplest — start there.
- Store secrets (database URLs, webhook secrets, API keys) in a `.env` file. Load them with `pydantic-settings` — which gives you typed, validated configuration with zero boilerplate. **Never hardcode secrets. Never commit `.env`.**

> **→ Next:** Once your venv is active and `fastapi` + `uvicorn` are installed, move to Phase 1.

---

## 3. Phase 1 — The Catch & Log

> **Your goal for this phase:** Get a working endpoint that receives a real GitHub webhook delivery and logs it. By the end, you should have seen a live `push` event arrive at your server.

### Core Concepts to Master

#### 3.1 REST and the HTTP Contract

REST (Representational State Transfer) is not a protocol — it is an **architectural style** built on top of HTTP. You need to internalize these HTTP fundamentals:

- **Methods** define intent: `GET` (read), `POST` (create), `PUT` (full replace), `PATCH` (partial update), `DELETE` (remove).
- **Status codes** are the server's vocabulary: `200` (success), `201` (created), `400` (bad request — your fault), `403` (forbidden), `404` (not found), `500` (server error — my fault).
- **Headers** carry metadata: `Content-Type` tells the recipient how to parse the body. `X-Hub-Signature-256` carries GitHub's HMAC signature. Headers are the "envelope" — the body is the "letter."
- **The body** is the payload. For webhooks, this is always JSON.

A webhook is a **server-to-server POST request**. There is no browser, no human, no interactive session. GitHub is the client; you are the server. This inverts the mental model most people have of APIs: you are not calling out to fetch data, you are sitting still and waiting for data to arrive.

#### 3.2 Decorators — The Routing Mechanism

FastAPI uses Python decorators to bind a function to an HTTP route. Before you use them, make sure you understand what a decorator actually *is* at the language level — because it's not magic, it's a higher-order function.

**The Mechanical Truth:** A decorator is a function that takes a function as input and returns a new function (usually wrapping the original with additional behavior). The `@` syntax is just syntactic sugar.

```python
# What a decorator IS, mechanically:

def log_calls(func):
    """A decorator that logs every call to the wrapped function."""
    def wrapper(*args, **kwargs):
        print(f"Calling: {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished: {func.__name__}")
        return result
    return wrapper


# These two are IDENTICAL:

@log_calls
def bake_bread(flavor):
    return f"Baked {flavor} bread"

# ...is the same as writing:
# bake_bread = log_calls(bake_bread)
```

When FastAPI uses `@app.post("/orders")`, it is registering your function in an internal routing table, keyed by the HTTP method and path. Uvicorn's event loop will call your function whenever a matching request arrives.

#### 3.3 Async Functions — Why `async def` Matters

FastAPI runs on an **ASGI** server (Uvicorn). ASGI is the async successor to WSGI. When you declare a route handler with `async def`, you are telling the event loop: *"This function will yield control when it hits I/O (network calls, database queries, file reads), so you can serve other requests in the meantime."*

For a webhook receiver, this matters because:

1. Multiple webhook deliveries can arrive simultaneously (e.g., several developers pushing to the same repo in quick succession).
2. Your handler will perform I/O: reading the request body, writing to the database, forwarding to downstream services.
3. If you use synchronous `def`, FastAPI offloads it to a **thread pool** — it still works, but you lose the efficiency of cooperative multitasking.

**The analogy:** A synchronous waiter takes one table's order, walks to the kitchen, stands there until the food is ready, walks back, and only then takes the next table's order. An async waiter takes the order, sends it to the kitchen, immediately moves to the next table, and picks up the food when the kitchen signals it's ready. Same waiter. Same kitchen. Dramatically different throughput.

```python
# Synchronous — blocks the event loop during I/O
def get_bakery_inventory():
    items = database.fetch_all_items()  # Blocks here
    return items


# Asynchronous — yields control during I/O
async def get_bakery_inventory():
    items = await database.fetch_all_items()  # Yields here
    return items
```

> **⚠️ IMPORTANT:** The `await` keyword is not optional decoration. It is the mechanism that suspends the coroutine and returns control to the event loop. If you call an async function without `await`, you get a coroutine object, not a result. Your IDE won't always catch this.

#### 3.4 FastAPI Route Handlers — Request Anatomy

FastAPI gives you multiple ways to access incoming request data. For webhooks, you need **two** things: the raw body (for HMAC verification later) and the parsed JSON (for processing). Here's the generic pattern:

```python
from fastapi import FastAPI, Request

app = FastAPI()


@app.post("/orders")
async def receive_order(request: Request):
    # Access raw bytes — needed for cryptographic verification
    raw_body = await request.body()

    # Parse JSON — needed for business logic
    order_data = await request.json()

    # Access headers — metadata from the sender
    content_type = request.headers.get("content-type")
    custom_header = request.headers.get("x-bakery-signature")

    return {"status": "received", "order_id": order_data.get("id")}
```

**Why `Request` and not a Pydantic model here?** When you define a parameter with a Pydantic type, FastAPI consumes the request body to validate it. But to verify an HMAC, you need the **exact raw bytes** that were signed — before any parsing or transformation. You can't get those back once FastAPI has consumed them into a model. This is a webhook-specific constraint that differs from typical CRUD API patterns.

#### 3.5 Cloudflare Tunnel — Exposing Localhost

Cloudflare Tunnel exposes your local server to the public internet through Cloudflare's edge network. Architecturally, it works like this:

1. You start your FastAPI server on `localhost:8000`.
2. You run `cloudflared tunnel --url http://localhost:8000` (for quick testing) or configure a named tunnel tied to your Cloudflare account.
3. `cloudflared` establishes **outbound-only** connections to Cloudflare's nearest data centers — no inbound ports are opened on your machine.
4. Cloudflare allocates a public URL (a `*.trycloudflare.com` subdomain for quick tunnels, or your own domain for named tunnels) and routes incoming traffic through the tunnel to your local port.

**Why Cloudflare Tunnel instead of Ngrok?** Two reasons that matter for your learning:

- **Stable URLs.** With a named tunnel tied to your Cloudflare account, your webhook URL doesn't change between restarts. You configure it once in GitHub and forget about it.
- **Production relevance.** Cloudflare Tunnels are used in real production infrastructure to expose services without opening firewall ports. You're learning a tool that transfers directly to professional work — not a dev-only crutch you'll discard later.

> **📝 NOTE — Quick tunnels vs. named tunnels:** Running `cloudflared tunnel --url http://localhost:8000` gives you a temporary `*.trycloudflare.com` URL — fine for a first test. For persistent use, create a **named tunnel** through the Cloudflare Zero Trust dashboard. This gives you a stable hostname and lets you configure access policies. The `cloudflared` CLI walks you through the setup.

#### 3.6 Practical Workflow for Phase 1

1. Create a minimal FastAPI application with a single POST endpoint.
2. Have the handler log the received headers and body to the console using Python's `logging` module (not `print` — `print` is not production-grade logging).
3. Run the app with Uvicorn.
4. Start a Cloudflare Tunnel pointing at Uvicorn's port.
5. Configure a GitHub repository's webhook settings to point at your tunnel URL.
6. Push a commit and watch the log output. Inspect the JSON structure. Read every field. Understand what GitHub sends you.

> **💡 TIP:** Use `logging` properly from day one. Configure structured logging with timestamps and log levels. When you're debugging a failed webhook delivery at 2 AM, `print("got here")` is useless. `logger.info("Received delivery %s for event %s", delivery_id, event_type)` is not.

```python
import logging

# Configure once, at application startup
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
)

logger = logging.getLogger("bakery")

logger.info("Oven preheated to %d°F", 425)
logger.warning("Low on flour — %d kg remaining", 2)
logger.error("Oven failed to ignite: %s", "gas valve closed")
```

> **→ Next:** Once you've received a live webhook and studied the payload structure, answer the checkpoint questions below, then move to Phase 2.

---

### 🔒 Mentor Checkpoint — Phase 1

Do not proceed to Phase 2 until you can answer these clearly and confidently:

1. **Data Flow:** When GitHub sends a webhook POST to your Cloudflare Tunnel URL, describe the exact sequence of network hops the request takes before it reaches your `async def` handler. What happens at each hop? What protocol transitions occur?

2. **Async Reasoning:** You defined your handler with `async def` and used `await request.body()`. If you removed the `await` and just wrote `body = request.body()`, what would `body` contain? Why? What class of bug does this introduce, and why is it particularly insidious (i.e., why might your code *appear* to work)?

3. **Architecture Decision:** Your handler currently logs the payload to the console. The console is ephemeral — if the server restarts, all received events are lost. What architectural component do you need to add to make event data durable, and what is the minimum information you would store per event to make the data useful for later analysis?

---

## 4. Phase 2 — The Vault

> **Your goal for this phase:** Persist every webhook event to PostgreSQL so nothing is lost when the server restarts. You'll need a running Postgres instance, an ORM model, a migration, and a refactored handler.

### Core Concepts to Master

#### 4.1 Why a Database? Why Not Just Files?

The instinct when you first need to persist data is: *"I'll just write it to a file."* A JSON file, a CSV, a log file — something simple. For a hobby script that runs once a day, that's fine. For a server that receives concurrent webhook deliveries, it will fail. Here is exactly why:

The answer is **concurrent access, queryability, and integrity**:

- **Concurrent writes:** Multiple webhook deliveries can arrive simultaneously. Appending to a file from multiple async handlers creates a race condition. Imagine two handlers both open the file, both read the current position, and both write at the same offset — one overwrites the other. You get partial writes, interleaved JSON, corruption. Databases handle this with **transactional isolation**: each write is atomic and serialized, even under heavy concurrency.
- **Queryability:** "Show me all `push` events to the `main` branch in the last 24 hours." With a JSON file, you load the entire file into memory, deserialize every object, and loop through them in Python. With a database, you write a query with a `WHERE` clause and the database engine uses **indexes** (pre-built lookup structures) to return results in milliseconds — without scanning every row.
- **ACID guarantees:** These are the four properties that distinguish a database from a file:
  - **Atomicity** — the write either fully succeeds or fully fails. No partial records.
  - **Consistency** — the data always satisfies your defined constraints (e.g., "this column cannot be null").
  - **Isolation** — concurrent transactions don't interfere with each other.
  - **Durability** — committed data survives a power failure. The database writes to a **write-ahead log** (WAL) before confirming the commit, so even if the process crashes mid-write, the data is recoverable.

#### 4.2 Relational Databases — The Mental Model

PostgreSQL is a **relational database**. The data model is tables (relations) with typed columns and rows. Think of it as a strictly enforced spreadsheet:

- Each **table** is a noun (e.g., `orders`, `customers`, `deliveries`).
- Each **column** has a name, a data type, and optional constraints (`NOT NULL`, `UNIQUE`, `FOREIGN KEY`).
- Each **row** is a record — one instance of the noun.
- **Primary keys** uniquely identify rows. Use UUIDs or auto-incrementing integers.
- **Indexes** are auxiliary data structures that make queries on specific columns fast, at the cost of slightly slower writes.

> **📝 NOTE:** PostgreSQL's `JSONB` column type is particularly relevant for your project. It stores JSON data in a binary format that supports indexing and querying into nested fields. This means you can store the full webhook payload as-is *and* still query into it efficiently — a pragmatic hybrid of relational and document storage.

#### 4.3 The ORM — Why an Abstraction Layer?

An **Object-Relational Mapper** (ORM) lets you interact with database tables as Python classes and rows as Python objects. Instead of writing raw SQL strings, you call methods.

**Why not just write SQL?** You *can*. And for complex analytical queries, you sometimes *should*. But for standard CRUD operations, raw SQL has problems:

1. **SQL injection.** Constructing queries with string concatenation is the single most common web security vulnerability. ORMs use parameterized queries by default.
2. **Boilerplate.** Mapping result rows to Python dicts or objects manually is tedious and error-prone.
3. **Portability.** An ORM abstracts dialect differences between databases (though in practice, you pick Postgres and stay there).
4. **Migration management.** ORMs integrate with migration tools that version-control your schema changes.

**SQLAlchemy** is the dominant ORM in the Python ecosystem — and the one you will use for this project. There are alternatives (SQLModel, Tortoise ORM, Peewee), but SQLAlchemy is what you'll encounter in virtually every professional Python codebase. It is more verbose than some alternatives, and that's a feature at your stage: the verbosity forces you to understand what each layer does (engine, session, model, query) instead of hiding it behind convenience methods. Once you've internalized SQLAlchemy's patterns, picking up a higher-level wrapper like SQLModel later takes an afternoon.

Here's what ORM mapping looks like conceptually:

```python
# Generic example: A bakery's order tracking system

from sqlalchemy import Column, Integer, String, DateTime, JSON
from sqlalchemy.orm import DeclarativeBase
from datetime import datetime, timezone


class Base(DeclarativeBase):
    pass


class Order(Base):
    __tablename__ = "orders"

    id = Column(Integer, primary_key=True, autoincrement=True)
    customer_name = Column(String(100), nullable=False)
    items = Column(JSON, nullable=False)        # JSONB in PostgreSQL
    status = Column(String(20), default="pending")
    created_at = Column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
    )
```

This class *is* your table definition. When the migration tool reads it, it generates the `CREATE TABLE` SQL. When you create an instance of `Order(...)`, you're building an in-memory representation of a row. When you add it to a session and commit, the ORM generates and executes the `INSERT` statement.

#### 4.4 Sessions and the Unit of Work Pattern

The ORM session is the central concept you need to internalize. A session is **not** a database connection — it is a **staging area** (a "unit of work") that tracks changes to objects and flushes them to the database in a single transaction when you commit.

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker


# Engine = connection pool manager (create ONCE at startup)
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/bakery")

# Session factory = creates new sessions on demand
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


# Using a session to persist a record
async def place_order(customer: str, items: list):
    async with AsyncSessionLocal() as session:
        new_order = Order(customer_name=customer, items=items)
        session.add(new_order)          # Stage the object
        await session.commit()          # Execute INSERT, commit transaction
        await session.refresh(new_order)  # Reload to get DB-generated fields (id, created_at)
        return new_order
```

> **⚠️ IMPORTANT:** `expire_on_commit=False` matters in async contexts. By default, SQLAlchemy expires (invalidates) all loaded attributes on an object after commit, forcing a lazy reload on next access. In async mode, lazy loading triggers a synchronous I/O call, which **raises an error**. Setting this flag tells the session to keep the in-memory attributes valid after commit.

#### 4.5 Dependency Injection — FastAPI's Session Pattern

FastAPI uses **dependency injection** to provide resources (like database sessions) to route handlers. This is not a framework quirk — it is a deliberate architectural pattern that decouples your route logic from resource lifecycle management.

```python
from fastapi import Depends
from typing import AsyncGenerator


async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session       # Provide the session to the handler
        finally:
            await session.close()  # Cleanup after the handler finishes


@app.post("/orders")
async def create_order(request: Request, db: AsyncSession = Depends(get_db_session)):
    # 'db' is a live session, provided and cleaned up by the framework
    data = await request.json()
    order = Order(customer_name=data["customer"], items=data["items"])
    db.add(order)
    await db.commit()
    return {"status": "created"}
```

**Why is this better than creating the session inside the handler?** Three reasons:

1. **Testability.** In tests, you override the dependency to inject a test database session. Your handler code doesn't change.
2. **Lifecycle safety.** The `finally` block guarantees cleanup even if the handler raises an exception.
3. **Single Responsibility.** The handler focuses on business logic. Session plumbing is someone else's job.

#### 4.6 Migrations — Version Control for Your Schema

Your database schema will evolve. You'll add columns, rename tables, add indexes. You cannot just drop and recreate the database each time — in production, that destroys data.

**Alembic** is the migration tool for SQLAlchemy. It works like `git` for your database schema:

1. You change your ORM model (e.g., add a column to the `Order` class).
2. You run `alembic revision --autogenerate -m "add priority column to orders"`.
3. Alembic diffs your model definitions against the current database schema and generates a migration script (a Python file with `upgrade()` and `downgrade()` functions).
4. You review the script (auto-generated migrations are suggestions, not gospel).
5. You run `alembic upgrade head` to apply it.

> **🚨 CAUTION:** Never blindly trust auto-generated migrations. Alembic cannot detect all changes (e.g., column renames look like a drop + add, which destroys data). Always read the generated script before applying it. This is especially critical when working with JSONB columns — Alembic has limited introspection into JSON structure.

#### 4.7 Designing Your Data Model — What to Store

This is a design decision, not a syntax question. Think about it before you code. Consider:

- **What queries will you run?** "All events in the last hour" needs a timestamp column with an index. "All push events" needs an event type column. Design for your access patterns.
- **Raw vs. structured?** Store the full JSON payload for auditability, but also extract key fields into dedicated columns for fast querying. This is a **denormalization** trade-off — you're trading storage space for query speed.
- **Idempotency.** GitHub sends a unique delivery ID with every webhook. Storing it and adding a `UNIQUE` constraint prevents duplicate processing if GitHub retries a delivery your server already handled.

> **→ Next:** Once you can query your database and see persisted webhook events, answer the checkpoint questions below, then move to Phase 3.

---

### 🔒 Mentor Checkpoint — Phase 2

Do not proceed to Phase 3 until you can answer these clearly and confidently:

1. **Session Lifecycle:** You open an async session, add an ORM object, and call `await session.commit()`. Then your handler raises an unhandled exception *after* the commit but *before* returning the response. Is the data persisted in the database? Why or why not? Now consider the reverse: the exception occurs *before* `commit()`. What happens to the staged object?

2. **Schema Design Trade-off:** You're deciding whether to store the entire webhook payload as a single `JSONB` column or to extract every field into separate typed columns. Argue *both* sides. What does the hybrid approach look like, and why is it the standard practice for event-sourcing systems?

3. **Migration Danger:** You have a table in production with 50,000 rows. You rename a column in your ORM model and run `alembic revision --autogenerate`. What does the auto-generated migration *actually* do (hint: it is **not** a rename)? What is the consequence, and how would you write the migration correctly?

---

## 5. Phase 3 — The Secure Router

> **Your goal for this phase:** No unsigned or forged request should be processed. Add HMAC verification, an idempotency guard, and at least one background task. After this, your pipeline handles the full lifecycle: verify → persist → dispatch.

### Core Concepts to Master

#### 5.1 Why HMAC? The Trust Problem

Your endpoint is a public URL. Anyone who knows it can send a POST request. Without verification, an attacker could forge payloads — fake push events, fake deployment statuses, fake security alerts. Your server would process them as legitimate.

**HMAC (Hash-based Message Authentication Code)** solves this with a shared secret:

1. You and GitHub both know a secret string (you set it when configuring the webhook).
2. GitHub computes `HMAC-SHA256(secret, raw_request_body)` and sends the hex digest in the `X-Hub-Signature-256` header as `sha256=<digest>`.
3. Your server receives the request, computes the same HMAC over the raw body using the same secret, and compares.
4. If the digests match, the message is **authentic** (it came from someone who knows the secret) and has **integrity** (it hasn't been modified in transit).

This is not encryption — the payload is transmitted in plaintext (though TLS encrypts the wire). HMAC provides **authentication** and **tamper detection**.

#### 5.2 The Cryptographic Implementation

Python's `hmac` and `hashlib` modules are all you need. Here is the generic pattern:

```python
import hmac
import hashlib


def verify_signature(payload_body: bytes, secret: str, received_signature: str) -> bool:
    """
    Verify that a payload was signed with the expected secret.

    Generic example: verifying a bakery's order authentication token.
    """
    # Compute the expected signature
    computed = hmac.new(
        key=secret.encode("utf-8"),
        msg=payload_body,
        digestmod=hashlib.sha256,
    ).hexdigest()

    expected_signature = f"sha256={computed}"

    # CRITICAL: Use constant-time comparison
    return hmac.compare_digest(expected_signature, received_signature)
```

> **🚨 CAUTION:** Never use `==` to compare cryptographic digests. String equality comparison in Python short-circuits: it returns `False` as soon as it finds the first differing character. This leaks information about how many leading characters are correct, enabling **timing attacks** where an adversary iteratively guesses the signature one character at a time. `hmac.compare_digest()` always takes the same amount of time regardless of where the strings differ.

#### 5.3 Where Verification Fits Architecturally

HMAC verification must happen **before** any processing — before JSON parsing, before database writes, before background task dispatch. It is a **guard clause** at the gate.

In FastAPI, you have two choices for where to place this guard:

**Option A: Inside the handler.** Simple, explicit, easy to understand. Fine for a single endpoint.

**Option B: As a dependency.** Create a `verify_webhook` dependency that reads the raw body and header, verifies the signature, and raises `HTTPException(403)` on failure. Inject it into the route. This is cleaner if you have multiple webhook endpoints.

```python
from fastapi import HTTPException, Header


async def verified_payload(
    request: Request,
    x_hub_signature_256: str = Header(...),
) -> bytes:
    """
    FastAPI dependency that verifies incoming request authenticity.
    Returns the raw body only if verification passes.

    Generic bakery example: verifying supplier delivery manifests.
    """
    body = await request.body()

    if not verify_signature(body, SECRET, x_hub_signature_256):
        raise HTTPException(status_code=403, detail="Invalid signature")

    return body
```

**Why a dependency and not middleware?** Middleware in ASGI intercepts *every* request — including health checks, static file serves, and documentation endpoints. Signature verification is specific to webhook routes. A dependency gives you per-route granularity.

#### 5.4 Background Tasks — The Async Dispatch Pattern

After validating and persisting a webhook event, you may need to take action: post to Slack, trigger a build, update a dashboard. These actions are **side effects** — they should not delay the response to GitHub.

FastAPI's `BackgroundTasks` parameter lets you schedule functions to run *after* the response is sent:

```python
from fastapi import BackgroundTasks
import httpx


async def notify_bakery_staff(order_id: int, items: list):
    """Send a notification to the bakery dashboard about a new order."""
    async with httpx.AsyncClient() as client:
        await client.post(
            "https://bakery-dashboard.internal/notifications",
            json={"order_id": order_id, "items": items},
        )


@app.post("/orders")
async def receive_order(
    request: Request,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db_session),
):
    data = await request.json()

    # Persist the order (fast)
    order = Order(customer_name=data["customer"], items=data["items"])
    db.add(order)
    await db.commit()

    # Schedule notification (runs AFTER the response is sent)
    background_tasks.add_task(notify_bakery_staff, order.id, data["items"])

    return {"status": "accepted"}
```

**How this works under the hood:** `BackgroundTasks` collects callables into a queue. After FastAPI sends the HTTP response, ASGI's lifespan protocol runs the queued tasks sequentially in the same event loop. This is lightweight but has limitations:

- Tasks run **in-process** — if the server crashes, queued tasks are lost.
- Tasks are **sequential** — they don't run in parallel by default.
- For production-grade reliability, you'd eventually migrate to a proper task queue (Celery, Dramatiq, Arq) with persistent storage. But for this project, `BackgroundTasks` is the right tool.

#### 5.5 Concurrency — Understanding the Event Loop

You're already using `async/await`, so you're in async territory. But there are subtleties:

**CPU-bound vs. I/O-bound work:**
- I/O-bound (network calls, database queries, file reads): use `async/await`. The event loop can serve other requests while waiting.
- CPU-bound (cryptographic computation, JSON parsing of very large payloads, compression): these **block** the event loop. For small payloads, the HMAC computation is negligible. For very large payloads, consider offloading to a thread pool via `asyncio.to_thread()`.

```python
import asyncio


def cpu_heavy_bread_scoring(dough_data: bytes) -> dict:
    """Simulate a CPU-intensive operation."""
    # ... complex computation ...
    return {"score": 42}


async def handle_dough_request(raw_dough: bytes):
    # Offload to thread pool to avoid blocking the event loop
    result = await asyncio.to_thread(cpu_heavy_bread_scoring, raw_dough)
    return result
```

**The practical implication for your project:** HMAC-SHA256 computation over a typical webhook payload (a few KB) takes microseconds — far too fast to warrant thread offloading. But if you add downstream HTTP calls in your handler (instead of background tasks), those *must* be `await`-ed using an async HTTP client like `httpx`, not the synchronous `requests` library, which would block the entire event loop.

#### 5.6 Putting It All Together — The Request Pipeline

At the end of Phase 3, your request handling pipeline should follow this sequence:

```
Request Arrives
    │
    ▼
[1] Extract raw body + signature header
    │
    ▼
[2] HMAC verification (reject → 403)
    │
    ▼
[3] Parse JSON from raw body
    │
    ▼
[4] Extract metadata (event type, delivery ID, repo name, timestamp)
    │
    ▼
[5] Check for duplicate delivery ID (idempotency guard)
    │
    ▼
[6] Persist to PostgreSQL (within a transaction)
    │
    ▼
[7] Schedule background tasks (forwarding, notifications)
    │
    ▼
[8] Return 200 OK
```

Every step before [8] can fail, and each failure has a specific, appropriate HTTP status code. Design your error handling to be as precise as your success path.

#### 5.7 Configuration and Secrets Management

Your application now has multiple configuration values: the database URL, the webhook secret, the tunnel hostname, downstream service endpoints. Hardcoding any of these is a deployment hazard.

Use `pydantic-settings` to create a typed settings class that reads from environment variables or `.env` files:

```python
from pydantic_settings import BaseSettings


class BakerySettings(BaseSettings):
    database_url: str
    webhook_secret: str
    notification_endpoint: str
    log_level: str = "INFO"

    model_config = {"env_file": ".env"}


# Instantiate once at startup
settings = BakerySettings()

# Now use: settings.database_url, settings.webhook_secret, etc.
```

This gives you runtime validation (the app crashes immediately with a clear error if a required variable is missing), type coercion, and a single source of truth.

> **→ Next:** Once a forged request gets rejected and a real delivery flows through verify → persist → background task, answer the checkpoint questions below.

---

### 🔒 Mentor Checkpoint — Phase 3

Do not proceed until you can answer these clearly and confidently:

1. **Security Reasoning:** An attacker discovers your webhook endpoint URL. They craft a valid-looking JSON payload mimicking a GitHub `push` event and POST it to your server. Walk through exactly what happens in your verification pipeline. At which step is the forgery detected? What information does the attacker need (and not have) to bypass your verification? Could they obtain it through a timing attack if you used `==` instead of `hmac.compare_digest()`?

2. **Concurrency Trade-off:** You have a background task that makes an HTTP POST to a slow external service (average 3-second response time). Two webhook deliveries arrive 100ms apart. Describe what happens with FastAPI's built-in `BackgroundTasks`. Do the two notifications run concurrently? What happens if the first one fails? How would the behavior differ if you used a dedicated task queue like Celery?

3. **Architectural Integrity:** Your handler currently reads the raw body for HMAC verification, then calls `await request.json()` to parse the payload. Under the hood, what does `request.json()` do — does it re-read the body from the network, or use a cached copy? Why does this matter for the correctness of your HMAC check? What would break if you swapped the order (parsed JSON first, then tried to read raw body)?

---

## 6. Next Steps — Presenting This Project

### The Project Is the Portfolio

Once all three phases are complete, you have a non-trivial backend system that demonstrates:

- **API design** — a purpose-built endpoint with clear contracts
- **Database engineering** — schema design, ORM usage, migration management
- **Security engineering** — cryptographic verification, secrets management
- **Async systems design** — non-blocking I/O, background task dispatch
- **Operational readiness** — structured logging, configuration management, error handling

### How to Present It on a Resume

**Project title:** GitHub Webhook Telemetry Hub (or similar — name it something that sounds like a product, not a tutorial)

**Description format (1-2 lines):**
> Engineered an event-driven webhook ingestion service using FastAPI and PostgreSQL. Implemented HMAC-SHA256 payload verification, async background task routing, and schema-versioned persistence for real-time GitHub event telemetry.

**Key technical bullets (pick 3-4):**
- Designed and implemented HMAC-SHA256 signature verification for webhook payload authentication
- Built async PostgreSQL persistence layer using SQLAlchemy with Alembic schema migrations
- Architected background task dispatch pipeline for downstream event forwarding with idempotency guards
- Structured application using layered architecture with dependency injection for testability

### Beyond the Three Phases — Growth Vectors

Once the core is solid, consider these extensions (each one is a portfolio-worthy enhancement):

| Extension | What It Proves |
|-----------|---------------|
| **Add a REST API to query stored events** (GET endpoints with filtering, pagination) | Full-stack API design, not just webhook receiving |
| **Dockerize the entire stack** (FastAPI + Postgres + Cloudflare Tunnel in Docker Compose) | Containerization, infrastructure-as-code |
| **Add rate limiting** (per-source, per-event-type) | Defensive programming, abuse prevention |
| **Write integration tests** (use `httpx.AsyncClient` + test database) | Test engineering, CI/CD readiness |
| **Deploy to a cloud provider** (Railway, Fly.io, or a VPS with Caddy) | Production deployment, DNS, TLS |
| **Build a real-time dashboard** (WebSocket-fed frontend showing live events) | Full-stack capability, real-time systems |

---

> **This is a living document.** As you progress through each phase, come back and annotate it with your own notes, gotchas you encountered, and solutions you discovered. The best engineering documentation is the kind that evolves with the engineer.
