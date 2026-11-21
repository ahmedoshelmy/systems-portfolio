# Interview guide

How I talk about each app: a 60-second pitch, the deep-dive points (with the real bugs and trade-offs met while
building them), questions I expect with short answers, and what I would do next. File links for every pattern are
in [patterns.md](patterns.md).

A theme that runs through all of them, worth saying once early: **at-least-once delivery, exactly-once effect.**
Brokers and HTTP clients redeliver; every app makes the second delivery harmless with a natural key, an inbox, a
unique index or a job id, rather than hoping it never happens.

---

## Qirsh (.NET microservices, RabbitMQ, gRPC, SignalR)

**60-second pitch.** Egyptian banks and wallets announce every payment by SMS, in English or Arabic and a dozen
formats. Qirsh parses them into a ledger, learns categories from your corrections, tracks monthly budgets and
pushes a live alert at 80% and 100%. It is three services (Ingestion, Ledger, Budgets) with a schema each in one
Postgres, talking through RabbitMQ (or Azure Service Bus with one setting) with the MassTransit transactional
outbox and inbox. The one synchronous question, "how much has this category already spent this month?", is a gRPC
call with a deadline. A Testcontainers test drives an SMS all the way to a SignalR alert.

**Deep-dive points.**

1. **The outbox bug only a real test could find.** MassTransit caches the outbox lock SQL per entity type in a
   static map. With schema-per-service and three services in one test process, the first service's schema won
   and the others polled the wrong table, so messages sat in the outbox. Unit tests could never see it; the
   Testcontainers end-to-end test did. Fix: `UsePostgres(enableSchemaCaching: false)`, documented in ADR 2.
2. **RabbitMQ would not boot under rootless Podman.** The compose health check ran `rabbitmq-diagnostics`. Run
   as root before the broker was up, the CLI created a root-owned Erlang cookie that the broker then could not
   read, so the broker never started. The health check now probes the management listener over HTTP, which only
   opens once the broker is up. Lesson: a health check is code that runs in your container and can break it.
3. **Tracing noise.** The outbox delivery service polls Postgres twice a second per service, and every poll
   became its own root trace in the Aspire dashboard, burying real requests. A parent-based sampler keeps every
   span inside a request or message trace and drops only parentless client spans.
4. **gRPC for questions, events for facts.** Spending flows as events. A budget created mid-month back-fills with
   one deadline-bound gRPC call; the reply lists the counted transaction ids so events already in flight for them
   are not counted twice.
5. **A pure, table-tested parser.** The bilingual SMS grammar is a dependency-free domain function tested with
   dozens of invented messages, so it can run behind REST, gRPC or on a phone unchanged.
