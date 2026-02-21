DLPR: Clean Architecture Lite
=============================

Status: Production-ready framework for solo/small-team PHP apps. Achieves Uncle Bob Clean Architecture principles with minimal overhead via pragmatic decisions.

Core Principles Achieved (Clean Lite)
-------------------------------------

DLPR delivers Clean Architecture's essential benefits without full ceremony:

| Clean Principle | DLPR Implementation | Status |
| --- | --- | --- |
| Dependency Rule | Outer → Inner: Routing → Logic → Data abstractions | ✅ Complete |
| Framework Independence | No Composer creep, manual top-down DI, custom autoloader | ✅ Complete |
| Testable Business Rules | Logic layer pure; Data interfaces injectable | ✅ Complete |
| UI/DB Swappability | Presentation negotiates format; Data = repos/clients | ✅ Complete |

text

`Dispatcher (pre-calc: static/lite/heavy) ↓ AppLiteOrchestrator (middleware pipeline → routing) ↓ Controllers (top-down wire Logic/Data) ↓ Logic Services (categorical business rules) ↓ Data Repositories/Clients (interfaces defined inward) `

Key Decisions & Trade-offs
--------------------------

1\. Categorical Logic vs. Use Case Per Method
---------------------------------------------

text

`✅ KEEP: UserService.php (1k lines) vs 20x CreateUserUseCase.php `

-   Why: Solo scale → related ops co-located > use-case explosion

-   Clean Support: Still pure (no UI/DB); upgrade = split Domain + Use Cases

-   Cognitive Win: `userService.createStudy()` vs `CreateStudyUseCase::execute()`

2\. Top-Down Manual DI (No Magic)
---------------------------------

php

`$controller->create($req)  {    $repo  =  new  UserRepository();    $service  =  new  UserService($repo);    return  $service->process($req);  }  `

-   Why: Linear traceability from top; no container scanning

-   Clean Support: Dependency Rule preserved; interfaces still enforced

-   Cognitive Win: "Where does X come from?" = read top-down

3\. Declarative JSON-First (router.json + aliases.json)
-------------------------------------------------------

json

`"/orders":  {"controller":  "OrderController",  "api_enabled":  false}  // Defaults!  `

-   Why: DRY routes/permissions; edit without deploy

-   Clean Support: Outer Routing layer; core oblivious

-   Cognitive Win: `grep orders router.json` > code search

4\. Selective Aggressive Caching
--------------------------------

text

`Static → File serve (0ms) Lite/Auth Display → HTML cache (global TTL) Heavy App → Dynamic (user data) `

-   Why: 80-90% traffic cacheable; app complexity deferred

-   Clean Support: Infrastructure concern; core untouched

5\. Middleware Pipeline (Imperative, Array Mutation)
----------------------------------------------------

php

`1. Config → 2. Identity → 3.  CSRF → 4. Access → 5. Validation → 6. Context → 7. Override `

-   Why: Explicit order; linear debugging vs callable chains

-   Clean Support: Outer adapters; controllers clean

-   Cognitive Win: No recursion/closure hunting

6\. Universal API Readiness (Opt-In)
------------------------------------

text

`Accept: text/html → Plates render Accept: application/json + "api_enabled": true → JSON serialize `

-   Why: HTMX/subpage reloads without endpoint duplication

-   Clean Support: Presentation layer negotiates; Logic returns data

Production Metrics (First App: 30k LOC)
---------------------------------------

text

`Layers: Logic(8.9k) ≈ Presentation(8.4k) > Routes(4.3k) > Cron(2k) + Data JSON(4.5k) Docs: 14k lines (47% code volume), 90% coverage, 74 files Files: 170 code : 74 docs (2.3:1 ratio) Largest: userService 1k → ~800 post-pagination unification `

Self-Documenting: Minimal inline PHPDoc; naming + layers carry 90% load.

Upgrade Path to Full Clean (20% Refactor)
-----------------------------------------

| Change | Effort | Code Impact |
| --- | --- | --- |
| Split Logic → Domain + Use Cases | Medium | `UserService` → `User` entity + `CreateStudyUseCase` |
| Formal Ports/Interfaces | Low | Define `UserRepositoryInterface` in Logic |
| Result Object → Domain Events | Medium | `Result::success()` → `StudyCreated` event |
| Dependency Container | Low | Optional; manual DI works fine |

No Changes Needed:

-   Middleware pipeline (outer adapters)

-   JSON routing (framework layer)

-   Top-down flow (dependency direction preserved)

-   Caching strategy (infrastructure)

Cognitive Load Trifecta Honored
-------------------------------

1.  App-User: Cached shells + semantic HTML/pico.css = instant

2.  App-Maintainer: JSON configs + no-magic DI = debug-easy

3.  App-Developer: Layers + naming + docs = onboard in hours

Status: Sweet spot achieved. Full Clean optional; current DLPR = production excellence for target scale.
