# DLPR Framework

**Data · Logic · Presentation · Routes**

A principled PHP architecture for solo developers and small teams who want clean separation of concerns without the overhead of enterprise frameworks.

DLPR is extracted from production use in [memorize.live](https://memorize.live) and documented here as a standalone framework reference.

---

## The Core Idea

Most PHP projects collapse under their own weight because logic, data access, and presentation bleed into each other. DLPR enforces four strict layers with one simple rule: **each layer talks only to the layer below it.**

```
Routes (Controllers)
    └── Logic (Services, Middleware)
            └── Data (Repositories, Clients)

Presentation (Templates)
    └── receives data from Routes, renders only
```

Controllers never query the database. Services never touch HTTP. Repositories never contain business rules. Templates never fetch data. When everyone stays in their lane, the codebase stays readable — six months later, by someone who wasn't there.

---

## The Four Layers

### Data (`App/Data/`)
All persistence and external API communication lives here.
- **Repositories** — SQL queries only. Return data objects or arrays. Nothing else.
- **Clients** — Wrappers for external APIs (mail, SMS, etc.).

### Logic (`App/Logic/`)
Business rules and cross-cutting concerns.
- **Services** — Orchestrate business rules. Coordinate repositories and clients. Own the "what should happen" decisions.
- **Middleware** — Auth, authorization, validation, CSRF. Processes the request before it reaches the controller.
- **Helpers** — Stateless utility functions.

### Presentation (`App/Presentation/`)
Renders the UI. Receives data. Does nothing else.
- **Templates** — PHP template files.
- **Presenter** — Orchestrates template rendering.

### Routes (`App/Routes/`)
Entry point for application logic.
- **Controllers** — Receive pre-processed requests, call services, return data to Presentation. Thin by design.

---

## Request Lifecycle

```
index.php (single entry point)
    → App.php (orchestrates)
    → Middleware Stack (auth, validation, etc.)
    → Router (URI → Controller + Method)
    → Controller → Service → Repository/Client
    → Presenter → Template → Response
```

---

## The Hard Rules

These are what make DLPR work. Violate them and you're back to spaghetti.

- **Controllers** — no business logic, no DB queries, no auth checks
- **Services** — no HTTP request/response handling
- **Repositories** — no business rules
- **Templates** — no data fetching, no complex logic

---

## What's in This Repo

```
architecture/     Core pattern docs — DLPR, routing, middleware, request lifecycle
guides/           How to build within the pattern (controllers, views, APIs, etc.)
middleware/       The middleware stack, layer by layer
standards/        Naming conventions, response patterns, explicit dependencies
cron/             How scheduled tasks fit the pattern
ui/               Interface level concepts
```

---

## Status

**Alpha 1** — February 2026

The documentation is the deliverable at this stage. The pattern is extracted from a production codebase ([memorize.live](https://memorize.live)), the docs are clean and framework-generic, and the repo is public.

What's planned: 1) a php code git-submodule impelementation 2) a minimalist sample app 3) a full-featured reference implementation.

More at [thepragmat.com](https://thepragmat.com).

---

## Philosophy

DLPR isn't trying to be Laravel. It's trying to be the thing you reach for when you want to build something solid without pulling in a framework that knows better than you do.

The principles here are stewardship principles wearing a tech costume: clear ownership, no hidden dependencies, every piece doing exactly one job. Simple enough to explain to a junior dev in an afternoon. Strict enough to keep a codebase honest for years.
