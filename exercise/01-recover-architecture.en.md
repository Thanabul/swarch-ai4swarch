**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| `frontend` → `cartservice` | `frontend.yaml` `CART_SERVICE_ADDR` (L73-74) | ✅ | L73-74: `CART_SERVICE_ADDR = "cartservice:7070"`, matches the AI |
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` `PAYMENT_SERVICE_ADDR` (L61-62) | ✅ | L61-62: `"paymentservice:50051"`. The code also calls it: `src/checkoutservice/main.go` L116 reads the env var, L123 connects with gRPC, L370 calls `PaymentService.Charge` |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` `EMAIL_SERVICE_ADDR` (L63-64) | ✅ | L63-64: `"emailservice:5000"`. `main.go` L380 calls `SendOrderConfirmation` |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` `PRODUCT_CATALOG_SERVICE_ADDR` (L60-61) | ✅ | L60-61: `"productcatalogservice:3550"`, matches the AI |
| `cartservice` → `redis-cart` | `cartservice.yaml` `REDIS_ADDR` (L50-51) | ✅ | L50-51: `"redis-cart:6379"`. The code uses it: `src/cartservice/src/Startup.cs` L29 reads `REDIS_ADDR` |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question):
  - **Missing container:** `shoppingassistantservice` exists in `src/` and `frontend` calls it over **HTTP**, not gRPC (`src/frontend/main.go` L139 `SHOPPING_ASSISTANT_SERVICE_ADDR`, `handlers.go` L464 `"http://" + ...`). Run B only listed it under Gaps because it has no manifest.
  - **Missing optional dependency:** `checkoutservice` connects to an OpenTelemetry collector over gRPC when `ENABLE_TRACING=1` (`main.go` L90, L164-165, `COLLECTOR_SERVICE_ADDR`). This is invisible in the K8s manifests.
  - **Container to question:** `loadgenerator` is drawn inside the system boundary, but it is a dev/test traffic tool, not part of the shop (it is commented out of `kustomization.yaml`).
  - **Weak:** languages are all "Not evidenced" because Run B could not see `src/`.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
