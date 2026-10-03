# Payment and Wallet Reliability in Go

By Manenimabasi Udoh | Senior Backend Engineer

Financial workflows need to remain correct when requests overlap, providers retry, and events arrive more than once. I created Go payment and wallet services and evolved their integration around explicit identities, database transactions, and replay-safe processing.

## Stable payment identity across retries

I separated the logical payment ID from the payment provider's retry reference. A payment keeps its stable internal UUID while each provider initiation receives a unique reference. This preserves the business identity of the payment without conflating individual provider attempts.

The payment service uses PostgreSQL persistence and gRPC interfaces for service-to-service initiation and confirmation. Wallet-backed payment flows also require recovery when a debit succeeds before downstream payment finalization completes.

## Wallet correctness under concurrency

I built the wallet service in Go and PostgreSQL with exact decimal balance handling. Balance mutations acquire a transaction-scoped row lock using `SELECT FOR UPDATE`. The balance update and immutable ledger entry commit in the same transaction, while database constraints prevent negative balances.

Unique transaction references make repeated credit or debit requests idempotent. Concurrency and idempotency tests exercise overlapping debit behavior, so correctness is checked at the transaction boundary rather than inferred from sequential requests.

## Durable payment events and duplicate-safe consumption

I implemented a PostgreSQL transactional outbox so finalized payment state and its integration event are written atomically. A versioned payment-finalized contract carries stable event IDs and integer minor-unit money fields across service boundaries.

I configured Debezium and Kafka Connect for PostgreSQL CDC and verified the complete PostgreSQL-to-Debezium-to-Kafka path with a live local smoke test. This verification establishes the integration path in the local environment.

On the wallet side, I added an inbox/receipt transaction that combines event receipt, balance mutation, and ledger insertion. Duplicate event IDs or ledger references become successful no-ops. The delivery model is at least once, with idempotent business effects.

## Recovering after partial failures

I hardened downstream consumers with stable event identities and processing checkpoints. Permanent invalid payloads go to dead letters with source context. Transient processing failures remain retryable rather than being acknowledged as completed.

Resumable stages prevent replay from repeating completed financial or business effects after a partial failure. I documented consumer-first migration, direct/dual/outbox publication modes, connector recovery, and rollback procedures.

## Engineering judgment

The key design choice is to place each correctness guarantee at an enforceable boundary: the database transaction for financial state, unique references for request identity, durable receipts for event deduplication, and checkpoints for multi-stage recovery. This makes retry behavior deliberate and testable.

Related public projects: [Distributed Task Orchestrator](https://github.com/manenim/task-orchestrator) and [Distributed Rate Limiter](https://github.com/manenim/gateway-rate-limiter).
