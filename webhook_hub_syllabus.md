# GitHub Webhook Hub — Learning Syllabus & Architectural Guide

> **What this is:** A hands-on guide for building a real backend system. Not a tutorial you follow blindly — a curriculum that teaches you to think like a professional backend engineer.
>
> **Who it's for:** You. A developer who knows Python well enough to build things, but hasn't yet had to make all the architectural decisions yourself. This is where that changes.
>
> **Stack:** Python · FastAPI · PostgreSQL · SQLAlchemy · ngrok

---

### How to Work Through This Document

I want to be upfront about how this is meant to be used, because it matters.

**Each section has two halves: concepts and practice.** Here's the workflow:

1. **Read the concept material for the entire section first.** Don't touch your editor yet. Just read. Your goal is to build a mental picture of what you're about to build and *why* each piece exists. If you start coding halfway through reading, you'll make decisions based on incomplete understanding and have to redo things.

2. **Then find the "Practical Workflow" at the end of the section.** That's your step-by-step build checklist. Work through it one step at a time. Each step tells you exactly what to create and where to put it.

3. **When you hit a checkpoint, stop.** The checkpoint questions aren't optional extras. They're how you find out if you actually understood the section or just *think* you did. If you can't answer one, go re-read the concept — don't keep building on a shaky foundation.

**"Where does my code go?"** — I hear you, and I'll be specific throughout. The project structure section (Section 2) lays out the directory map, and each Practical Workflow tells you exactly which file to create inside that map. But here's the key thing: **you don't create all the directories at once.** You build them as you need them. Phase 1 is just `app/main.py`. Phase 2 adds `app/models/` and `app/db/`. Phase 3 adds `app/security/` and `app/services/`. Don't create empty folders you're not ready to use yet.