6. **Measured, diagnosed, fixed, re-measured.** A k6 ramp in
   [observability-lab](https://github.com/ahmedoshelmy/observability-lab) (2→40 req/s from a cold start) left up
   to 1,990 messages stuck in the outbox, delivery p95 over 10 s, and messages dead-lettered. Logs showed
   Postgres `40001` serialization failures. Two causes together: MassTransit's EF outbox defaults to
   `RepeatableRead`, and every SMS bumped a few hot merchant-rule counters with read-modify-write. Fix: outbox at
   `ReadCommitted`, the counter as one atomic `UPDATE … + 1` (lowering isolation alone would have turned errors
   into lost updates), and a short jittered retry for lost races (40001, 40P01, 23505) nested inside the normal
   retry. A Testcontainers test firing 200 SMS at once failed before (196/200, 4 dead-lettered) and passes after.
   Re-measured: backlog peak 3–8, zero `40001`, zero dead letters, delivery p95 71–243 ms. ADR 5.

**Likely questions.**

- *Why not one database?* It is one server, but a schema per service: each service owns its tables, migrations
  and outbox, and can move to its own server by changing a connection string.
- *What if the same SMS is pasted twice?* Ingestion hashes the raw text; Ledger keys on `(account, hash)`; Budgets
  records applied transaction ids. Three natural keys plus the inbox.
- *Why gRPC and not REST between services?* A typed contract, deadlines and cancellation built in, and HTTP/2;
  and it is only for one query, so the services stay decoupled for everything else.
- *How would you deploy it?* Azure Container Apps with Service Bus; the Bicep exists and compiles, but the
  internal gRPC port needs an extra port mapping and it has never been deployed.

**What I'd do next.** Authentication and multi-user ledgers; an Android companion that reads SMS and calls the
gRPC intake; actually deploying the Bicep; an optional free-model categoriser for merchants no rule knows.

---

## Job Radar (.NET modular monolith + worker, RabbitMQ)

**60-second pitch.** A self-hosted radar over the public job APIs companies already publish (Greenhouse, Lever,
Ashby, Workable) plus two remote-job feeds. A worker polls politely, the Catalog removes duplicates across feeds,
Matching scores every opening with reasons you can read ("+12 .NET in the title"), and matches stream to the
browser over SignalR. A Tracker module runs your applications through a state machine, with a spaced-repetition
interview-question bank. Two deployables: a worker and a modular monolith of three bounded contexts.

**Deep-dive points.**

1. **Modular monolith first.** Catalog, Matching and Tracker each have their own project, `DbContext`, schema
   and migrations, never share tables, and an architecture test fails the build if one references another.
   Splitting one out later is a deployment change, not a rewrite.
2. **Fan-out that matters.** `JobPostingAdded` has two subscribers, Matching and a stats projection, each on its
   own queue. `JobMatched` goes to a temporary per-instance queue so every API replica hears every match and
   pushes to its own SignalR clients, with no backplane needed at this size.
3. **Being a good API citizen.** One resilience pipeline per host: timeout, retry with jitter that honours
   `Retry-After`, circuit breaker, token-bucket rate limiter. Conditional GETs (`If-None-Match`) turn unchanged
   feeds into cheap 304s; intervals under an hour fail validation at startup.
4. **Dedupe is a domain rule.** Exact URLs fail because of aggregator pages and tracking parameters, so postings
   are fingerprinted on normalised company, title and place, with a canonical URL as fallback. Each
   normalisation has a test with the example that motivated it.
5. **Publishing safely.** Before the repo went public, a secret scanner flagged synthetic GUIDs in the fixtures
   as card-number-like digit runs; I replaced them and rewrote the history so the public repo never contained
   them. Also a reminder that test data design is part of the job.

**Likely questions.**

- *Why not microservices?* Three contexts with low load and one team: the costs (network failures, distributed
  transactions, deployment overhead) buy nothing yet. The boundaries are enforced so the option stays open.
- *How do you avoid hammering a feed that is down?* The breaker opens per host, failing feeds back off
  exponentially, and the run state survives restarts.
- *Can the AI explanation change the score?* No. Scoring is a pure function; an optional free model only writes a
  two-sentence summary, off by default, and any error just hides it.

**What I'd do next.** Authentication; more feed adapters; a SignalR backplane once replicas grow; deploying the
Bicep to Container Apps.

---

## Listmate (.NET, SignalR, offline-first PWA)

**60-second pitch.** Shared lists that update live on every phone and keep working offline: tick items in a
supermarket basement and they sync later without overwriting what someone else changed. The client queues
mutations in IndexedDB with idempotency keys; the server applies each batch exactly once and resolves conflicts
per field. It is deliberately one service with no broker, and it still scales out: two replicas behind
round-robin nginx, with Redis for the SignalR backplane and a shared Data Protection key ring.

**Deep-dive points.**

1. **Offline sync without a CRDT.** Intent mutations, client-chosen ids, a `processed_mutations` row written in
   the same transaction as each change, per-field last-writer-wins and delete-wins. Simpler to reason about than
   Automerge or Yjs for lists, and written down in ADR 3.
2. **Racing replays.** The integration test fires the same offline batch twice in parallel, then a third time.
   Every change bumps the list version, so racing batches collide on Postgres' `xmin`; the loser re-reads and
   retries a bounded number of times, finds its mutation ids processed, and does nothing.
3. **Never trust the socket.** Pushes are hints. Each list has a gap-free version; after any reconnect the client
   asks `changes?since=N` and gets everything it missed, tombstones included.
4. **Scale-out has two halves.** The SignalR Redis backplane carries group messages between replicas, but a bearer
   token issued by replica B was not readable by replica A until the Data Protection keys were shared in Redis
   too. A scale smoke test signs in on B, calls A with that token and writes through B while listening on A.
5. **Revoking access in real time.** Removing a member is a domain rule, but the hard part is the open socket:
   the server tells the group, then evicts that user's connections from the SignalR group, and every REST and sync
   call re-checks membership anyway.

**Likely questions.**

- *What happens if a device is offline longer than you keep mutation ids?* A retention job prunes old ids; a
  replay after the window could apply twice. That trade-off is documented in the sync protocol.
- *Why last-writer-wins and not a merge?* For list items, fields are independent and short; per-field LWW keeps a
  tick and a rename both. Concurrent renames of the same item keep the last, which users accept for groceries.
- *Why no broker?* One bounded context and one transaction per batch: a broker would add eventual consistency to
  explain without buying anything.

**What I'd do next.** Presence in Redis (it is per replica today), cross-replica eviction, a Flutter client,
recurring lists.

---

## Price Watch EG (NestJS, RabbitMQ, gRPC sidecar, socket.io)

**60-second pitch.** Watch a product across Egyptian online shops, keep its price history, and get a live toast
when the lowest in-stock offer drops under your target or hits a 30-day low. Three NestJS processes: an API with a
transactional outbox, a checker that consumes check requests, and a gRPC scraper sidecar that wraps an existing
open-source scraping library and streams one result per shop. One OpenTelemetry trace spans HTTP, AMQP and gRPC.

**Deep-dive points.**

1. **Schedulers that scale out with leases.** Instead of electing one scheduler, any checker asks the API for due
   watches and the API hands them out in one statement: `UPDATE ... FOR UPDATE SKIP LOCKED RETURNING` with a
   `check_requested_at` stamp. Two checkers asking at once get disjoint batches (an integration test fires two
   claims in parallel), and a lost check is retried when the lease runs out.
2. **Competing consumers vs fan-out.** Checks and observations use shared durable queues with prefetch as
   back-pressure; `PriceDropped` goes to an exclusive queue per API replica so every replica's sockets hear it.
3. **Scraping is unreliable by nature.** Per-shop time boxes, a search deadline, a concurrency cap and a short
   cache. Every shop's outcome is stored; a check counts only if at least two shops answered and at most half
   failed, so a broken shop never triggers a fake drop.
4. **Exactly-once alerts.** Observations are unique per check id and alerts per `(watch, Cairo day, price)`, so a
   redelivery, a duplicate check or a second replica cannot alert twice.
5. **Reproducible builds.** The scraper image is built from a pinned public commit of the library, passed as a
   build argument; an earlier pin pointed at a commit that only existed locally and would have broken anyone
   else's build.

**Likely questions.**

- *Why a sidecar and not port the scraper?* Reuse a tested library, scale and fail it separately, and stream
  partial results so a deadline still yields data. The proto is the contract.
- *What if RabbitMQ goes down?* The process exits and the orchestrator restarts it; the outbox keeps every
  unpublished message and the queues are durable.
- *How does a trace survive the outbox?* The outbox row stores the trace headers, and the relay restores them
  when it publishes seconds later.

**What I'd do next.** A per-shop circuit breaker with backoff state, an outbox retention job, authentication, and
a notifier consumer (email or messaging app).

---

## Trip Split (Java, Spring Modulith, event sourcing, STOMP)

**60-second pitch.** Split trip costs with friends: three split rules, any currency, live balances, the fewest
transfers to settle up, and a full history with undo and a time-travel slider. Every change is an event in a
hand-rolled Postgres event store; balances are an inline projection updated in the same transaction, the timeline
an asynchronous one fed through Spring Modulith's outbox. ETags carry the stream version so a stale write gets
`412`.

**Deep-dive points.**

1. **When event sourcing pays.** Shared money needs an audit trail, undo and "what did we owe on Saturday?", and
   events give all three by construction. The costs are replays, projections and events that can only evolve
   additively or through upcasters, so event type names are versioned.
2. **An event store is a table and a unique index.** `UNIQUE (stream_id, version)` is the optimistic concurrency;
   the version doubles as the HTTP `ETag`. Postgres also lets the append and the inline projection share a
   transaction, which a separate event database cannot.
3. **Facts, not formulas.** `ExpenseAdded` stores the computed shares and the exchange rate, so fixing a rounding
   rule later cannot rewrite last week's balances on replay.
4. **Money correctness.** `long` minor units, one half-even conversion, then largest-remainder allocation with a
   join-order tie-break. jqwik properties check that shares always add up and balances net to zero.
5. **Settle-up honesty.** The greedy algorithm needs at most n-1 transfers; the true minimum is NP-hard (it hides
   subset-sum). For a holiday group greedy is optimal or one transfer longer, and I say so.

**Likely questions.**

- *Why not Axon or EventStoreDB?* For one aggregate type, a table and a unique index are enough and keep the
  projection in the same transaction. A framework would be the right call with many aggregates and teams.
- *How do you handle a schema change in an event?* Additive fields with defaults; otherwise a new versioned type
  and an upcaster on read. Never edit stored events.
- *Is the CSV export safe?* It follows RFC 4180 quoting and neutralises spreadsheet formula injection, because
  user-typed descriptions end up in someone's Excel.

**What I'd do next.** Authentication; a STOMP relay to a real broker so more than one instance can run; exchange
rates from a free API (still stored per expense); renaming members.

---

## Upkeep (NestJS, GraphQL, CQRS, BullMQ)

**60-second pitch.** Car and home upkeep with reminders that actually fire: "every 10 000 km or 6 months,
whichever comes first", meter readings, renewals. Two odometer readings give a km/day projection so distance
becomes a date. Commands and a saga replan one delayed BullMQ job per reminder whenever a due date may move; a
separate worker fires them with retries, a dead-letter queue and a sweep job as the safety net, then pushes a
GraphQL subscription and an email.

**Deep-dive points.**

1. **The core rule is a pure function.** Due = the earliest of the time side, the distance side projected through
   km/day, and any known date. A car that is not driven never becomes due by distance; month arithmetic clamps
   (31 January plus one month is the end of February).
2. **Delayed jobs vs polling.** Exact firing times, no table scans, idempotent because job id = reminder id, and the
   notification row is unique per reminder and written in the same transaction that marks it fired. Two
   competing workers fire each reminder exactly once in the integration test.
3. **Retrying the saga step.** Planning is "cancel old plan, store rows, enqueue jobs"; a Redis blip in the middle
   used to leave rows without jobs until the sweep noticed. Now the step is retried with backoff and is safe to
   repeat. The wait goes through a clock port, so the test asserts the backoff without sleeping. Honest gap: a
   crash mid-retry still loses the plan, and an outbox would close it.
4. **GraphQL hardening.** A validation rule rejects over-deep or over-costly queries before any resolver runs;
   a unit test checks that every operation the web app sends stays under the limits.
5. **A CSRF surprise.** The web client's GET queries were blocked by Apollo Server's CSRF prevention (simple GETs
   without a preflight-forcing header). Sending queries as POST fixed it; the protection stayed on.

**Likely questions.**

- *Is CQRS overkill here?* On the write side the saga earns it: handlers never touch the queue, one place reacts
  to "the due date may have moved". On the read side it is uniformity only, and I say that.
- *Why GraphQL?* The screen is a graph with a computed field that needs batched readings; DataLoader keeps it to
  one query per relation, and subscriptions scale across replicas through Redis.
- *What if a job is lost?* A repeatable sweep re-enqueues scheduled reminders that are past due and never fired.

**What I'd do next.** An outbox for the plan step, authentication and per-user ownership, persisted queries,
Bicep for Container Apps.

---

## Dose Keeper (.NET Azure Functions, Durable Functions)

**60-second pitch.** Medication schedules with escalating reminders, refill forecasts and adherence charts,
built serverless on Azure Functions with Durable Functions and run fully locally with Azurite and the official
Functions host image. Each schedule is an eternal orchestration: timer, reminder, race "answered" against an
escalation timer, mark missed and alert a caregiver, `ContinueAsNew`. It is a demo with synthetic data, not
medical advice.

**Deep-dive points.**

1. **The `CancelAsync` hang.** Unit tests were green, but in the live compose run the "answered" path silently
   stalled. `CancellationTokenSource.CancelAsync()` runs the timer's callbacks on the thread pool, resuming the
   orchestrator off its single-threaded scheduler, which Durable Functions does not allow. Synchronous `Cancel()`
   fixed it; the reason is a comment in the code and a suppressed analyzer warning with a justification.
2. **Self-healing loops.** If starting an orchestration fails, or one is terminated, that schedule would never
   remind again. A timer every 15 minutes asks the Durable client for each schedule's status and restarts only
   missing, failed or terminated loops, from now (no reminders days late). The smoke test terminates a live loop
   and watches it come back.
3. **Time is domain logic.** Wall-clock times in a zone, expanded per date with that zone's rules: spring-forward
   gaps shift forward, fall-back ambiguity takes the first occurrence so nobody is reminded twice. A late bug:
   "today" and the week start were computed in UTC, so just after local midnight the page showed yesterday's
   doses. They now use the user's zone (Cairo by default), with a test.
4. **Poison messages.** `maxDequeueCount: 3`, then a poison-queue trigger parks the message in a dead-letters
   table, exposed at `/api/deadletters` (the Azure CLI table query cannot talk to the local emulator, so the
   smoke test needed an endpoint anyway).
5. **Local parity, no keys.** Azurite plus the official isolated-worker image, a dev-storage connection string
   with a proxy URI so no account key is committed, and identity-based storage access in the Bicep.

**Likely questions.**

- *Why Durable Functions and not a cron job?* The reminder is a long-running conversation with a human:
  timers, waiting for an event, escalation, all durable across restarts and billed only while running.
- *What must orchestrator code avoid?* Non-determinism: `DateTime.Now`, I/O, random, and (as found) anything that
  resumes on another thread. Use `context.CurrentUtcDateTime` and activities.
- *Cold starts?* Flex Consumption with always-ready instances would trade a little money for latency; a reminder
  a few seconds late is acceptable here (ADR 4).

**What I'd do next.** A real notification channel (web push), a screenshot and accessibility pass, deploying
the Bicep, and authentication.

---

## Gym Scout (Java, Spring Modulith, PostGIS, Resilience4j)

**60-second pitch.** Find gyms near you or along your commute, ranked by the real price per visit at your training
frequency plus travel time, each number with a plain-language reason. Then log workouts, detect personal records
and track what each visit has actually cost. PostGIS does the geometry; live OpenStreetMap and OSRM data sit
behind ports with offline defaults and a polite caller (rate limit, bulkhead, retry, circuit breaker, fallback).

**Deep-dive points.**

1. **Let the database do geometry.** `geography` points with a GiST index, `ST_DWithin` in metres,
   `ORDER BY location <-> :point` for an index-assisted KNN scan, and a corridor around the home-work line for the
   commute search. A Testcontainers PostGIS test checks the nearest-first order.
2. **Polite to free community servers.** A 1 request/second rate limiter, a bulkhead of 2, retry with backoff on
   429 and 5xx, a circuit breaker, and a straight-line travel estimate as the fallback. WireMock tests prove the
   retry, the open breaker and the fallback; fixtures are hand-written, never recorded from the live servers.
3. **The midnight bug.** The smoke test ran just after midnight Cairo time while the container ran in UTC, so
   "today" for visits and weekly volume was still yesterday and the dated checks failed. The fix pins the API's zone in compose and
   makes the smoke test use explicit dates instead of "now".
4. **Explainable, relative scoring.** Price, travel and amenities points are relative to the best gym in the
   result set, so scores stay meaningful at 1 km or 20 km. A jqwik property checks a cheaper and closer gym never
   scores lower.
5. **Records replayed, not stored.** Personal records come from replaying sets in order, so editing history never
   leaves a stale record.

**Likely questions.**

- *Why `geography` and not `geometry`?* Distances in metres on the sphere without picking a projection; at city
  scale the speed difference does not matter.
- *How do modules talk?* Through public APIs or events in the Modulith publication registry; a logged session
  becomes a membership visit via `SessionLogged`, inserted idempotently.
- *Why not call OSRM for every gym?* Cost and politeness: answers are cached per rounded origin and destination,
  and the straight-line estimate covers failures.

**What I'd do next.** OpenTelemetry tracing (it has Micrometer metrics only), authentication, a screenshot, and
an import job that refreshes gyms from OpenStreetMap on a slow schedule.

---

## Study Forge (Java, Kafka, two services)

**60-second pitch.** Learning roadmaps and spaced-repetition cards, graded 0 to 5 and rescheduled with SM-2.
Every review and completed item is an event. The learning service writes it through Spring Modulith's outbox to
Kafka (Redpanda locally), keyed by learner; a separate insights service projects it into streaks, retention,
weekly minutes and a due forecast, pushed live over Server-Sent Events. Two services, two databases, shared only
by event contracts with JSON Schemas.

**Deep-dive points.**

1. **Kafka for history, not only delivery.** Insights are projections, so the log is the source: a new read model
   can be built by replaying from offset zero. Keying by learner keeps one learner's events in order on one
   partition. The sibling .NET apps use RabbitMQ, which fits commands and work queues better.
2. **Outbox, then idempotent consumer.** The review row and the event publication share one transaction; the
   consumer dedupes by event id in the same transaction as the projection.
3. **Non-blocking retries.** `@RetryableTopic` moves a failing record to delayed retry topics so one bad record
   does not stall its partition; unreadable messages go straight to a dead-letter topic and a table. Trade-off,
   written down: a retried event can be overtaken by a later one.
4. **A hung test should not hang CI.** A Kafka integration test once waited forever, so forked test JVMs are now
   capped at ten minutes: a stuck consumer fails the build instead of stalling it.
5. **Contracts that can only grow.** A test checks every event field is required and typed in its JSON Schema,
   examples round-trip, and unknown fields are ignored.

**Likely questions.**

- *Why two services here when the others are monoliths?* To show the other side deliberately: the read side has a
  different shape, different scaling and can be rebuilt from the log. ADR 3 explains the cost.
- *What happens if the projection logic changes?* Reset the consumer group offset and replay into a fresh table.
- *How is "today" defined for a streak?* In the learner's zone with an injected clock, so a review at 00:30 local
  time counts for the right day.

**What I'd do next.** FSRS already runs as a switchable second `SchedulingStrategy` (`STUDYFORGE_SCHEDULER=fsrs`,
ADR 0004); next would be fitting its weights to a learner's own review log, a Redis cache for "due today", and a
test that runs two insights replicas to show partition rebalancing.

---

## Questions that cut across all of them

- *Why so many stacks?* To show the patterns are mine, not a framework's: the same outbox idea appears as
  MassTransit, a hand-written TypeScript relay and Spring Modulith's registry, and the same real-time need as
  SignalR, socket.io, STOMP, GraphQL subscriptions and SSE.
- *How do you know it works?* Every repo has unit and architecture tests, Testcontainers integration tests
  against the real infrastructure, a compose smoke script that drives the running stack, and CI running all of it.
- *What is not production-ready?* Most apps have no authentication, the Bicep is compiled but never deployed, and
  several have single-instance limits. The [README](../README.md#what-is-not-production-ready) lists them per app.
