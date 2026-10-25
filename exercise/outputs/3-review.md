Agent: Claude (Anthropic), fresh agent session; model: claude-opus-5-5 (as configured)

# Review: ADR-0002, use RabbitMQ order events for post-order actions

Reviewed: `docs/adr/0002-use-rabbitmq-for-post-order-actions.md` (cited as "ADR l.N") against `docs/scenario.md`.

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| S1 Performance | Customers | 200 `PlaceOrder`/s | 11.11 peak; email queue backlog growing by 150 msgs/s; MailFast at p99 8 s | Payment and shipping run synchronously, then `OrderPlaced` is published with a publisher confirm | `PlaceOrder` p95 ≤ 500 ms, so publish + confirm + retry must fit in the ~200 ms headroom (scenario.md; ADR l.31) |
| S2 Availability / correctness | Infrastructure | RabbitMQ unreachable, or `loyaltyservice` killed before it acks | Peak load, card already charged (ADR l.18) | Order still succeeds; the event survives; points are awarded after recovery | 0 failed orders; 0 lost and 0 duplicate awards; order-to-points lag ≤ 60 s (scenario.md) |
| S3 Throughput | MailFast | 50 emails/s cap, ~0.5 % failures | Peak of T minutes at 200 orders/s | Queue buffers; consumer throttled across all replicas; retries with backoff | 0 rate-limit rejections; every email sent or in DLQ; worst email delay ≈ 3·T min (ADR l.122 math) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | If the broker is down, the event is only "logged at error level" after the card is charged (ADR l.82, l.127). `checkoutservice` has no store (ADR l.24), so this event is the **only** durable record of the order for loyalty and analytics. | S2 fails: points are lost, which breaks "awarded within 1 minute, exactly once". Logs are not a durable queue, and the log line holds email and shipping address (privacy). The ADR's "effectively exactly once" (l.70) is really "at most once" during a broker outage. |
| Risk | Peak backlog in the broker is not sized. A 10-minute peak leaves 90,000 messages in `email.order-confirmation`, and the last email waits ~30 min. | The ADR does not say what happens to the publish path when the broker is short on memory or disk. **[Inferred]** RabbitMQ blocks publishers under resource alarms, which would put the email backlog back into `PlaceOrder` latency (S1) — the coupling this ADR is meant to remove. |
| Risk | "Throttled to ≤ 50/s in total across all replicas" (ADR l.76) has no mechanism. | A per-replica limit multiplied by the replica count breaks MailFast's cap (S3). The same coordination problem is listed as a con of option 2 (ADR l.64) but is not solved for option 3. |
| Sensitivity point | Publish latency with publisher confirms on a replicated quorum queue, and the bounded retry budget (ADR l.74-75, l.82). | All of S1's ~200 ms headroom depends on it. The "few ms" figure is marked [Inferred] (ADR l.65) and is not measured. |
| Sensitivity point | 11.11 peak duration T (open question 1). | Customer-visible email delay (≈ 3·T), broker storage, and whether "may take a few minutes" (ADR l.77) is true. |
| Trade-off point | Email dedupe store (ADR l.78): correctness (no duplicate emails) vs. operability and cost (one more store). Recording *after* the send still leaves a crash window, unless MailFast accepts an idempotency key (unknown, open question 2). | This goes beyond R-12, which does not require no duplicate emails. It is a fair choice, but the ADR should state that duplicates are still possible in that window. |
| Trade-off point | Queue per action (ADR l.75): availability, modifiability and isolation (email never delays loyalty) vs. operability (3 queues + 3 DLQs now, more next year, each with alerts). | A good trade for this load, but the operating cost grows with every new action. |
| Non-risk | `loyaltyservice` stores the processed `order_id` under a unique constraint in the same transaction as the points, and acks after commit (ADR l.79). | Correctly turns at-least-once delivery into exactly-once points, as long as the event is never lost (see first Risk). |
| Non-risk | Publish failure does not fail the order (ADR l.82); payment and shipping stay unchanged. | Keeps the "never fail because email/loyalty/analytics is down" goal and today's email behaviour (ADR l.20). |
| Non-risk | Throughput and option choice: 200 events/s is small; RabbitMQ gives built-in DLQs and retries; two developers know it (scenario.md). | Rejecting Kafka on effort is justified for this load (ADR l.66, l.70). |

## 3. Missing or weak

- **Option not considered:** a transactional outbox (add a small store to `checkoutservice`) as its own option, rather than a deferred follow-up (ADR l.127). It is the only option here that fully meets S2. The ADR treats "no store today" as fixed, but adding a store is a design choice.
- **Claim likely wrong or incomplete:** "messages are deleted after they are consumed, so there is no built-in history to replay" (ADR l.65). **[Inferred]** RabbitMQ also offers streams, a retained log. This weakens the "replay favours Kafka" argument (open question 5).
- **Unsupported claims stated as results:** "loyalty points arrive within the 1-minute target" (ADR l.118) and "never goes over MailFast's limit" (l.117) are goals still to be validated (l.136, l.139), not consequences.
- **Consequences not stated:** the cost to `PlaceOrder` latency when the broker is slow or blocked; that retries use part of the 50/s budget (small at 0.5 % failures, but it should be counted); PII in error logs (l.82), not just in queues (l.125).
- **Validation gap:** no test of `PlaceOrder` p95 with the broker **down or blocked** at 200 orders/s (l.140 only kills one node), and no test of a long peak that fills broker memory.
- **Modifiability limit:** fraud scoring may need to block before payment (open question 6). If so, the "no change to `checkoutservice`" claim (l.119) does not hold for it.

## 4. Verdict: **Accept with changes**

1. **Close or formally accept the event-loss gap.** Either add the transactional outbox now, or get written sign-off from the loyalty owner that a broker outage can lose points, plus a reconciliation process. Replace "log at error level" with a durable, PII-safe fallback.
2. **Bound the publish path and size the broker.** Set a hard publish/confirm timeout inside the ~200 ms headroom, size the broker for the peak duration T, state what happens under broker back-pressure, and add load tests with the broker down or blocked.
3. **Specify the global 50/s throttle** (mechanism, with retries counted toward it), and replace "may take a few minutes" with a delay figure derived from T. Correct the RabbitMQ replay claim, and restate the unvalidated "Positive" consequences as targets.
