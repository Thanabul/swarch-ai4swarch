**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | the ADR's author | defends the ADR |
| **AI analyst** | | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | | role card P (business side) |
| **Site reliability engineer (SRE)** | | role card S (operations side) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-use-rabbitmq-for-post-order-actions.md` by Thanabul Parodom
**Board members and roles:** All roles (architect, AI analyst, product owner, SRE, security & privacy officer) were played by me, Thanabul Parodom, because I attended class online; the instructor allowed me to do Part B individually. I read the role cards only after finishing my final ADR and the AI review. The comparison below was drafted with AI help from my own notes on the role cards.

**Decision:** accept with changes

**Required changes:**
1. **Phase the rollout to fit the 3 weeks before 11.11.** Start the 3-node RabbitMQ budget approval (2 weeks, including the extra Kubernetes nodes) today, and prepare monitoring, alerts, backups and the runbook (1 week) in parallel, leaving the last week for the load test. If approval is not done in time, 11.11 runs on today's synchronous email path and the broker goes live after the sale.
2. **Show loyalty points on the confirmation page immediately.** `checkoutservice` calculates the points earned from the order total with the same rule as `loyaltyservice` and shows them as "pending"; `loyaltyservice` still credits them asynchronously, exactly once per `order_id`, within 1 minute.
3. **Protect personal data.** Analytics receives its own event without email or name; queues and logs that hold email or address are encrypted at rest and kept at most 30 days; replace "log the full event at error level" with a PII-safe fallback. MailFast is not used until legal approves the cross-border transfer under PDPA.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | The confirmation page must show the loyalty points total immediately | PO | No | No. The scenario only says "within 1 minute"; this requirement exists only on the PO card, and it conflicts with the scenario |
| 2 | Points are real money (100 points = 10 THB): double points are a direct loss, missed points cause complaints | PO | Partly: it flagged lost points during a broker outage | Partly. "Exactly once" is in the scenario, but the money value of points is not |
| 3 | Only 3 weeks until 11.11; the design must be built, tested and load-tested in time | PO | No | No. No deadline is given in the scenario |
| 4 | Fraud scoring next year must be able to block an order before confirmation | PO | Yes: "fraud scoring may need to block before payment" | Partly. The ADR already listed it as an open question |
| 5 | No one has run a message broker in production; only 2 people in the on-call rotation | SRE | Partly: only "operating cost grows with every queue" | No. Team operations staffing is not in the scenario |
| 6 | A new stateful component needs ~1 week for monitoring, alerts, backups and a runbook; a 3-node cluster needs 2 weeks of budget approval; the Kubernetes cluster is near its node limit | SRE | No. It even endorsed the 3-node cluster | No. Internal processes and cluster capacity are not in the scenario |
| 7 | Last year's 11.11 email outage went unnoticed for 20 minutes: replay, DLQ and tracing across async hops must be clear | SRE | Partly: no test with the broker down or blocked | Partly. The past incident is not in the scenario |
| 8 | MailFast stores data in the US; PDPA cross-border transfer is not yet approved by legal | Security | No | No. Legal status and data location are not in the scenario |
| 9 | Queues and logs with email or address must be encrypted at rest and kept at most 30 days | Security | Partly: PII in error logs | Partly. The 30-day rule is company policy, not in the scenario |
| 10 | Analytics must never receive email or real name | Security | No | Partly. The policy is not in the scenario, but the ADR sends one event with the email to every queue, so the AI could have spotted it |
| 11 | During a broker outage the event is lost after the card is charged ("at most once") | AI review | Yes | Yes. It follows from the ADR text |
| 12 | The "≤ 50 emails/s across all replicas" throttle has no mechanism | AI review | Yes | Yes. It follows from the ADR and the MailFast limit |

**The AI's best finding** (one the humans had missed): The global 50 emails/s throttle has no mechanism; a per-replica limit multiplied by the number of replicas would break MailFast's limit. None of the role cards raised it.
**The AI's weakest finding** (generic, wrong or irrelevant): The claim that RabbitMQ streams can replay messages. It is general product knowledge, not based on the scenario, and irrelevant to the board's real blockers (timeline, budget approval, PDPA).

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
