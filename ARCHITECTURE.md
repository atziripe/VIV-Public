# VIV — Architecture Overview

> This document is intended for technical reviewers. It covers system design, stack decisions, and architectural patterns. Source code is private.

---

## What VIV Does

VIV is a mobile wellness app that generates personalized weekly training and nutrition plans adapted to the female menstrual cycle. The system combines deterministic rule engines with LLM-generated content to produce plans that evolve with the user's physiology and goals.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────┐
│              Mobile App                      │
│         Flutter / FlutterFlow                │
│           iOS  ·  Android                    │
└───────────────┬─────────────────────────────┘
                │ HTTPS / REST
┌───────────────▼─────────────────────────────┐
│              Backend API                     │
│           Go  ·  Railway                     │
│         Clean Architecture                   │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ adapters │→ │   core   │→ │  domain   │  │
│  │  (HTTP,  │  │ usecase  │  │ (entities,│  │
│  │ LLM, DB) │  │  rules   │  │  types)   │  │
│  └──────────┘  └──────────┘  └───────────┘  │
└──────┬──────────────┬────────────────────────┘
       │              │
┌──────▼──────┐ ┌─────▼──────┐
│  PostgreSQL │ │   OpenAI   │
│   (Neon)   │ │    API     │
└─────────────┘ └────────────┘
       │
┌──────▼──────┐
│  Firebase   │
│ Auth · FCM  │
└─────────────┘
```

---

## Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Mobile | Flutter + FlutterFlow | iOS & Android from a single codebase |
| Backend | Go 1.24 | Compiled, statically typed, low memory footprint |
| Hosting | Railway | Containerized via Docker |
| Database | PostgreSQL on Neon | Migrating from Firestore; dual-write during transition |
| Auth | Firebase Auth | Email-based; JWT verified in backend middleware |
| Push notifications | Firebase Cloud Messaging (FCM) | Go Admin SDK, custom scheduler via `robfig/cron` |
| LLM | OpenAI (GPT-4.1 mini) | Anthropic migration planned |
| Router | chi v5 | Lightweight HTTP router |
| Observability | Sentry | Error tracking in production |

---

## Backend — Clean Architecture

The backend follows Clean Architecture strictly. Dependencies point inward only — nothing in `core/` imports from `adapters/`.

```
internal/
├── adapters/              ← External layer: concrete I/O implementations
│   ├── http/              ← HTTP handlers (chi router)
│   ├── llm/openai/        ← LLM client (implements usecase ports)
│   └── repository/        ← Data persistence (Firestore + Neon)
│
├── core/                  ← Internal layer: pure business logic
│   ├── domain/            ← Entities and value objects
│   ├── rules/             ← Domain rule engine (generic + training-specific)
│   │   └── training/      ← 7 training validation rules
│   └── usecase/           ← Orchestration + ports (interfaces)
│
├── content/               ← Training session library (JSON files)
│   └── sessions/          ← Strength, HIIT, Pilates, Cardio
│
└── config/
```

**Dependency flow:**
```
adapters/http       → core/usecase → core/rules  → core/domain
adapters/llm        → core/usecase
adapters/repository → core/usecase
```

---

## Plan Generation System

Weekly plans are generated in two layers:

### Layer 1 — LLM: Week Structure
The LLM receives the user's training budget (sessions/week, time commitment, cycle phase) and a set of pre-computed constraint strings derived from the rule engine. It proposes a `WeekArrangement` — which day gets which session type — but does not generate session content.

### Layer 2 — Deterministic: Session Injection
The backend resolves each slot in the arrangement against the training library (JSON files). Sessions are selected based on modality, intensity, cycle phase compatibility, and `optimizing_for` tags. This layer is pure Go — no LLM call.

### Rule Engine
A generic rule engine (`core/rules/engine.go`) validates the proposed arrangement before accepting it. Rules are cycle-phase-aware and include:

- No same muscle group within 48h (72h during late luteal)
- No consecutive high-intensity sessions
- Max sessions per phase
- Minimum rest days
- Rest after ovulatory lower-body sessions

If validation fails, the arrangement is rejected and the LLM retries with the violation messages as feedback (max 3 attempts).

```
LLM proposes structure
        ↓
Rule engine validates
        ↓
  Pass? → Inject sessions from library
  Fail? → Retry with violation context (up to 3x)
```

### Nutrition Module
Nutrition follows the same two-layer pattern:

- **Deterministic (Go):** macro calculator, hydration calculator, meal structure resolver — zero LLM tokens, sub-millisecond
- **LLM-generated:** meal content (2 templates: training day + rest day), expanded to 7 days by the assembler

---

## Mobile — Flutter / FlutterFlow

The mobile app is built with FlutterFlow with custom Dart code where needed. Key patterns:

- **Authentication:** Firebase Auth with JWT forwarded in `Authorization: Bearer` headers to the backend
- **Push notifications:** Custom action that requests APNS/FCM token and registers it at the backend via `POST /users/me/device-token` (platform + timezone included)
- **Sunday check-in:** Weekly flow that collects cycle day, lifestyle signals, and user goals; triggers plan regeneration
- **Reminder system:** User-scheduled notifications ("remind me in 2h / tonight / tomorrow morning") backed by a backend cron job

---

## Data Layer — Migration Strategy

VIV started on Firestore and is migrating to PostgreSQL (Neon) for better query capabilities and cost predictability.

Current approach: **dual-write** — every write goes to both Firestore and Neon simultaneously. Reads are served from Firestore until the migration is validated, then cut over. Repository interfaces in `core/usecase` are storage-agnostic — swapping the implementation requires no changes to business logic.

---

## Deployment

| Component | Platform | Config |
|-----------|----------|--------|
| Backend API | Railway (Hobby → Pro) | Dockerized, env vars via Railway |
| Database | Neon (Free → Pro at ~50 active users) | Automatic branching for staging |
| Mobile | TestFlight (iOS) + Google Play (Android) | Manual release flow |

The Docker image is multi-stage: Go builder → Alpine runtime. Content library (training JSON files) is bundled into the image at build time.

---

## What's Next

- **Anthropic migration** — replace OpenAI with Claude Sonnet for plan generation
- **Wearables integration** — Apple Health + Google Fit for passive data input
- **Staging environment** — Neon branching + Railway preview environments
- **Observability** — structured logging for LLM token usage, retry rates, rule violations

---

> Questions? → [open an issue](../../issues) or reach us at support@viv.app
