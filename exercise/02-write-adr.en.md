**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No. It had no numbers, so latency, peak load and MailFast limits are all listed as Open questions (#3, #4, #7). | Yes, all of them: p95 ≤ 500 ms, 200 orders/s, MailFast p95 1.5 s / p99 8 s / 0.5 % errors / 50 per s, points within 1 min exactly once, team and ops constraints. It also computes from them: 75 % of emails over the limit at peak, a backlog of 9,000 emails per minute, about 75 MailFast calls in flight. |
| Options considered (list them) | A: synchronous direct calls; B: publish/subscribe via a message broker; C: a post-order orchestrator service | 1: synchronous gRPC; 2: in-process async goroutines; 3: RabbitMQ events (self-hosted); 4: Kafka event log (self-hosted) |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Exactly once: idempotency on order ID (unique constraint in the loyalty store). Rate limit: not handled, because the limit was unknown; only generic retries, backoff and a DLQ. | Exactly once: at-least-once delivery plus an idempotent `loyaltyservice` that stores the `order_id` in the same transaction as the points and acks only after commit. Rate limit: the email consumer is throttled to ≤ 50 emails/s across all replicas, and the backlog waits in the queue. |
| Anything invented (a number, tool or fact not in the scenario)? | No numbers invented, and guesses about the current system are labelled [Inferred]. But two claims don't fit the scenario: it recommends "prefer a managed broker" (the company prefers self-hosted), and it says events "can be replayed", which is not true for every broker (e.g. RabbitMQ deletes consumed messages). | No scenario numbers invented, and the code references to `main.go` are correct. Weak spots: the Kafka "cons" cell ends with a broken, unfinished sentence ("more than 200 events/s needs [Inferred]"), and the publish time ("few ms") is only a guess (labelled [Inferred]). |
## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Set **Deciders** to my name and added a note that the ADR is based on the AI draft | The AI left Deciders open; I am the architect who signs this ADR |
| 2 | Fixed the broken, unfinished sentence in the Kafka "Cons" cell | The AI's text was incomplete |
| 3 | Diagram: `frontend` technology is **Go** and Customer → `frontend` is **HTTP** (removed [Inferred]) | I verified this in Task 1: `src/frontend/*.go` and the `frontend-external` http port |
| 4 | **Decision:** accept the email delay at the 11.11 peak; the confirmation page tells customers the email may take a few minutes; alert on queue age | An email is not needed to complete an order. Waiting for MailFast to raise its limit would block the release and add cost |
| 5 | **Decision:** `emailservice` deduplicates on `order_id` (a sent log), plus a MailFast idempotency key if supported; added a duplicate-email test | The AI accepted possible duplicate emails; duplicate confirmations confuse customers |
| 6 | Kept the AI's handling of a broker outage (3-node cluster, log and replay) and marked it as an **accepted risk** | I did not add a transactional outbox yet, because `checkoutservice` has no store today; this stays as Open question 4 |
