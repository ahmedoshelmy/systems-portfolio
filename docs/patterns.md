# Patterns, with the code that implements them

Each section explains a pattern in a few lines and links to the exact file on `main` that does it. Where two apps
do the same thing differently, both are linked, because the contrast is the interesting part.

## Clean Architecture and DDD

Domain code is plain language code with no framework, database or broker imports; use cases depend on ports
(interfaces); adapters implement the ports; the composition root wires them. Every app enforces the dependency
rule automatically, so a layer leak fails the build rather than a code review.

- .NET, a test per rule: [qirsh `DependencyRuleTests.cs`](https://github.com/ahmedoshelmy/qirsh/blob/main/tests/Qirsh.ArchitectureTests/DependencyRuleTests.cs),
  [dose-keeper `LayeringTests.cs`](https://github.com/ahmedoshelmy/dose-keeper/blob/main/tests/DoseKeeper.ArchitectureTests/LayeringTests.cs)
- Java, ArchUnit plus Spring Modulith: [trip-split `LayeringTest.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/test/java/dev/tripsplit/LayeringTest.java),
  [trip-split `ModularityTests.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/test/java/dev/tripsplit/ModularityTests.java)
- TypeScript, a lint rule: [price-watch-eg `eslint.config.mjs`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/eslint.config.mjs)
- A pure domain function at the heart of an app: [qirsh SMS parser lives in the Ingestion domain](https://github.com/ahmedoshelmy/qirsh/tree/main/src/Ingestion),
  [job-radar `MatchScorer.cs`](https://github.com/ahmedoshelmy/job-radar/blob/main/src/JobRadar.Matching/Domain/MatchScorer.cs),
  [upkeep `interval-rule.ts`](https://github.com/ahmedoshelmy/upkeep/blob/main/apps/api/src/domain/interval-rule.ts)
- An aggregate that refuses illegal state changes: [job-radar `JobApplication.cs`](https://github.com/ahmedoshelmy/job-radar/blob/main/src/JobRadar.Tracker/Domain/JobApplication.cs)

## CQRS

Writes and reads take different paths and may use different models. In Upkeep it is explicit (`@nestjs/cqrs`
command and query buses, plus a saga reacting to events); in Trip Split it falls out of event sourcing (commands
append events, "now" reads hit projections, "then" reads replay); in Study Forge the read side is a separate
service fed by Kafka.

- [upkeep `reminder.saga.ts`](https://github.com/ahmedoshelmy/upkeep/blob/main/apps/api/src/application/reminder.saga.ts)
- [trip-split `TripCommands.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/main/java/dev/tripsplit/trips/application/TripCommands.java)
  and [`TripQueries.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/main/java/dev/tripsplit/trips/TripQueries.java)
- [study-forge insights `Projector.java`](https://github.com/ahmedoshelmy/study-forge/blob/main/insights-service/src/main/java/dev/studyforge/insights/application/Projector.java)

## Event sourcing

State is the left fold of an append-only list of events. Trip Split's event store is one Postgres table with
`UNIQUE (stream_id, version)`: that index is the whole optimistic-concurrency story, and the same version is sent
as the HTTP `ETag`, so a stale `If-Match` becomes `412`. Undo is a compensating event; time travel is a replay up
to a timestamp.

- [trip-split `JdbcEventStore.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/main/java/dev/tripsplit/eventstore/internal/JdbcEventStore.java)
- [trip-split `Trip.java` (the aggregate)](https://github.com/ahmedoshelmy/trip-split/blob/main/src/main/java/dev/tripsplit/trips/domain/Trip.java)

## Transactional outbox and inbox

A crash between "save" and "publish" either loses an event or, with naive retries, sends it twice. The outbox
writes the event in the same database transaction as the state change and a relay publishes it later; the inbox
(or a dedupe table) makes a redelivered message a no-op. Result: at-least-once delivery, exactly-once effect.

- MassTransit EF Core outbox + inbox: [qirsh `MessagingExtensions.cs`](https://github.com/ahmedoshelmy/qirsh/blob/main/src/Qirsh.ServiceDefaults/MessagingExtensions.cs),
  [job-radar `CatalogDbContext.cs`](https://github.com/ahmedoshelmy/job-radar/blob/main/src/JobRadar.Catalog/Infrastructure/CatalogDbContext.cs)
- Hand-written relay with `SKIP LOCKED` and the trace context stored on the row: [price-watch-eg `outbox-relay.ts`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/apps/api/src/watches/infrastructure/messaging/outbox-relay.ts)
- Spring Modulith event publication registry: [trip-split `TimelineProjector.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/main/java/dev/tripsplit/timeline/application/TimelineProjector.java),
  [gym-scout `MembershipService.java`](https://github.com/ahmedoshelmy/gym-scout/blob/main/src/main/java/dev/gymscout/memberships/application/MembershipService.java),
  [study-forge `EventExternalization.java`](https://github.com/ahmedoshelmy/study-forge/blob/main/learning-service/src/main/java/dev/studyforge/learning/adapter/messaging/EventExternalization.java)
- Inbox idea over HTTP: Listmate stores each client `mutationId` with the change in one transaction:
  [listmate `SyncService.cs`](https://github.com/ahmedoshelmy/listmate/blob/main/src/Listmate.Application/Sync/SyncService.cs),
  and prunes the log later: [`ProcessedMutationRetention.cs`](https://github.com/ahmedoshelmy/listmate/blob/main/src/Listmate.Infrastructure/Retention/ProcessedMutationRetention.cs)

## Pub/sub (RabbitMQ, Kafka, Redis)

Publishers do not know their subscribers. Job Radar's `JobPostingAdded` has two independent subscribers (matching
and a stats projection), each on its own queue, so either can be down without the other noticing. Study Forge
uses Kafka instead because its read models are projections that can be rebuilt by replaying the log.

- [job-radar `StatsProjection.cs`](https://github.com/ahmedoshelmy/job-radar/blob/main/src/JobRadar.Api/Stats/StatsProjection.cs)
- [price-watch-eg `rabbit-bus.ts`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/libs/platform/src/rabbit-bus.ts) (publisher confirms, dead-letter queue, trace headers)
- [study-forge `EventListeners.java`](https://github.com/ahmedoshelmy/study-forge/blob/main/insights-service/src/main/java/dev/studyforge/insights/adapter/messaging/EventListeners.java)
- [upkeep `redis-notification-bus.ts`](https://github.com/ahmedoshelmy/upkeep/blob/main/apps/api/src/infrastructure/messaging/redis-notification-bus.ts)

## Competing consumers vs fan-out

Work that must happen once goes to one shared durable queue, and adding consumers adds throughput (with prefetch
as back-pressure). Notifications that every replica must see go to a queue per replica instead.

- [price-watch-eg `event-subscriptions.ts`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/apps/api/src/watches/infrastructure/messaging/event-subscriptions.ts)
- [upkeep `reminder-worker.ts`](https://github.com/ahmedoshelmy/upkeep/blob/main/apps/api/src/infrastructure/queue/reminder-worker.ts) (BullMQ worker; `docker compose up --scale worker=3`)

## gRPC

Contract-first RPC with deadlines. Qirsh uses it for the one question asked synchronously ("how much has this
category spent this month?") while facts flow as events. Price Watch uses server streaming so a search deadline
still yields the shops that already answered.

- [qirsh `ledger_query.proto`](https://github.com/ahmedoshelmy/qirsh/blob/main/src/Qirsh.Contracts/protos/ledger_query.proto),
  [`LedgerGrpcSpendQuery.cs`](https://github.com/ahmedoshelmy/qirsh/blob/main/src/Budgets/Qirsh.Budgets.Infrastructure/LedgerGrpcSpendQuery.cs)
- [price-watch-eg `scraper.proto`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/libs/contracts/proto/scraper.proto),
  [`grpc-scraper-client.ts`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/apps/checker/src/infrastructure/grpc-scraper-client.ts)

## Real-time push (SignalR, STOMP, socket.io, GraphQL subscriptions, SSE)

Five transports for the same need, chosen to fit each stack. In every app the push is a hint and the database is
the truth: a client that missed a message re-reads.

- SignalR: [qirsh `AlertsHub.cs`](https://github.com/ahmedoshelmy/qirsh/blob/main/src/Budgets/Qirsh.Budgets.Api/AlertsHub.cs),
  [listmate `ListsHub.cs`](https://github.com/ahmedoshelmy/listmate/blob/main/src/Listmate.Api/Hubs/ListsHub.cs) with a Redis backplane in
  [`ScaleOut.cs`](https://github.com/ahmedoshelmy/listmate/blob/main/src/Listmate.Api/Hosting/ScaleOut.cs)
- STOMP: [trip-split `WebSocketConfig.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/main/java/dev/tripsplit/live/WebSocketConfig.java)
- socket.io: [price-watch-eg `prices.gateway.ts`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/apps/api/src/watches/presentation/prices.gateway.ts)
- GraphQL subscriptions over graphql-ws: [upkeep `resolvers.ts`](https://github.com/ahmedoshelmy/upkeep/blob/main/apps/api/src/presentation/graphql/resolvers.ts)
- Server-Sent Events: [study-forge `LiveStats.java`](https://github.com/ahmedoshelmy/study-forge/blob/main/insights-service/src/main/java/dev/studyforge/insights/adapter/web/LiveStats.java)

## Offline sync

The client keeps an outbox of intent mutations in IndexedDB, each with an idempotency key and a client-chosen item
id, and replays it when back online. The server applies a batch exactly once, resolves conflicts per field
(last writer wins, delete wins over edit), and every list has a gap-free version so `changes?since=N` repairs
anything a socket missed. Postgres' `xmin` is the optimistic-concurrency token.

- [listmate web `outbox.ts`](https://github.com/ahmedoshelmy/listmate/blob/main/src/Web/src/outbox.ts)
- [listmate `ConflictRules.cs`](https://github.com/ahmedoshelmy/listmate/blob/main/src/Listmate.Domain/Items/ConflictRules.cs)
- [listmate `ListmateDbContext.cs` (`xmin`)](https://github.com/ahmedoshelmy/listmate/blob/main/src/Listmate.Infrastructure/Persistence/ListmateDbContext.cs)
- Protocol write-up: [listmate `docs/sync-protocol.md`](https://github.com/ahmedoshelmy/listmate/blob/main/docs/sync-protocol.md)

## Serverless and Durable Functions

One eternal orchestration per schedule: wait for the dose time with a durable timer, send a reminder, then race
"the user answered" (an external event) against an escalation timer, mark missed if nobody answers, and
`ContinueAsNew` to keep the history small. A separate timer repairs loops that are missing, failed or terminated.
The weekly report is fan-out / fan-in.

- [dose-keeper `DoseReminderOrchestrator.cs`](https://github.com/ahmedoshelmy/dose-keeper/blob/main/src/DoseKeeper.Functions/Orchestration/DoseReminderOrchestrator.cs)
- [dose-keeper `WeeklyAdherenceOrchestrator.cs`](https://github.com/ahmedoshelmy/dose-keeper/blob/main/src/DoseKeeper.Functions/Orchestration/WeeklyAdherenceOrchestrator.cs)
- [dose-keeper `ReminderLoopRepairService.cs`](https://github.com/ahmedoshelmy/dose-keeper/blob/main/src/DoseKeeper.Application/ReminderLoopRepairService.cs)
- Poison handling (`maxDequeueCount`): [dose-keeper `host.json`](https://github.com/ahmedoshelmy/dose-keeper/blob/main/src/DoseKeeper.Functions/host.json)

## Geospatial

Let the database do geometry: `geography(Point,4326)` with a GiST index, `ST_DWithin` to filter in metres,
`ORDER BY location <-> :point` for an index-assisted nearest-neighbour scan, and a line with a corridor for "gyms on
my commute".

- [gym-scout `JdbcGymRepository.java`](https://github.com/ahmedoshelmy/gym-scout/blob/main/src/main/java/dev/gymscout/places/adapter/persistence/JdbcGymRepository.java)
- [gym-scout `V1__places.sql`](https://github.com/ahmedoshelmy/gym-scout/blob/main/src/main/resources/db/migration/V1__places.sql)

## Resilience

Outbound calls to free public APIs get a timeout, retry with jitter (honouring `Retry-After`), a circuit breaker
and a rate limiter, and a fallback where one makes sense. Inside a system, retries come with dead-letter queues so
one bad message cannot block the rest.

- Per-host pipeline with `Microsoft.Extensions.Http.Resilience`: [job-radar `FeedsServiceCollectionExtensions.cs`](https://github.com/ahmedoshelmy/job-radar/blob/main/src/JobRadar.Feeds/FeedsServiceCollectionExtensions.cs)
- Resilience4j rate limiter, bulkhead, retry, breaker and a straight-line fallback: [gym-scout `PoliteCaller.java`](https://github.com/ahmedoshelmy/gym-scout/blob/main/src/main/java/dev/gymscout/outbound/PoliteCaller.java),
  [`OsrmTravelTimes.java`](https://github.com/ahmedoshelmy/gym-scout/blob/main/src/main/java/dev/gymscout/scouting/adapter/routing/OsrmTravelTimes.java)
- Retrying a saga step with backoff through a clock port: [upkeep `retry.ts`](https://github.com/ahmedoshelmy/upkeep/blob/main/apps/api/src/application/retry.ts)
- Lease-based claiming so several schedulers never request the same work: [price-watch-eg `typeorm-due-watch-claims.ts`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/apps/api/src/watches/infrastructure/persistence/typeorm-due-watch-claims.ts)
- Non-blocking retry topics and a dead-letter topic: [study-forge `EventListeners.java`](https://github.com/ahmedoshelmy/study-forge/blob/main/insights-service/src/main/java/dev/studyforge/insights/adapter/messaging/EventListeners.java)
- Protecting a GraphQL server from expensive queries: [upkeep `query-limits.ts`](https://github.com/ahmedoshelmy/upkeep/blob/main/apps/api/src/presentation/graphql/query-limits.ts)

## Observability (OpenTelemetry)

Traces across HTTP, the broker and gRPC, viewable locally in the .NET Aspire dashboard
(`docker compose --profile obs up -d`). The trace context travels in AMQP headers, gRPC metadata and even the
outbox row, so a message published seconds later still joins the request's trace.

- [qirsh `ServiceDefaultsExtensions.cs`](https://github.com/ahmedoshelmy/qirsh/blob/main/src/Qirsh.ServiceDefaults/ServiceDefaultsExtensions.cs)
  and [`SkipBackgroundClientSpans.cs`](https://github.com/ahmedoshelmy/qirsh/blob/main/src/Qirsh.ServiceDefaults/SkipBackgroundClientSpans.cs) (sampler that hides outbox polling)
- [price-watch-eg `telemetry.ts`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/libs/platform/src/telemetry.ts)

## Testcontainers

Integration tests start the real infrastructure in containers (Postgres, RabbitMQ, Redis, Redpanda, PostGIS,
Azurite), host the services in-process and drive the whole flow end to end.

- [qirsh `SmsToBudgetFlowTests.cs`](https://github.com/ahmedoshelmy/qirsh/blob/main/tests/Qirsh.IntegrationTests/SmsToBudgetFlowTests.cs)
- [listmate `SyncFlowTests.cs`](https://github.com/ahmedoshelmy/listmate/blob/main/tests/Listmate.IntegrationTests/SyncFlowTests.cs) (the same batch fired twice in parallel lands once)
- [price-watch-eg `main-flow.int-spec.ts`](https://github.com/ahmedoshelmy/price-watch-eg/blob/main/tests/integration/main-flow.int-spec.ts)
- [upkeep `reminders.int-spec.ts`](https://github.com/ahmedoshelmy/upkeep/blob/main/tests/integration/reminders.int-spec.ts)
- [trip-split `TripFlowIntegrationTest.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/test/java/dev/tripsplit/TripFlowIntegrationTest.java)
- [gym-scout `GymScoutFlowIntegrationTest.java`](https://github.com/ahmedoshelmy/gym-scout/blob/main/src/test/java/dev/gymscout/GymScoutFlowIntegrationTest.java)
- [dose-keeper `AzuriteFixture.cs`](https://github.com/ahmedoshelmy/dose-keeper/blob/main/tests/DoseKeeper.IntegrationTests/AzuriteFixture.cs)
- [study-forge `InsightsFlowIntegrationTest.java`](https://github.com/ahmedoshelmy/study-forge/blob/main/insights-service/src/test/java/dev/studyforge/insights/InsightsFlowIntegrationTest.java)

## Property-based tests for money and ranking

Money is `long` minor units; splits use the largest-remainder method so shares always add up. jqwik checks it on
thousands of random inputs, and the same approach checks that settle-up needs at most n-1 transfers and that a gym
that is cheaper and closer never scores lower.

- [trip-split `Allocation.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/main/java/dev/tripsplit/money/Allocation.java),
  [`AllocationProperties.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/test/java/dev/tripsplit/money/AllocationProperties.java)
- [trip-split `SettleUp.java`](https://github.com/ahmedoshelmy/trip-split/blob/main/src/main/java/dev/tripsplit/settlement/domain/SettleUp.java)
- [gym-scout `ScorerProperties.java`](https://github.com/ahmedoshelmy/gym-scout/blob/main/src/test/java/dev/gymscout/scouting/ScorerProperties.java)

## Contract tests for events

Events are records with a JSON Schema each; a test checks that every field is required and correctly typed, that
example payloads round-trip, and that unknown fields are ignored so the contract can only grow additively.

- [study-forge `EventSchemaTest.java`](https://github.com/ahmedoshelmy/study-forge/blob/main/contracts/src/test/java/dev/studyforge/contracts/EventSchemaTest.java)

## Infrastructure as code (Bicep)

Azure Container Apps (consumption), Postgres Flexible Server, Service Bus or Redis where needed. CI compiles the
Bicep; none of it has been deployed.

- [qirsh `infra/main.bicep`](https://github.com/ahmedoshelmy/qirsh/blob/main/infra/main.bicep)
- [listmate `infra/main.bicep`](https://github.com/ahmedoshelmy/listmate/blob/main/infra/main.bicep)
- [dose-keeper `infra/main.bicep`](https://github.com/ahmedoshelmy/dose-keeper/blob/main/infra/main.bicep)

## CI

Every repo has one GitHub Actions workflow: format and lint, build with warnings as errors, unit and architecture
tests, Testcontainers integration tests, web lint/build/test, and Docker image builds.

- [listmate `ci.yml`](https://github.com/ahmedoshelmy/listmate/blob/main/.github/workflows/ci.yml) (includes a two-replica scale-out job)
- [trip-split `ci.yml`](https://github.com/ahmedoshelmy/trip-split/blob/main/.github/workflows/ci.yml)
