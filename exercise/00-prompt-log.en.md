**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID:** Thanabul Parodom / 6872036421
**Agent(s) used:** Claude (Anthropic) as the AI agent. Each run was a separate, fresh agent session that read only the files allowed by the task. A separate Claude chat helped me as a tutor (explaining the tasks, checking my worksheets).
**Model(s):** claude-opus-5-5 (as configured)
**Plan / tier:** Claude via claude.ai

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude, claude-opus-5-5 | `prompts/1-…` run A | AGENTS.md, CLAUDE.md, prompts/1-recover-architecture.md, protos/demo.proto | Yes | 3 | 9 services, 0 dependencies: the proto shows interfaces, not callers |
| 2 | 1-B | Claude, claude-opus-5-5 | `prompts/1-…` run B | AGENTS.md, CLAUDE.md, prompts/1-recover-architecture.md, protos/demo.proto, all 13 files in kubernetes-manifests/ | Yes | 4 | 12 containers, 17 dependencies. The session hit a usage limit at the very end; the output file was complete, files read were checked from its tool log |
| 3 | 2-a | Claude, claude-opus-5-5 | `prompts/2-…` run a | AGENTS.md, CLAUDE.md, prompts/2-write-adr.md, docs/adr/template.md | Yes (did not open scenario.md) | 3 | Generic pub/sub decision, no numbers, 15 open questions |
| 4 | 2-b | Claude, claude-opus-5-5 | `prompts/2-…` run b | AGENTS.md, prompts/2-write-adr.md, docs/adr/template.md, docs/scenario.md, docs/adr/0001-…, src/checkoutservice/main.go | Yes | 5 | Chose RabbitMQ using the scenario's numbers. Session hit a usage limit after writing; files read were checked from its tool log |
| 5 | 3 | Claude, claude-opus-5-5 | `prompts/3-…` adr: docs/adr/0002-use-rabbitmq-for-post-order-actions.md | AGENTS.md, CLAUDE.md, prompts/3-review-adr.md, docs/adr/0002-…, docs/scenario.md | Yes | 4 | Accept with changes; found the event-loss gap and the missing global throttle |
| 6 | Help | Claude (tutor chat) | Help to verify Task 1, compare 2a/2b, draft reflections and the board record | Read source files to check Task 1 claims; never read docs/review-board/ | n/a | 4 | AI-assisted: Task 1 verification table, Task 1 reflections, the 2a/2b comparison, applying my ADR decisions, drafting the board record from my notes on the role cards |