> **Type everything yourself.** This document shows example code using a bakery theme — not webhook code. That's deliberate. Your job is to read the concept, understand the pattern, then write *your own* version for webhooks. If you copy-paste, you learn nothing. If you type it out and adapt it, you learn the pattern.

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Development Environment Setup](#2-development-environment-setup)
3. [Phase 1 — The Catch & Log](#3-phase-1--the-catch--log)
4. [Phase 2 — The Vault](#4-phase-2--the-vault)
5. [Phase 3 — The Secure Router](#5-phase-3--the-secure-router)
6. [Next Steps — Presenting This Project](#6-next-steps--presenting-this-project)
7. [Appendix A — How to Get Unstuck](#appendix-a--how-to-get-unstuck)
8. [Appendix B — Glossary of Terms](#appendix-b--glossary-of-terms)

---

## 1. System Architecture Overview

### Why Start With Architecture?

Most people learning backend development want to jump straight into writing code. I get it — code feels productive, reading doesn't. But here's what happens when you skip this step: you build something that works for the first feature, then you have to tear it apart when the second feature doesn't fit. I've watched experienced developers lose entire days to this.

So before you write a single line, I want you to be able to close your eyes and trace the entire journey of a webhook event — from a developer's `git push` to your server's `200 OK`. Every line of code you write in Phases 1–3 implements a piece of this picture. If you know the picture, you know where each piece goes.

**Read this entire section before moving on. No coding here — just understanding.**

### The Request-Response Lifecycle — End to End

Here is exactly what happens, step by step, when a developer pushes code and your server handles the resulting event:

**Step 1 — The Trigger (GitHub's Side)**

A developer pushes a commit. GitHub's internal event system detects this and looks up any **webhook configurations** registered on the repository. A webhook is just a URL the repository owner has told GitHub: *"When something happens, send the details here."* GitHub puts together a JSON payload with all the event details, attaches some metadata headers (event type, a unique delivery ID, and optionally a signature), and fires an HTTP POST request to that URL.

**What this means for you:** You have zero control over this step. GitHub decides the payload format, the headers, and the timing. Your entire application is a *response* to someone else's system. This is different from building a normal API where you design the rules — here, GitHub sets the rules and you follow them.

**Step 2 — The Tunnel (ngrok's Role)**

Your FastAPI server runs on `localhost`. GitHub can't reach `localhost` — it's not a public address. ngrok solves this by creating a **reverse tunnel**: it opens an outbound connection from your machine to ngrok's servers, and ngrok then routes incoming traffic back through that connection to your local port. You run `ngrok http 8000`, it gives you a public URL, and anything that hits that URL ends up at your local server.

**Step 3 — The Arrival (Your Server)**

The POST request arrives at your FastAPI application. FastAPI runs on an async event loop (via Uvicorn), which means it can handle multiple requests at the same time without them blocking each other. Your route handler receives the request, grabs the raw body and headers, and now has to answer a question: *Is this request actually from GitHub, or is someone messing with us?*

**Step 4 — The Verification (You'll build this in Phase 3)**

Before trusting the payload, you recompute the HMAC-SHA256 digest of the raw request body using a shared secret. You compare your result against the signature GitHub attached in the `X-Hub-Signature-256` header. If they don't match, the request is either tampered with or forged — you reject it with a `403 Forbidden`. In production, this is not optional. It's how you know the request is legitimate.

**Step 5 — The Persistence (You'll build this in Phase 2)**

The verified payload is written to PostgreSQL through an ORM. You don't just dump the raw JSON — you also pull out structured metadata: the event type, the delivery ID, the timestamp, the repository name. This turns your application from a temporary echo server into a **permanent event ledger** you can query later.

**Step 6 — The Routing (You'll build this in Phase 3)**

After storing the event, your server might need to tell other systems — a Slack channel, a CI pipeline, a dashboard. These notifications are dispatched as **background tasks** so you can send the response to GitHub without waiting for Slack to respond. GitHub expects a `2xx` response within 10 seconds or it'll start retrying.

**Step 7 — The Response**

Your server returns `200 OK` to GitHub. The whole cycle — tunnel → receive → verify → persist → acknowledge — should take well under a second.

```
  Developer         GitHub           ngrok Tunnel          FastAPI Server      PostgreSQL     Downstream
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

### The Key Principle: Fail Fast, Respond Faster

Your webhook endpoint isn't a general-purpose API. It's a **receiver under a contract.** GitHub will retry failed deliveries, but it also has timeout limits. Your design priority is: validate quickly, store reliably, respond immediately, and push any heavy work to background tasks.

**Think of it like a postal receiving dock:** Check the delivery driver's ID (HMAC verification), log the package in the ledger (database write), put it on the conveyor belt for processing (background tasks), and sign the driver's clipboard (HTTP 200). You don't open the package and assemble its contents while the driver waits.

---

## 2. Development Environment Setup

### Project Structure — Where Everything Goes

This is probably the question on your mind, so let me answer it directly. Below is the final structure your project will grow into. **But — and this is important — you don't create all of this now.**

Build it incrementally:
- **Phase 1:** You only need `app/main.py`. That's it.
- **Phase 2:** You add `app/models/` (for your database table definition) and `app/db/` (for your database connection setup).
- **Phase 3:** You add `app/security/` (for HMAC verification) and `app/services/` (for business logic).

Here's the full picture so you can see where everything ends up:

```
project-root/
├── app/
│   ├── __init__.py
│   ├── main.py              ← Your FastAPI app lives here. Phase 1 starts and ends here.
│   ├── config.py            ← Settings class (reads from .env). Add when you need config.
│   ├── models/              ← Your database table definitions go here.
│   │   ├── __init__.py
│   │   └── webhook_event.py ← The WebhookEvent class (Phase 2)
│   ├── schemas/             ← Pydantic models for request/response validation
│   │   ├── __init__.py
│   │   └── ...
│   ├── routes/              ← Route handlers (if you move them out of main.py later)
│   │   ├── __init__.py
│   │   └── ...
│   ├── services/            ← Business logic (Phase 3)
│   │   ├── __init__.py
│   │   └── ...
│   ├── db/                  ← Database engine, session factory, dependency function
│   │   ├── __init__.py
│   │   └── session.py       ← Engine + session setup (Phase 2)
│   └── security/            ← HMAC verification logic (Phase 3)
│       ├── __init__.py
│       └── ...
├── alembic/                 ← Migration scripts (generated by Alembic in Phase 2)
├── tests/
│   └── ...
├── alembic.ini
├── pyproject.toml
├── .env                     ← Secrets go here. NEVER commit this file.
└── .gitignore
```

**Why is the code split across so many files?** Imagine you're a new developer joining a team. You're asked to fix a bug in the HMAC signature check. If everything lives in one massive `main.py`, you have to read through hundreds of lines of database code, route handlers, and config loading just to find the 20 lines that matter. But if security logic lives in `app/security/`, you open that folder, read those files, and you're done. The file system itself becomes a map of the application's responsibilities.

This isn't just about organisation for its own sake — it's about **making the codebase navigable for humans**. The person reading your code six months from now (probably you) shouldn't need to understand the entire application to fix one part of it. Each file should be answerable in one sentence: "What does this file do?" If you can't answer that clearly, the file is doing too much.

### Why This Structure? (The Reasoning, Not Just the Layout)

I don't want you to memorize this — I want you to understand *why* each directory exists. Each separation prevents a specific problem:

| Directory | What goes in it | Why it's separate | What goes wrong if you don't separate it |
|-----------|----------------|-------------------|----------------------------------------|
| `routes/` | Functions that accept HTTP requests and return responses | Routes should be **thin** — they parse input and call other functions. If your route handler is more than ~15 lines, you're putting logic in the wrong place. | You can't test your business rules without spinning up a full HTTP server. |
| `services/` | Business logic: validation, transformation, decision-making | This is where things like your HMAC check live. You can test this code by calling the function directly — no HTTP server needed. | Testing becomes painful. Changing a validation rule means digging through HTTP-layer code. |
| `models/` | Database table definitions (ORM classes) | Your database table and your API response shape are almost never the same thing. Keeping them separate means changing one doesn't break the other. | Adding a database column forces you to change your API response. Internal fields (like auto-generated IDs) leak into your public API. |
| `schemas/` | Pydantic models for request/response data shapes | FastAPI uses these to validate incoming data and auto-generate API docs. | No input validation at the boundary. Bad data reaches your business logic and causes confusing errors deep in the stack. |
| `db/` | Database connection pooling, session lifecycle | Database connections are expensive. This layer manages the pool and makes sure connections get cleaned up properly. | Connection leaks. After a few hundred requests, your database runs out of connections and everything stops. |
| `security/` | Cryptographic verification, secrets handling | Security code is audited and tested separately. Isolating it makes that practical. | Security logic is scattered across multiple handlers. A refactor in one handler accidentally disables verification in another. |

**The real-world principle here:** In professional codebases, you organise code by *responsibility*, not by convenience. If you ask "where does the HMAC check go?", the answer isn't "wherever it's closest to where it's used" — it's "in the security module, because that's where all authentication logic lives, so anyone auditing security knows exactly where to look."

### Environment Management

Every Python project gets its own **virtual environment** — an isolated set of installed packages. Without one, installing something for this project can break a different project. If you haven't been doing this, start now. It's non-negotiable.

**The concept:** Python has one global place it puts installed packages. If Project A needs `sqlalchemy==2.0` and Project B needs `sqlalchemy==1.4`, they fight. A virtual environment gives each project its own separate set of packages. You create one with `python -m venv .venv`, activate it, and now `pip install` only affects that one project.

**Think of it like separate kitchens.** Each project gets its own kitchen with its own pantry. One project's ingredients don't contaminate another's.

- Use `pyproject.toml` as your dependency file (not `requirements.txt` — that's the old way). Tools like `uv`, `pdm`, or `poetry` all support this. `uv` is the fastest and simplest right now — use it.
- Store secrets (database URLs, webhook secrets, API keys) in a `.env` file. Load them with `pydantic-settings` — this gives you typed, validated config with almost no boilerplate. **Never hardcode secrets. Never commit `.env`.**

> **→ Before you move on, confirm:**
> - [ ] Your virtual environment is active (your terminal prompt shows `(.venv)` or similar)
> - [ ] `fastapi` and `uvicorn` are installed (run `python -c "import fastapi; print(fastapi.__version__)"`)
> - [ ] You have `ngrok` installed and can run `ngrok version`
> - [ ] You have a GitHub repository where you can configure a webhook

---

## 3. Phase 1 — The Catch & Log

> **Goal:** Get a working endpoint that receives a real GitHub webhook and logs it. By the end, you should see a live `push` event arrive at your server.
>
> **What you're learning:** HTTP fundamentals, async Python, FastAPI routing, and the developer toolchain (tunneling, logging).
>
> **Where your code goes:** Everything in this phase lives in `app/main.py`. Just that one file. Don't create any other directories yet.
>
> **Why `main.py`?** Right now, your entire application is one thing: receive a request and log it. There's no database logic, no security logic, no separate business rules. When an application does one thing, putting it in one file is the correct choice — splitting it up would just add complexity with no benefit. You'll split things out *when the need arises* in later phases. This is a professional instinct to develop: don't add structure until the code demands it.

**How to work through this section:** Read sections 3.1 through 3.6 first — all the concept material. Then go to section 3.7 (the Practical Workflow) and build it step by step. The concepts explain *why*; the workflow tells you *what to do*.

### Core Concepts — Read These First

#### 3.1 REST and the HTTP Contract

Before you write endpoint code, you need to understand the protocol your code speaks. HTTP is a **request-response** protocol — every interaction is a client sending a request and a server sending a response. REST is a set of conventions built on top of HTTP for how to use its features meaningfully.

**The things you need to internalise:**

- **Methods** define intent: `GET` (read), `POST` (create), `PUT` (full replace), `PATCH` (partial update), `DELETE` (remove). A webhook is always a `POST` because GitHub is *creating* an event notification on your server.

- **Status codes** are how the server communicates what happened. Each range has a meaning:
  - `2xx` — Success: understood and handled
  - `3xx` — Redirection: go look somewhere else
  - `4xx` — Client error: *you* (the sender) messed up
  - `5xx` — Server error: *I* (the receiver) messed up
  
  The ones you'll use right away: `200` (OK), `201` (created), `400` (bad request), `403` (forbidden), `404` (not found), `500` (server error).

- **Headers** carry metadata. `Content-Type` tells the recipient how to parse the body. `X-Hub-Signature-256` carries GitHub's HMAC signature. Think of headers as the envelope, and the body as the letter inside.

- **The body** is the payload. For webhooks, it's always JSON.

**The key shift for webhooks:** A webhook is a **server-to-server POST**. No browser, no human, no interactive session. GitHub is the client; you are the server. This flips the usual mental model — you're not reaching out to fetch data, you're sitting still and waiting for data to come to you. Your server is a **listener**, not an **initiator**.

> **Before you code:** Go to GitHub's [Webhook events and payloads](https://docs.github.com/en/webhooks/webhook-events-and-payloads) page. Find the `push` event and study its payload schema. Knowing what to expect makes debugging much easier — when something weird shows up, you'll immediately know whether it's your code or an unusual payload.

#### 3.2 Decorators — How FastAPI Routing Works

FastAPI uses Python decorators to bind a function to an HTTP route. Before you use them in anger, make sure you understand what a decorator actually *does* — because it's not magic, it's just a function that wraps another function.

**What a decorator is:** A function that takes another function as input and returns a new function (usually wrapping the original with extra behaviour). The `@` syntax is just shorthand for an assignment.

**Why you need to understand this:** If you treat decorators as magical incantations, you'll struggle to debug routing issues, understand middleware, or write your own reusable patterns later. They're just functions wrapping functions.

```python
# What a decorator actually IS:

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

**How FastAPI uses this:** When you write `@app.post("/webhook")`, FastAPI's `post` method is a decorator factory. It registers your function in an internal **routing table** — keyed by HTTP method (`POST`) and path (`/webhook`). When a request arrives matching that method and path, FastAPI calls your function. The decorator doesn't change what your function *does* — it changes *when* it gets called.

> **Try this after you've got your first endpoint working:** Print `app.routes` to see the routing table FastAPI built. You'll see your function listed alongside FastAPI's auto-generated routes (like `/docs` and `/openapi.json`). It makes the abstraction concrete.

#### 3.3 Async Functions — Why `async def` Matters

FastAPI runs on an ASGI server (Uvicorn). When you write `async def`, you're telling the event loop: *"This function will give up control when it hits I/O (network calls, database queries, file reads), so you can serve other requests in the meantime."*

**Why this matters for your webhook receiver:**

1. Multiple webhook deliveries can arrive at the same time (several developers pushing to the same repo).
2. Your handler will do I/O: reading the request body, writing to the database, forwarding to other services.
3. If you use plain `def`, FastAPI puts it in a thread pool — it still works, but you lose the efficiency of the event loop.

**The waiter analogy:** A synchronous waiter takes one table's order, walks to the kitchen, stands there until the food is ready, walks back, and only *then* takes the next table's order. An async waiter takes the order, sends it to the kitchen, immediately goes to the next table, and picks up the food when the kitchen signals it's ready. Same waiter, same kitchen, dramatically different throughput.

**Three things you must understand about `async/await`:**

1. **`async def` declares a coroutine** — a function that *can* be suspended and resumed. It doesn't run any differently until you use `await` inside it.

2. **`await` is the pause point** — it tells the event loop: "I'm waiting for this I/O. Go handle something else and come back to me when it's done." Without `await`, your coroutine blocks the event loop.

3. **Forgetting `await` is a silent bug** — If you call an async function without `await`, you get a coroutine *object*, not a result. Python might warn you, but your code won't crash — it just silently does nothing. This is one of the nastiest async bugs because the code *appears* to work.

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

> **Watch out for this:** Writing `body = request.body()` instead of `body = await request.body()`. The first gives you a coroutine object (something like `<coroutine object Request.body at 0x...>`), not actual bytes. Your code won't crash — it'll pass this useless object around, and you'll get confusing errors later when you try to use it as bytes. **If you see a type error mentioning "coroutine", you almost certainly forgot an `await`.**

#### 3.4 FastAPI Route Handlers — Request Anatomy

FastAPI gives you multiple ways to access incoming request data. For webhooks, you need **two** things: the raw body (for HMAC verification later) and the parsed JSON (for processing). Here's the pattern:

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

**Why `Request` and not a Pydantic model here?** This is a deliberate architectural choice. When you use a Pydantic type as a parameter, FastAPI consumes the request body to validate it. But to verify an HMAC, you need the **exact raw bytes** that were signed — before any parsing. You can't get those back once FastAPI has consumed them into a model. This is a webhook-specific constraint.

**The general principle:** When you need both raw bytes and parsed data, use `Request` directly. When you only need validated structured data (like in a typical REST endpoint), use Pydantic models — they give you automatic validation and documentation.

> **Understanding `request.body()` vs `request.json()`:**
> - `request.body()` returns the raw bytes exactly as they arrived over the wire. This is what the HMAC was computed over.
> - `request.json()` calls `request.body()` internally, then parses the bytes as JSON. Starlette (the framework under FastAPI) caches the body, so calling both doesn't read from the network twice.
> - You should call `body()` first for clarity, but the caching means the order doesn't affect correctness.

#### 3.5 Logging — Your First Debugging Tool

Before we get to the tunnel and GitHub configuration, let's talk about proper logging. This isn't an afterthought — it's one of the most important habits you'll build.

**Why `logging` and not `print`:**

| `print()` | `logging` |
|-----------|-----------|
| No timestamps | Automatic timestamps |
| No severity levels | DEBUG, INFO, WARNING, ERROR, CRITICAL |
| Can't be filtered | Filter by level, module, or handler |
| Can't be sent to files | Route to files, services, or monitoring tools |
| Gone when you remove debug prints | Stays as permanent instrumentation |
| Useless in production | Essential in production |

**The mindset shift:** `print` is for quick throwaway debugging. `logging` is **permanent instrumentation** baked into your application. Good logging is like security cameras — you don't watch them every moment, but when something goes wrong at 2 AM, you rewind the tape and see exactly what happened.

```python
import logging

# Configure once, at application startup
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
)

logger = logging.getLogger("webhook_hub")

# Use %s formatting — NOT f-strings
# The string is only formatted if the log level is active
logger.info("Received delivery %s for event %s", delivery_id, event_type)
logger.warning("Missing signature header on delivery %s", delivery_id)
logger.error("Database write failed for delivery %s: %s", delivery_id, error)
```

> **Why `%s` and not f-strings?** With `logger.info(f"Got {data}")`, the f-string is evaluated *immediately*, even if the log level is set to WARNING and the message will be thrown away. With `logger.info("Got %s", data)`, the formatting only happens if the message will actually be emitted. For hot code paths, this matters. Build the habit now.

#### 3.6 ngrok — Exposing Localhost

ngrok exposes your local server to the public internet by creating a reverse tunnel. Here's what actually happens:

1. You start your FastAPI server on `localhost:8000`.
2. You run `ngrok http 8000`.
3. ngrok opens an **outbound-only** connection to ngrok's servers — no ports are opened on your machine.
4. ngrok gives you a public URL (like `*.ngrok-free.app`) and routes incoming traffic through the tunnel to your local port.
5. The ngrok terminal shows you real-time request traffic — extremely useful for debugging.

**Free tier limitation:** ngrok's free tier gives you a new URL each time you restart the tunnel. You'll need to update your GitHub webhook config each time. Annoying, but not a blocker — and it means you'll get comfortable navigating GitHub's webhook settings. If you want a permanent URL later, ngrok has paid plans, or you can explore **Cloudflare Tunnel** (which offers stable URLs on the free tier).

> **ngrok's Web Inspector:** When ngrok runs, it starts a local inspector at `http://127.0.0.1:4040`. It shows every request that came through — headers, body, status code, timing. It's like a built-in network debugger. Use it a lot during development.

> **Common mistake:** Starting ngrok before starting your FastAPI server. ngrok forwards traffic to `localhost:8000` — if nothing is listening there, GitHub's delivery fails. **Always start Uvicorn first, then ngrok.** If you restart your server, ngrok keeps working (no need to restart the tunnel).

---

### Practical Workflow — Phase 1

**You've read the concepts above. Now build it.** Do each step fully before moving to the next.

Everything in this phase goes in **`app/main.py`**. That's the only file you're editing.

1. **Create a minimal FastAPI application** with a single POST endpoint at `/webhook`.
2. **Have the handler log the received headers and body** using Python's `logging` module (not `print`). Log the event type, delivery ID, and a truncated version of the body.
3. **Run the app with Uvicorn.** You should see it listening on `http://127.0.0.1:8000`.
4. **Test locally first** with `curl` before involving GitHub. This eliminates tunneling and GitHub as variables:
   ```bash
   # Send a fake webhook payload to verify your endpoint works
   curl -X POST http://localhost:8000/webhook \
     -H "Content-Type: application/json" \
     -H "X-GitHub-Event: push" \
     -H "X-GitHub-Delivery: test-123" \
     -d '{"ref": "refs/heads/main", "commits": []}'
   ```
   You should see your log output appear. If it doesn't, fix your handler before adding more complexity.
5. **Start ngrok** pointing at Uvicorn's port: `ngrok http 8000`. Copy the `Forwarding` URL.
6. **Configure a GitHub webhook** to point at your ngrok URL + endpoint path (e.g., `https://xxxx-xx-xx.ngrok-free.app/webhook`). You'll need to update this URL each time you restart ngrok.
7. **Push a commit** and watch the log output. This is the milestone moment — a real GitHub event arriving at your server.
8. **Study the payload carefully.** Open it in a JSON viewer. Read every field. This payload is the input your entire system processes — you need to know it well.

> **If you push a commit and nothing shows up in your logs:**
> 1. Check GitHub's webhook delivery history (Settings → Webhooks → Recent Deliveries). GitHub shows you the exact request it sent and the response it received.
> 2. If GitHub shows a connection error → your tunnel isn't running or points at the wrong port.
> 3. If GitHub shows a response from your server (even an error) → the tunnel works, the issue is in your handler.
> 4. If GitHub shows a 404 → your endpoint path doesn't match (e.g., GitHub is POSTing to `/webhook` but your route is `/webhooks` — note the plural).

---

### 🔒 Checkpoint — Phase 1

Stop here. Answer these before moving to Phase 2. If you can't answer one, re-read the relevant concept section — don't keep building on a shaky foundation.

1. **Data Flow:** When GitHub sends a webhook POST to your ngrok URL, describe the exact sequence of network hops the request takes before reaching your `async def` handler. What happens at each hop?

   *Think about: DNS resolution → ngrok's servers → the tunnel → your local port → Uvicorn → FastAPI → your function. There are at least 4 distinct hops.*

2. **Async Reasoning:** You used `await request.body()`. If you removed the `await` and just wrote `body = request.body()`, what would `body` contain? Why wouldn't your code crash immediately? Why is this bug particularly hard to find?

   *Think about: what happens when you later pass `body` to a function expecting `bytes`. Does Python complain immediately, or somewhere unexpected?*

3. **Architecture Decision:** Your handler currently logs to the console. The console is ephemeral — restart the server and all events are gone. What do you need to add to make event data survive a restart, and what's the minimum set of fields you'd store per event to make it useful?

   *Think about: "Show me all push events to main in the last 24 hours" — what columns does that query need?*

---

## 4. Phase 2 — The Vault

> **Goal:** Persist every webhook event to PostgreSQL so nothing is lost when the server restarts. By the end, you'll have a running database, an ORM model, a migration, and a refactored handler.
>
> **What you're learning:** Why databases exist, how ORMs work, the session/transaction lifecycle, dependency injection, and schema migrations. These concepts underpin virtually every backend application.
>
> **Where your code goes:**
> - `app/models/webhook_event.py` — your database table definition
> - `app/db/session.py` — your database engine, session factory, and dependency function
> - `app/main.py` — refactor your handler to use the database
> - `alembic/` — migration scripts (generated by Alembic)

**How to work through this section:** Read sections 4.1 through 4.7 first — all the concept material. Then go to section 4.8 (the Practical Workflow) and build it step by step. The workflow tells you exactly which files to create and in what order.

### Core Concepts — Read These First

#### 4.1 Why a Database? Why Not Just Files?

Before reaching for PostgreSQL, understand the *problem* it solves. When you first need to persist data, the instinct is: *"I'll just write it to a file."* A JSON file, a CSV, a log file. For a hobby script that runs once a day, that's fine. For a server receiving concurrent webhook deliveries, it'll break. Here's exactly why.

**Problem 1: Concurrent writes.** Multiple webhook deliveries can arrive at the same time. Appending to a file from multiple async handlers creates a race condition — two handlers both try to write at the same position, and one overwrites the other. You get partial writes, garbled JSON, corruption. Databases handle this with **transactional isolation**: each write is atomic and serialised, even under heavy load.

**Problem 2: Queryability.** "Show me all `push` events to `main` in the last 24 hours." With a JSON file, you load the entire thing into memory, deserialise every object, and loop through them in Python. With 100 events, that's fine. With 100,000 events, you're loading hundreds of megabytes and scanning every one. A database uses a `WHERE` clause and **indexes** (pre-built lookup structures) to return results in milliseconds.

**Problem 3: ACID guarantees.** These are the four properties that make a database fundamentally different from a file:
  - **Atomicity** — a write either fully succeeds or fully fails. No partial records.
  - **Consistency** — the data always satisfies your defined constraints (e.g., "this column cannot be null").
  - **Isolation** — concurrent transactions don't interfere with each other.
  - **Durability** — committed data survives a crash. The database writes to a **write-ahead log** (WAL) before confirming, so even if the process dies mid-write, the data is recoverable.

> **Making it concrete:** Your server receives a webhook. Midway through writing to a file, the process crashes. With a file: you have a half-written JSON object that corrupts the entire file. With a database: the incomplete transaction is rolled back. The data is either fully there or not there at all. That's atomicity and durability working together.

#### 4.2 Relational Databases — The Mental Model

PostgreSQL is a **relational database**. The mental model: strictly enforced spreadsheets with superpowers.

- Each **table** is a noun (e.g., `orders`, `webhook_events`).
- Each **column** has a name, a data type, and optional constraints (`NOT NULL`, `UNIQUE`, `FOREIGN KEY`).
- Each **row** is one record.
- **Primary keys** uniquely identify rows. Use auto-incrementing integers or UUIDs.
- **Indexes** are auxiliary structures that make queries on specific columns fast, at the cost of slightly slower writes. Like the index at the back of a textbook — without it, finding a topic means reading every page.

> **PostgreSQL's `JSONB` column type** is particularly useful for your project. It stores JSON in a binary format that supports indexing and querying into nested fields. You can store the full webhook payload as-is *and* still query into it. You get the structure of relational columns for your metadata *and* the flexibility of a document store for the raw payload.

#### 4.3 The ORM — Why an Abstraction Layer?

An **Object-Relational Mapper** (ORM) lets you interact with database tables as Python classes and rows as Python objects. Instead of writing raw SQL strings, you call methods.

**Why not just write SQL?** You *can*, and for complex analytical queries you sometimes *should*. But for standard operations, raw SQL has problems:

1. **SQL injection.** Building queries with string concatenation is the #1 web security vulnerability. ORMs use parameterised queries by default.
2. **Boilerplate.** Manually mapping result rows to Python objects is tedious and error-prone.
3. **Migration management.** ORMs integrate with tools that version-control your schema changes.

**SQLAlchemy** is the dominant ORM in Python. It's more verbose than some alternatives (SQLModel, Tortoise ORM), and that's actually helpful at your stage — the verbosity forces you to understand each layer (engine, session, model) instead of hiding it behind convenience methods.

**The three layers of SQLAlchemy** (confusing these is a common source of bugs):

| Layer | What It Is | Analogy |
|-------|-----------|---------|
| **Engine** | The connection pool manager. Created *once* at startup. | The phone line to the database — always open, shared by everyone. |
| **Session** | A "workspace" that tracks changes to objects. One per request. | A notepad where you draft changes before submitting them. |
| **Model** | A Python class that maps to a database table. | A blank form — each instance is one filled-out form (one row). |

Here's what a model looks like:

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

This class *is* your table definition. When the migration tool reads it, it generates the `CREATE TABLE` SQL. When you create an instance of `Order(...)`, you're building an in-memory representation of a row. When you add it to a session and commit, the ORM generates and runs the `INSERT` statement.

> **The `Base` class:** Every model inherits from `Base`. This is how SQLAlchemy's migration tool discovers which classes represent database tables. If a class inherits from `Base`, SQLAlchemy *expects* a corresponding table to exist.

**Where does this code go in your project?** The model class goes in `app/models/webhook_event.py`. The `Base` class goes in `app/models/__init__.py` (or its own file like `app/models/base.py`).

**Why a separate `models/` directory — why not just define the class in `main.py`?** Two reasons:

1. **Your model serves two masters.** Your route handler needs to import `WebhookEvent` to create instances. But Alembic *also* needs to import it to generate migrations — and Alembic doesn't run through FastAPI. If the model lives in `main.py`, Alembic has to import your entire FastAPI app (with all its routes, dependencies, and startup logic) just to read a class definition. That's fragile and will break in confusing ways. Putting the model in its own file means both your app and Alembic can import it independently.

2. **Models describe your data, not your HTTP interface.** A model says "here's what a webhook event looks like in the database." A route handler says "here's what happens when a request arrives." These are different concerns, and they change for different reasons. When you add a column to your model, you shouldn't have to scroll past 50 lines of HTTP handling code to find it.

**Why does `Base` go in `__init__.py`?** Because every model in the `models/` directory needs to inherit from it. Putting `Base` in the directory's `__init__.py` makes the import clean: `from app.models import Base`. If you had multiple model files (say, `webhook_event.py` and `user.py`), they'd all import `Base` from the same place. It's the shared foundation for the entire models directory.

#### 4.4 Sessions and the Unit of Work Pattern

The session is the central concept. A session is **not** a database connection — it's a **staging area** that tracks changes and flushes them to the database when you commit.

**How it works:**

1. You stage objects: `session.add(new_event)` marks a new record for insertion.
2. The session tracks everything in memory — nothing goes to the database yet.
3. When you call `session.commit()`, the session generates SQL, sends it in a single transaction, and the database either accepts or rejects it atomically.
4. If anything fails before commit, you call `session.rollback()` to discard all staged changes.

**Why this matters:** If your handler crashes *after* staging an object but *before* committing, nothing gets written. No half-written records. The session also batches work — if you insert 10 records, you stage all 10 and commit once, sending a single transaction instead of 10 separate ones.

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

**Where does this code go?** The engine and session factory go in `app/db/session.py`. The dependency function (next section) also goes in that file or in `app/db/__init__.py`.

**Why `app/db/session.py` — why not put the engine in `main.py`?** The engine and session factory are **infrastructure** — they manage the connection pool to your database. Your route handler doesn't need to know *how* a session gets created, just that one shows up when it asks for it. If the engine setup lives in `main.py`, you can't change your database configuration (switching from SQLite for tests to Postgres for production, for example) without editing the same file that defines your HTTP routes. Keeping them separate means you can swap out the entire database layer without touching a single route handler.

**Why its own directory (`db/`) and not just a single file?** Right now you'll only have `session.py` in there, and that might feel like overkill. But this directory is where all database-related infrastructure lives: connection setup now, and potentially things like custom query helpers or health-check functions later. The directory gives you a clear place to put those things when the time comes.

> **`expire_on_commit=False` — Why you need this:** By default, SQLAlchemy invalidates all loaded attributes on an object after commit, forcing a reload on next access. In sync code, this triggers a transparent database query. In async code, it triggers a *synchronous* I/O call inside an async context, which **raises an error**. Setting `expire_on_commit=False` keeps the in-memory attributes valid after commit. You'll hit this error if you forget — now you know why.

> **Common mistake — Session scope:** Never create one session at startup and reuse it for all requests. Sessions are lightweight and meant to be short-lived — one per request. A shared session accumulates stale state and creates concurrency bugs that are extremely hard to track down. Use the dependency injection pattern (next section) to get a fresh session per request.

#### 4.5 Dependency Injection — FastAPI's Session Pattern

FastAPI uses **dependency injection** to provide resources (like database sessions) to route handlers. This isn't a framework quirk — it's a deliberate pattern that keeps your route logic clean.

**The concept:** Instead of your handler *creating* the resources it needs, it *declares* what it needs, and the framework provides them. Your handler says "I need a database session" and FastAPI gives it one — creating it before the handler runs and cleaning it up afterwards.

**Why this matters:**

1. **Testability.** In tests, you swap in a test database session. Your handler code doesn't change.
2. **Lifecycle safety.** The `finally` block guarantees cleanup even if the handler crashes. No leaked sessions.
3. **Single Responsibility.** The handler focuses on business logic. Session plumbing is someone else's problem.
4. **Reusability.** Same dependency used by every handler that needs a session, no duplicate code.

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

**How `yield` works here:** This is a generator-based dependency. FastAPI calls the function, which runs up to the `yield` and pauses — the yielded value (the session) goes to your handler. After your handler finishes (or crashes), FastAPI resumes the generator, which enters the `finally` block and closes the session. It's Python's context manager pattern, adapted for dependency injection.

**Where does this go?** The `get_db_session` function lives in `app/db/session.py` alongside your engine and session factory. Your handler in `app/main.py` imports it.

**Why does the dependency function live next to the engine, not next to the handler?** Because `get_db_session` is *about the database*, not about HTTP. It knows how to create sessions and clean them up — that's database infrastructure. Your handler just declares `db: AsyncSession = Depends(get_db_session)` and doesn't care how the session was made. If you later change from PostgreSQL to a different database, you'd change `app/db/session.py` and nothing else. The handler doesn't need to know.

> **If `yield` in this context confuses you:** You've probably only seen `yield` producing sequences of values. The key insight is that `yield` can also split a function into "setup" (before yield) and "teardown" (after yield) phases. FastAPI uses this to manage resource lifecycles — acquire before the handler, release after.

#### 4.6 Migrations — Version Control for Your Schema

Your database schema will change over time. You'll add columns, rename tables, add indexes. You can't just drop and recreate the database each time — in production, that destroys data.

**Alembic** is the migration tool for SQLAlchemy. Think of it like `git` for your database schema:

1. You change your model (e.g., add a column).
2. You run `alembic revision --autogenerate -m "add priority column"`.
3. Alembic compares your model against the current database schema and generates a migration script.
4. You **review** the script (auto-generated migrations are suggestions, not gospel).
5. You run `alembic upgrade head` to apply it.

**Why you must review migrations:** Alembic's auto-generation is a diff tool, not an intelligence. It can detect column additions and removals, but it can't detect *renames*. If you rename a column, Alembic sees a drop + add, which destroys all data in that column. **Always read the generated migration before applying it.**

> **The most common migration mistake:** You rename `event_type` to `github_event` in your model. Alembic generates a migration that drops `event_type` and adds `github_event`. All existing data in that column is permanently deleted. The correct migration uses `op.alter_column('table', 'event_type', new_column_name='github_event')`. **If a generated migration contains both `drop_column` and `add_column` for what should be a rename, do not apply it.** Edit the script manually.

#### 4.7 Designing Your Data Model — What to Store

This is a design question, not a syntax question. Think about it before you code — these decisions affect every query you write later.

**Start with the questions you want to answer:**

| Question You Want to Answer | Column(s) You Need |
|---|---|
| "What events arrived in the last hour?" | `created_at` (with an index) |
| "Show me all push events" | `event_type` |
| "Has this delivery already been processed?" | `delivery_id` (UNIQUE constraint) |
| "What repo generated this event?" | `repository` (extracted from payload) |
| "What was the full payload?" | `payload` (JSONB) |

**The hybrid approach:** Store the full JSON payload for auditability (so you never lose data), but also extract key fields into dedicated columns for fast querying. You're trading a small amount of storage for dramatically faster queries. This is standard practice in event-sourcing systems.

**Idempotency:** GitHub sends a unique delivery ID with every webhook. Store it and add a `UNIQUE` constraint — this prevents duplicate records if GitHub retries a delivery. Without it, a network hiccup could cause your system to process the same event twice.

---

### Practical Workflow — Phase 2

**You've read the concepts above. Now build it.** Each step tells you what to do and where to put it.

1. **Start PostgreSQL.** Use Docker to run a local Postgres container:
   ```bash
   docker run --name webhook-db -e POSTGRES_PASSWORD=secret -d -p 5432:5432 postgres
   ```
   Verify it's running with `docker ps`. Try connecting with a database tool (DBeaver, pgAdmin, or `psql`) to confirm.

2. **Install dependencies.** Add `sqlalchemy`, `asyncpg`, and `alembic` to your project:
   ```bash
   uv add sqlalchemy asyncpg alembic
   ```

3. **Add the database URL to `.env`:**
   ```
   DATABASE_URL=postgresql+asyncpg://postgres:secret@localhost:5432/postgres
   ```

4. **Create the Base class.** Create the file `app/models/__init__.py`. In it, define your `DeclarativeBase`:
   ```python
   from sqlalchemy.orm import DeclarativeBase

   class Base(DeclarativeBase):
       pass
   ```
   This is the parent class all your models inherit from. It lives in `__init__.py` because it's the shared foundation for this directory — any model file you add later will import `Base` from here. Think of it as the "root" that ties all your table definitions together so Alembic can discover them.

5. **Create the WebhookEvent model.** Create the file `app/models/webhook_event.py`. Define a `WebhookEvent` class that inherits from `Base`, with columns for `id`, `delivery_id`, `event_type`, `payload` (JSON), and `created_at`. Think about which columns need constraints (NOT NULL, UNIQUE) and which need indexes. Refer back to section 4.7 for guidance.

   Why its own file? Because this class describes your data structure, and both your FastAPI app *and* Alembic need to import it independently. Keeping it separate from your routes means neither system drags in code it doesn't need.

6. **Set up the database engine and session.** Create the file `app/db/session.py`. In it:
   - Create the async engine using `create_async_engine` with the URL from your `.env`
   - Create a session factory using `sessionmaker`
   - Write the `get_db_session` dependency function (the `yield`-based one from section 4.5)
   
   Also create `app/db/__init__.py` (it can be empty, or re-export what you need).
   
   Why `db/session.py`? This file answers one question: "How does my app connect to the database and get sessions?" Your handler shouldn't know or care about connection strings, pool sizes, or session cleanup — it just asks for a session and gets one. Isolating this here means you can change your database setup (different URL, different pool config, test vs production) without touching any of your route handlers.

7. **Initialise Alembic.** Run this from your project root:
   ```bash
   alembic init -t async alembic
   ```
   This creates `alembic.ini` and an `alembic/` directory with migration templates.

8. **Connect Alembic to your models.** Open `alembic/env.py` and:
   - Import your `Base` from `app.models`
   - Set `target_metadata = Base.metadata`
   - Point it at your database URL (or load it from your `.env`)
   
   This is the step most people struggle with. Read the generated `env.py` carefully — it has comments telling you where to make changes. You may also need to add your project root to `sys.path` so that Alembic can import from `app/`.

9. **Generate and run the first migration:**
   ```bash
   alembic revision --autogenerate -m "create webhook_events table"
   ```
   **Read the generated migration script** in `alembic/versions/`. Verify it creates the table and columns you expect. Then apply it:
   ```bash
   alembic upgrade head
   ```

10. **Update your handler.** In `app/main.py`:
    - Import your `get_db_session` dependency and your `WebhookEvent` model
    - Add `db: AsyncSession = Depends(get_db_session)` to your handler's parameters
    - Extract the relevant headers (`X-GitHub-Delivery`, `X-GitHub-Event`)
    - Create a `WebhookEvent` instance with the extracted data
    - Add it to the session and commit

11. **Verify it works.** Trigger a webhook from GitHub. Then connect to your database and run:
    ```sql
    SELECT * FROM webhook_events;
    ```
    You should see your event. Now **restart your server** and query again — the data should still be there. That's persistence.

> **Also test the failure case:** What happens if you send a webhook but your database is down? Does your server crash? Does it return a 500? Does it log a useful error? How your application handles failures is as important as how it handles success.

---

### 🔒 Checkpoint — Phase 2

Stop here. Answer these before moving to Phase 3:

1. **Session Lifecycle:** You open a session, add an ORM object, and call `await session.commit()`. Then your handler raises an exception *after* the commit but *before* returning the HTTP response. Is the data in the database? Why? Now flip it: the exception happens *before* `commit()`. What happens to the staged object?

   *Key insight: the database transaction and the HTTP response are independent operations. Once committed, it's committed — the response doesn't matter to the database.*

2. **Schema Design Trade-off:** You're choosing between storing the entire payload as one big `JSONB` column versus extracting every field into separate typed columns. Argue both sides. What does the hybrid approach look like, and why is it standard for event-sourcing?

   *Think about: "find all push events touching file X in repo Y". Is that information in a structured column or buried in the JSON? What are the index implications?*

3. **Migration Danger:** You have a table with 50,000 rows. You rename a column in your model and run `alembic revision --autogenerate`. What does the generated migration *actually* do? What's the consequence, and how would you fix it?

   *Alembic sees the old column disappear and a new one appear. It doesn't know they're the same column. What does "drop column" + "add column" do to existing data?*

---

## 5. Phase 3 — The Secure Router

> **Goal:** No unsigned or forged request should be processed. Add HMAC verification, an idempotency guard, and at least one background task. After this, your pipeline is: verify → persist → dispatch.
>
> **What you're learning:** Cryptographic message authentication, secure comparison, the dependency-as-guard pattern, background task dispatch, and request pipeline design.
>
> **Where your code goes (and why):**
> - `app/security/verify.py` — your HMAC verification function. *Why here?* Security logic gets its own directory because it's audited and tested separately from everything else. If someone asks "how does your app verify requests?", they look in one place.
> - `app/config.py` — your Settings class (reads webhook secret from `.env`). *Why here?* Configuration is used by multiple modules (your db setup needs the database URL, your security module needs the webhook secret). A central config file means every module imports settings from one place instead of each one calling `os.getenv()` independently.
> - `app/main.py` — update your handler to use verification and background tasks. *Why still here?* Your handler is still the "orchestrator" that ties the pieces together: call verify, call persist, schedule background work. It imports from the other modules but the flow logic stays in the route.

**How to work through this section:** Read sections 5.1 through 5.6 first. Then go to section 5.7 which shows the complete pipeline flow, and section 5.8 for configuration. The Practical Workflow at the end of Phase 2 set up the pattern — this phase follows the same approach.

### Core Concepts — Read These First

#### 5.1 Why HMAC? The Trust Problem

Your endpoint is a public URL. Anyone who discovers it can send a POST request. Without verification, an attacker could forge payloads — fake push events, fake deployment statuses, fake security alerts. Your server would happily process them.

**The threat model:** Your webhook URL has no login page, no API key, no OAuth token. The URL itself is the only thing protecting it — and URLs can be guessed, leaked, or discovered through scanning. You need a way to answer: "Did this request *actually* come from GitHub, or is someone pretending?"

**HMAC (Hash-based Message Authentication Code)** solves this with a shared secret:

1. You and GitHub both know a secret string (you set it when configuring the webhook).
2. GitHub computes `HMAC-SHA256(secret, raw_request_body)` and puts the hex digest in the `X-Hub-Signature-256` header as `sha256=<digest>`.
3. Your server receives the request, computes the same HMAC using the same secret, and compares.
4. If the digests match: the message is **authentic** (it came from someone who knows the secret) and has **integrity** (it hasn't been tampered with in transit).

**Why not just check the IP address?** GitHub publishes its webhook source IP ranges, but IPs can be spoofed, and the ranges change. HMAC verification is more robust because it proves the sender *knows a secret*, not just that they're sending from a particular address.

**Why not encryption?** HMAC is not encryption — the payload travels in plaintext (though TLS encrypts the wire). HMAC provides **authentication** (who sent it) and **tamper detection** (was it modified), not **confidentiality** (can someone read it). Since the payload is just event metadata (commit messages, branch names), confidentiality isn't the concern — authenticity is.

#### 5.2 The Cryptographic Implementation

Python's `hmac` and `hashlib` modules handle everything. Here's the generic pattern:

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

**Understanding each piece:**

- **`secret.encode("utf-8")`** — HMAC works on bytes. The secret must be encoded first. Both you and GitHub must use the same encoding (UTF-8).
- **`msg=payload_body`** — This must be the *exact* raw bytes that GitHub signed. If you modify the body in any way (parse and re-serialise, strip whitespace), the HMAC won't match and every legitimate request gets rejected.
- **`hexdigest()`** — Produces the hex string of the hash. GitHub sends its signature as hex.
- **`f"sha256={computed}"`** — GitHub's header includes the `sha256=` prefix. Include it in your comparison string.

**Where does this go?** Create `app/security/verify.py` and put the verification function there. Also create `app/security/__init__.py` (can be empty). Your handler in `app/main.py` imports this function.

**Why its own file, and why under `security/`?** Look at the function signature: it takes bytes, a string, and a string, and returns a bool. It has zero knowledge of FastAPI, zero knowledge of your database, zero knowledge of HTTP. It's pure cryptographic logic. That means:

1. **You can test it in isolation.** Write a test that calls `verify_signature()` with known inputs and checks the output. No HTTP server, no database, no mocking needed. Just call the function.
2. **You can audit it in isolation.** If someone (or you, in six months) needs to verify that your security is correct, they open `app/security/verify.py` and read 15 lines. They don't need to understand your database schema or your route handlers.
3. **It doesn't change when other things change.** If you add a new route, change your database schema, or reorganise your config — this file stays exactly the same. It only changes if the *verification logic* changes.

> **CRITICAL — Never use `==` for cryptographic comparison:**
>
> Python's `==` for strings **short-circuits**: it returns `False` the instant it finds a mismatched character. This means comparing a guess against the real signature takes *less time* if the first character is wrong than if the first 10 characters match.
>
> An attacker exploits this with a **timing attack**: they send thousands of requests with slightly different guesses and measure response times. Slower responses mean more matching characters. Character by character, they reconstruct the valid signature.
>
> `hmac.compare_digest()` always takes the same amount of time regardless of where the strings differ. This is called **constant-time comparison**. It's one line that closes a real attack vector. Use it. Always.

#### 5.3 Where Verification Fits Architecturally

HMAC verification must happen **before** any processing — before JSON parsing, before database writes, before background tasks. It's a **guard clause** at the gate. If you verify after processing, you've already acted on potentially forged data.

In FastAPI, you have two options:

**Option A: Inside the handler.** Simple, explicit, easy to understand. Good for a single endpoint. The verification logic is visible right where it's used.

**Option B: As a dependency.** Create a `verify_webhook` dependency that reads the raw body and header, verifies the signature, and raises `HTTPException(403)` on failure. Cleaner if you have multiple webhook endpoints, and follows separation of concerns.

```python
from fastapi import HTTPException, Header


async def verified_payload(
    request: Request,
    x_hub_signature_256: str = Header(...),
) -> bytes:
    """
    FastAPI dependency that verifies incoming request authenticity.
    Returns the raw body only if verification passes.
    """
    body = await request.body()

    if not verify_signature(body, SECRET, x_hub_signature_256):
        raise HTTPException(status_code=403, detail="Invalid signature")

    return body
```

**Why a dependency and not middleware?** Middleware intercepts *every* request — including health checks, documentation pages, everything. Signature verification only applies to webhook routes. A dependency gives you per-route control.

> **My recommendation:** Start with Option A — verification inside the handler. It's more explicit and easier to debug. Once it works, refactor to Option B as a learning exercise. The refactoring process teaches you how dependencies work better than implementing them from scratch.

#### 5.4 Idempotency — Handling Duplicate Deliveries

GitHub retries webhook deliveries if it doesn't get a 2xx response within 10 seconds, or if your server returns an error. The *same* event can arrive multiple times. Without a guard, you'd store duplicate records.

**The solution:** GitHub sends a unique delivery ID in the `X-GitHub-Delivery` header. Before inserting, check whether an event with that ID already exists:

- If it exists → return `200 OK` without inserting a duplicate.
- If it doesn't → insert and return `200 OK`.

**Implementation choice:** You can check-then-insert (two queries), or rely on a `UNIQUE` constraint on `delivery_id` and catch the database error (one query, let the database enforce uniqueness). The second approach is generally better because it's atomic — no race condition between the check and the insert.

> **Why this matters beyond webhooks:** Idempotency is fundamental in distributed systems. Any time two systems communicate over a network, messages can be duplicated (retries), lost (timeouts), or arrive out of order. Designing your handlers to produce the same result whether called once or ten times is a core professional skill.

#### 5.5 Background Tasks — The Async Dispatch Pattern

After validating and persisting a webhook event, you might want to take action: post to Slack, trigger a build, update a dashboard. These are **side effects** that shouldn't delay the response to GitHub.

**Why not do everything in the handler?** GitHub expects a response within 10 seconds. If your Slack API call takes 3 seconds and your CI trigger takes 2 seconds, you're using 5 seconds just on side effects. Add database latency and you risk timing out.

FastAPI's `BackgroundTasks` lets you schedule functions to run *after* the response is sent:

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

**How it works under the hood:** `BackgroundTasks` collects callables into a queue. After FastAPI sends the response, it runs the queued tasks sequentially.

**Limitations you should know:**
- Tasks run **in-process** — if the server crashes, queued tasks are lost.
- Tasks are **sequential** — they don't run in parallel.
- There's **no retry** — if a task fails, it fails silently unless you add error handling.
- For production reliability, you'd eventually use a proper task queue (Celery, Dramatiq, Arq). But for this project, `BackgroundTasks` teaches the concept without the infrastructure overhead.

> **Common mistake:** Using the synchronous `requests` library inside a background task. `requests.post(...)` blocks the event loop until the call completes. Use `httpx.AsyncClient()` instead. If your entire server freezes during background task execution, this is probably why.

#### 5.6 Concurrency — Understanding the Event Loop

You're using `async/await`, so you're working with the event loop. A few subtleties to understand:

**CPU-bound vs. I/O-bound work:**
- **I/O-bound** (network calls, database queries, file reads): use `async/await`. The event loop serves other requests while waiting.
- **CPU-bound** (heavy computation, processing huge payloads): these **block** the event loop. For typical webhook payloads (a few KB), HMAC computation is negligible. For very large payloads, offload to a thread pool with `asyncio.to_thread()`.

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

**For your project specifically:** HMAC-SHA256 over a typical webhook payload takes microseconds — no need for thread offloading. But if you add HTTP calls in your handler (instead of background tasks), those *must* use an async client like `httpx`, not the synchronous `requests` library.

**The key takeaway:** The event loop is single-threaded. One blocking call delays *all* concurrent requests, not just the one being processed.

#### 5.7 The Complete Pipeline

At the end of Phase 3, your request handling should follow this sequence:

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

**Error handling at each step:**

| Step | What Can Go Wrong | Correct Response |
|------|------------------|-----------------|
| 1 | Missing signature header | `400 Bad Request` — malformed request |
| 2 | Invalid signature | `403 Forbidden` — authentication failed |
| 3 | Invalid JSON | `400 Bad Request` — body is malformed |
| 4 | Missing required fields | `422 Unprocessable Entity` — parseable but invalid |
| 5 | Duplicate delivery | `200 OK` — acknowledge but don't reprocess |
| 6 | Database error | `500 Internal Server Error` — your system failed |
| 7 | Background task scheduling fails | Should not fail (just queuing a callable) |

Every step can fail, and each failure has a specific, appropriate status code. **Design your error handling to be as precise as your success path.** A `500` when you should return a `403` is both a bug and a security information leak.

#### 5.8 Configuration and Secrets Management

Your application now has multiple config values: database URL, webhook secret, downstream service endpoints. Hardcoding any of these means different environments (dev, staging, production) can't use different values.

Use `pydantic-settings` to create a typed settings class that reads from `.env`:

```python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    database_url: str
    webhook_secret: str
    log_level: str = "INFO"

    model_config = {"env_file": ".env"}


# Instantiate once at startup
settings = Settings()

# Now use: settings.database_url, settings.webhook_secret, etc.
```

**Where does this go?** Create `app/config.py` and put the `Settings` class there. Import `settings` wherever you need config values.

**Why `app/config.py` — why not put settings inside `db/` or `security/`?** Because config is used by *everything*. Your database module needs the database URL. Your security module needs the webhook secret. Your main app might need the log level. If you put it inside `db/`, then your security code has to import from `db/` to get its own secrets — that's a weird dependency (security shouldn't depend on the database module). A top-level `config.py` sits at the root of `app/` because configuration is a *cross-cutting concern* — it serves all modules equally, so it shouldn't live inside any one of them.

**Why this beats `os.getenv()`:**
- **Fails at startup.** If `WEBHOOK_SECRET` is missing from `.env`, the app crashes immediately with a clear error — not 3 hours later when the first webhook arrives and `os.getenv("WEBHOOK_SECRET")` returns `None`.
- **Type coercion.** Converts `"8000"` to `int`, `"true"` to `bool`, etc.
- **Documentation.** The class itself documents every config value your app needs.
- **Single source of truth.** No `os.getenv()` calls scattered across 12 files with inconsistent defaults.

---

### Practical Workflow — Phase 3

**Build it step by step:**

1. **Set up a webhook secret.** In GitHub's webhook settings, add a secret string. Add the same string to your `.env` as `WEBHOOK_SECRET=your-secret-here`.

2. **Create the config module.** Create `app/config.py` with the `Settings` class. Have it read `DATABASE_URL` and `WEBHOOK_SECRET` from `.env`.

3. **Create the verification function.** Create `app/security/__init__.py` (empty) and `app/security/verify.py`. Write your `verify_signature` function there (based on the pattern in section 5.2, but adapted for your actual code).

4. **Add verification to your handler.** In `app/main.py`, import your verification function and your settings. At the top of your handler, read the raw body and signature header, call your verification function, and return `403` if it fails. **Test this** — send a request with a bad signature and confirm you get a 403.

5. **Add the idempotency guard.** Before inserting, check if a record with the same `delivery_id` already exists. If it does, return 200 without inserting. (Or use the UNIQUE constraint approach from section 5.4.)

6. **Add a background task.** Pick something simple — log a message, write to a file, whatever. The point is to practice the `BackgroundTasks` pattern. You can replace it with a real notification (Slack webhook, email, etc.) later.

7. **Test the complete pipeline:** Send a forged request (wrong signature) → should get 403. Send a real GitHub webhook → should get 200 and see the event in the database. Send the same delivery ID again → should get 200 without a duplicate record.

---

### 🔒 Checkpoint — Phase 3

Stop here. Answer these before considering the project complete:

1. **Security Reasoning:** An attacker discovers your webhook URL. They craft a valid-looking JSON payload and POST it. Walk through your verification pipeline — at which step is the forgery detected? What does the attacker need (and not have) to bypass it? Could they extract it through a timing attack if you used `==` instead of `hmac.compare_digest()`?

   *The attacker has the URL, can construct any payload, and can set any headers. What's the one thing they can't produce?*

2. **Concurrency Trade-off:** You have a background task that makes an HTTP POST to a slow external service (3-second response time). Two webhooks arrive 100ms apart. Do the background tasks run concurrently? What if the first one fails? How would this differ with a dedicated task queue like Celery?

   *Remember: `BackgroundTasks` runs tasks sequentially after the response. Think about what "after the response" means when two responses are sent 100ms apart.*

3. **Architectural Integrity:** Your handler reads the raw body for HMAC verification, then calls `request.json()`. Under the hood, does `request.json()` re-read from the network, or use a cached copy? Why does this matter for HMAC correctness? What would break if you parsed JSON first, then tried to get the raw body?

   *HTTP request bodies are streams — once read, are they available again? What does Starlette do to handle this?*

---

## 6. Next Steps — Presenting This Project

### The Project Is the Portfolio

Once all three phases are complete, you have a non-trivial backend system that demonstrates:

- **API design** — a purpose-built endpoint with clear contracts
- **Database engineering** — schema design, ORM usage, migration management
- **Security engineering** — cryptographic verification, secrets management
- **Async systems design** — non-blocking I/O, background task dispatch
- **Operational readiness** — structured logging, configuration management, error handling

Each of these maps to something hiring managers ask about in backend engineering interviews. This project gives you *concrete answers* backed by code you wrote and decisions you made.

### How to Present It on a Resume

**Project title:** GitHub Webhook Telemetry Hub (or similar — name it something that sounds like a product, not a homework assignment)

**Description (1-2 lines):**
> Engineered an event-driven webhook ingestion service using FastAPI and PostgreSQL. Implemented HMAC-SHA256 payload verification, async background task routing, and schema-versioned persistence for real-time GitHub event telemetry.

**Key bullets (pick 3-4):**
- Designed and implemented HMAC-SHA256 signature verification for webhook payload authentication
- Built async PostgreSQL persistence layer using SQLAlchemy with Alembic schema migrations
- Architected background task dispatch pipeline for downstream event forwarding with idempotency guards
- Structured application using layered architecture with dependency injection for testability

### How to Talk About It in Interviews

The most impressive thing you can do in a technical interview is **explain the *why* behind your decisions**. Anyone can say "I used PostgreSQL." What separates you:

- *"I chose PostgreSQL over file storage because webhook deliveries arrive concurrently, and I needed ACID guarantees to prevent data corruption from race conditions."*
- *"I used HMAC-SHA256 verification with constant-time comparison because the endpoint is publicly accessible, and I needed to prevent both payload forgery and timing attacks."*
- *"I used FastAPI's dependency injection to provide database sessions because it decouples resource lifecycle from business logic and makes the handlers unit-testable."*

Each of these shows understanding, not just usage.

### Growth Vectors — Where to Go Next

Once the core is solid, each of these is a portfolio-worthy extension:

| Extension | What It Proves | Difficulty |
|-----------|---------------|------------|
| **REST API to query stored events** (GET endpoints with filtering, pagination) | Full-stack API design | ⭐⭐ |
| **Dockerize the stack** (FastAPI + Postgres + tunnel in Docker Compose) | Containerisation, infrastructure-as-code | ⭐⭐ |
| **Rate limiting** (per-source, per-event-type) | Defensive programming, abuse prevention | ⭐⭐⭐ |
| **Integration tests** (using `httpx.AsyncClient` + test database) | Test engineering, CI/CD readiness | ⭐⭐⭐ |
| **Deploy to a cloud provider** (Railway, Fly.io, or a VPS with Caddy) | Production deployment, DNS, TLS | ⭐⭐⭐ |
| **Real-time dashboard** (WebSocket-fed frontend showing live events) | Full-stack capability, real-time systems | ⭐⭐⭐⭐ |

---

## Appendix A — How to Get Unstuck

Getting stuck is normal. Getting stuck and not knowing how to get *un*stuck is what derails learning. Here's a systematic approach.

### The Debugging Ladder

Work through these in order. Most problems are solved by step 2.

**Step 1: Read the error message — the whole thing.**

Python tracebacks are read **bottom-up**: the last line tells you what went wrong, the lines above show the call stack (how you got there). Read the actual exception type and message before doing anything else.

Common patterns:
- `ModuleNotFoundError` → dependency not installed, or wrong virtual environment active
- `AttributeError: 'coroutine' object has no attribute X` → you forgot `await`
- `sqlalchemy.exc.OperationalError` → database connection problem (is Postgres running? Is the URL correct?)
- `422 Unprocessable Entity` → FastAPI received a request that doesn't match the expected schema

**Step 2: Isolate the problem.**

Don't debug your entire system at once. If the webhook isn't being stored:
- Can you manually insert a record using a Python script (no FastAPI)?
- Can your endpoint receive a request at all (test with curl)?
- Is the database reachable (can you connect with psql or a GUI)?

Each "yes" eliminates an entire layer from suspicion.

**Step 3: Add logging, not print statements.**

Before changing code, add logging around the suspicious area. Log the actual values of variables. The number of bugs that evaporate when you log what a variable *actually* contains is staggering.

**Step 4: Read the documentation.**

Not a blog post. Not a StackOverflow answer. The **official documentation** for the specific library and version you're using. FastAPI, SQLAlchemy, and Alembic all have excellent docs. Read the section relevant to your problem.

**Step 5: Reduce to a minimal reproduction.**

Create the smallest possible program that shows the same problem. The process of simplification usually reveals the bug — you remove a piece and the problem disappears.

**Step 6: Ask for help — with context.**

When you ask (mentor, colleague, AI), provide:
1. What you expected to happen
2. What actually happened (including the full error message)
3. What you've already tried
4. The relevant code (not your entire codebase)

---

## Appendix B — Glossary of Terms

Quick reference for terminology used in this syllabus. These definitions are scoped to this project's context.

| Term | Definition |
|------|-----------|
| **ASGI** | Asynchronous Server Gateway Interface — the protocol connecting your Python web app to a server like Uvicorn. The async successor to WSGI. |
| **Coroutine** | A function declared with `async def` that can be suspended and resumed. Doesn't run until you `await` it or schedule it on an event loop. |
| **CRUD** | Create, Read, Update, Delete — the four basic database operations. Most REST APIs are CRUD APIs. |
| **Dependency Injection** | A pattern where a function receives its dependencies from the outside (via parameters) instead of creating them itself. Improves testability and modularity. |
| **Event Loop** | The core of async Python. A single-threaded loop that manages coroutine execution. When one coroutine awaits I/O, the loop runs another. |
| **HMAC** | Hash-based Message Authentication Code — a cryptographic mechanism for verifying message authenticity and integrity using a shared secret. |
| **Idempotency** | The property that doing an operation multiple times produces the same result as doing it once. Critical for reliability in distributed systems. |
| **ORM** | Object-Relational Mapper — maps database tables to classes and rows to objects, abstracting raw SQL into method calls. |
| **Race Condition** | A bug that depends on the timing of events (e.g., two processes writing to the same file at the same time). |
| **Reverse Tunnel** | A tunnel initiated outward from the private network, allowing external traffic to reach internal services without opening inbound ports. |
| **Session (SQLAlchemy)** | A staging area that tracks changes to ORM objects. Changes accumulate in memory and flush to the database as a single transaction on commit. |
| **WAL** | Write-Ahead Log — a database technique where changes are logged before being applied to main data files, enabling crash recovery. |
| **Webhook** | An HTTP callback — a server-to-server POST request triggered by an event. The receiver doesn't poll; it waits for the notification to arrive. |

---

> **This is a living document.** As you work through each phase, come back and add your own notes — gotchas you hit, solutions you found, things that clicked. The best engineering documentation evolves with the engineer.
