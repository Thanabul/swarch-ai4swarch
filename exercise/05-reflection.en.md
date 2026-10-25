**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Thanabul Parodom / 6872036421
**My role on the review board:** All roles (architect, AI analyst, product owner, SRE, security), done individually with the instructor's permission

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I would trust AI with the analytical work that is based on evidence it can read: recovering dependencies from manifests, applying scenario numbers, and checking an ADR for internal gaps. With the scenario, its ADR used every number correctly, and its review found a real flaw the role cards missed: the 50 emails/s limit had no mechanism across replicas.

But I would keep decisions for humans. On the board, the three blockers were invisible to the AI: only 3 weeks remain before 11.11 while budget approval and setup also take 3 weeks, the product owner needs points shown immediately, and legal has not approved MailFast under PDPA. The AI's review was plausible but not appropriate, because it did not know the company's timeline, processes or law. AI should be the analyst; humans must stay the decision makers.

_(Drafted with AI help, checked and edited by me.)_
