# QuickBite Support Copilot — Software Requirements Specification

Oct 30, 2026 · @Praveen raja

How the copilot is built: a LangGraph state graph orchestrates two agents, MCP servers expose the order, payment and ticket systems as governed tools, and every model runs locally on Ollama.

## 1. Introduction

**Purpose.** This SRS specifies how to build the QuickBite Support Copilot described in the PRD tab, precisely enough that each requirement can be implemented and tested.

**Scope.** Customer chat, agent approval console, LangGraph orchestration, three MCP servers, policy RAG, rules engine, evaluation suite. Payment, auth and order systems are local mocks backed by Postgres.

**Definitions**

| Term | Meaning |
| --- | --- |
| Case | One complaint about one order, from intake to resolution |
| Decision | The rules engine output: eligible yes/no, amount, method, whether approval is required, reason codes |
| Read tool | MCP tool that only fetches data |
| Action tool | MCP tool that changes state (refund, credit, ticket) and needs an idempotency key |
| Interrupt | A LangGraph pause that persists state and waits for a human resume command |

**Tech stack.** Pin exact versions in `pyproject.toml` when you start; model names are examples, so set them in config.

| Layer | Choice |
| --- | --- |
| Language | Python 3.12 (backend), TypeScript (UI) |
| Orchestration | LangGraph (state graph, `interrupt()`, Postgres checkpointer) |
| Tool protocol | MCP Python SDK (`FastMCP`), streamable HTTP transport; `langchain-mcp-adapters` on the client side |
| Models | Ollama: a 3–4B model for intent and guards, a 7–8B tool-calling model for agents, `nomic-embed-text` for embeddings |
| Model gateway (optional) | LiteLLM in front of Ollama, for routing, timeouts and fallbacks |
| API | FastAPI with Server-Sent Events |
| Vector store | Qdrant with payload filters |
| Database | Postgres 16 (orders, payments, refunds, audit, checkpoints) |
| UI | React + Vite: customer chat and agent console |
| Tracing | Langfuse self-hosted, or OpenTelemetry spans to Postgres |
| Tests and evals | pytest + a custom eval runner over a JSONL golden set |

## 2. System architecture

&#91;embedded content: System architecture · 10 components\]

The graph never touches Postgres directly. Every order, payment and ticket call goes through an MCP server that re-checks the customer's JWT, and only those servers hold database credentials (NFR-07).

## 3. LangGraph design

The graph is mostly a deterministic workflow with two small agents placed where judgement is needed. They're split by **permission**, not by persona. The Investigator gets read tools only. The Resolver gets action tools only, and only after a decision is confirmed or approved.

**State**

```python
class CaseState(TypedDict):
    messages: Annotated[list, add_messages]
    customer_id: str            # from the JWT, never from the model
    correlation_id: str
    intent: Intent | None       # missing_item | wrong_item | late | quality | payment | other
    order_id: str | None
    facts: dict                 # order, items, timeline, payment, refund_history
    policy: list[dict]          # clauses with clause_id, version, score
    decision: Decision | None   # rules engine output incl. decision_id
    confirmation: Literal["accepted", "declined"] | None
    approval: Approval | None   # approver, action, final_amount, reason
    status: CaseStatus
    error: str | None
    tool_calls: int             # loop guard
```

**Nodes**

| Node | Type | Does | Next |
| --- | --- | --- | --- |
| `guard` | Small model + rules | Scope check, injection check, "other customer" check | `classify` or `refuse` |
| `classify` | Small model, JSON | Sets `intent` | `investigator` |
| `investigator` | **Agent** (ReAct, read tools) | Finds the order and gathers facts; asks the customer when ambiguous | `retrieve_policy`, or ends the turn with a question |
| `retrieve_policy` | Code | Qdrant search filtered by city, active status, intent | `rules` |
| `rules` | Code (no LLM) | Writes the `Decision` row | `respond`, `confirm` or `await_approval` |
| `confirm` | `interrupt()` | Waits for the customer's yes or no | `resolver` or `respond` |
| `await_approval` | `interrupt()` | Waits for the console decision | `resolver` or `respond` |
| `resolver` | **Agent** (action tools, bound to `decision_id`) | Executes refund or credit, checks status on timeout, creates a ticket if needed | `respond` |
| `respond` | 7–8B model, streamed | Explains the outcome from state, citing the clause | END |
| `handoff` | Code | Summary + ticket; reachable from any node on unrecoverable error | END |

&#91;embedded content: LangGraph flow · 11 nodes, 2 interrupts\]

The Investigator can also end a turn early to ask which order the customer means. A declined confirmation goes straight to `respond`, the same as a rejection.

**Human-in-the-loop**

```python
def await_approval(state: CaseState):
    answer = interrupt({"kind": "approval", "decision_id": state["decision"].id})
    return {"approval": answer, "status": "approved" if answer["action"] != "reject" else "rejected"}

# console endpoint resumes the paused thread
graph.invoke(Command(resume=payload), config={"configurable": {"thread_id": session_id}})
```

The Postgres checkpointer persists state at every step, so a pending approval survives a server restart (NFR-03). Set `recursion_limit` to about 25 supersteps (agent tool loops count too) and cap each agent at 3 tool calls per turn to stop loops.

## 4. MCP servers and tool contracts

Three MCP servers, each a thin, governed adapter over its own Postgres tables. No server exposes raw SQL. Every call carries the customer's JWT in the `Authorization` header; the server validates it and scopes every query to that `customer_id`. The model never supplies `customer_id`.

| Server | Tool / resource | Tier | Inputs | Returns | Rules enforced in code |
| --- | --- | --- | --- | --- | --- |
| `orders-mcp` | `list_recent_orders` | Read | `limit` (≤ 5) | order ID, restaurant, total, status, delivered\_at | Only the caller's orders |
| `orders-mcp` | `get_order` | Read | `order_id` | items with prices, totals, status | Order must belong to the caller, else `NOT_FOUND` (never `FORBIDDEN`, so IDs aren't leaked) |
| `orders-mcp` | `get_delivery_timeline` | Read | `order_id` | promised\_at, picked\_at, delivered\_at, delay\_minutes | Same ownership check |
| `payments-mcp` | `get_payment` | Read | `order_id` | method, amount, status, prior refunds | Masked method ("UPI •••• 21") |
| `payments-mcp` | `get_refund_history` | Read | — | count and total of refunds in the last 30 days | Caller only |
| `payments-mcp` | `issue_refund` | **Action** | `decision_id`, `idempotency_key` | refund\_id, status | Amount and order come from the stored decision, never from model arguments; decision must be approved or confirmed; same key returns the same refund |
| `payments-mcp` | `issue_wallet_credit` | **Action** | `decision_id`, `idempotency_key` | credit\_id | Same as above |
| `support-mcp` | `create_ticket` | Action (low risk) | `order_id`, `summary`, `category` | ticket\_id | Summary PII-scrubbed; idempotent per case |
| `support-mcp` | `policy://refund/{city}` | Resource | city | current policy text and version | Read only |

**Design rules for every tool**

- Pydantic input schema; reject unknown fields.
- Timeout of 3 s for reads and 5 s for actions; errors returned as typed codes (`NOT_FOUND`, `NOT_ELIGIBLE`, `DECISION_NOT_APPROVED`, `UPSTREAM_TIMEOUT`).
- One audit row per call: correlation ID, customer ID, tool, input summary, result code, latency.
- Tool descriptions say *when* to use the tool and when not to, because the description is what drives tool selection.

**Why `decision_id` instead of `amount`:** an agent can't be talked into refunding ₹5000, because the action tool only executes a decision the rules engine already wrote and that a customer confirmed or a human approved.

## 5. Model configuration (Ollama)

Use the smallest model that reliably does each job. Model names live in `config/models.yaml`; the names below are starting points to benchmark on your GPU.

| Role | Used by node | Model class (example) | Settings | Fallback |
| --- | --- | --- | --- | --- |
| Intent + guard classifier | `guard`, `classify_intent` | 3–4B instruct (e.g. `llama3.2:3b`, `qwen3:4b`) | temperature 0, JSON output, max 64 tokens | Keyword rules → `other` |
| Tool-calling agent | `investigator`, `resolver` | 7–8B with native tool calling (e.g. `qwen3:8b`, `llama3.1:8b`) | temperature 0.1, max 3 tool calls per turn | Retry once, then human handoff |
| Response writer | `respond` | Same 7–8B model | temperature 0.3, max 300 tokens, streamed | Template response from the decision |
| Groundedness judge (evals only) | eval runner | Largest model you can run locally | temperature 0 | Human spot check |
| Embeddings | ingestion, `retrieve_policy` | `nomic-embed-text` | 768-dim, cosine | — |

**Rules**

- Call Ollama's native chat API with `tools` (or `ollama_chat/` through LiteLLM), never a hand-rolled XML tool format. This is the bug you already hit at AHS, so it makes a good story.
- Every call has a timeout (classifier 5 s, agent 30 s) and is wrapped by the fallback in section 10.
- Log model name, prompt version, tokens in and out, and latency on every call.

## 6. Functional requirements

| ID | Requirement | Priority | Acceptance criteria |
| --- | --- | --- | --- |
| FR-01 | The system shall stream assistant replies to the chat UI over SSE. | Must | First token reaches the browser before the full reply is generated |
| FR-02 | The system shall classify each new complaint into one of 6 intents. | Must | ≥ 90% accuracy on the golden set |
| FR-03 | The system shall identify the order in question, asking the customer to choose when more than one recent order matches. | Must | Never assumes an order when 2+ match |
| FR-04 | The system shall fetch order, delivery and payment facts only through MCP read tools scoped to the authenticated customer. | Must | Requesting another customer's order ID returns "not found" |
| FR-05 | The system shall retrieve policy clauses from Qdrant filtered by city, `status=active` and latest version. | Must | Answers cite clause ID and version |
| FR-06 | The system shall compute eligibility, amount, method and approval need using the rules engine only. | Must | 100% match with rule unit tests; LLM output never changes the amount |
| FR-07 | The system shall persist each decision with a unique `decision_id` before any action. | Must | Action tools reject unknown or unapproved decision IDs |
| FR-08 | The system shall ask the customer to confirm before executing an auto-eligible refund. | Must | No refund row created without a confirmation event |
| FR-09 | The system shall pause the graph with an interrupt when approval is required, and resume from the checkpoint after the agent's decision. | Must | Restarting the API server mid-wait doesn't lose the case |
| FR-10 | The agent console shall show the case card: complaint, facts, policy clause, decision, history, and approve / reduce / reject actions. | Must | Approver can act in one screen; a reduced amount can't exceed the proposed one |
| FR-11 | The system shall execute refunds idempotently using key `{order_id}:{issue_type}`. | Must | Replaying the action 5 times yields 1 refund |
| FR-12 | The system shall block out-of-scope requests, prompt-injection attempts and requests about other customers before any tool call. | Must | All 15 safety cases in the golden set blocked |
| FR-13 | The system shall hand off to a human with an auto-generated summary on any unrecoverable failure or customer request. | Must | Handoff contains order, intent, facts gathered, last error |
| FR-14 | The system shall create a support ticket when a case is unresolved after handoff or the approval SLA expires. | Should | Ticket exists with case link |
| FR-15 | The system shall record a trace per turn and an audit row per tool call. | Must | Any refund can be traced to its conversation, decision, approver and tool call |
| FR-16 | The system shall expire pending approvals after a configurable SLA (default 30 min) and notify the customer. | Should | Expired cases move to ticket and the customer gets a message |

## 7. Non-functional requirements

| ID | Category | Requirement |
| --- | --- | --- |
| NFR-01 | Performance | P95 time to first token ≤ 3 s and P95 full turn ≤ 12 s on the reference machine; per-step latency logged |
| NFR-02 | Performance | Policy retrieval P95 ≤ 300 ms; MCP read tools P95 ≤ 200 ms |
| NFR-03 | Reliability | A server restart during a pending approval loses no state (Postgres checkpointer) |
| NFR-04 | Reliability | Action tools are idempotent; read tools retried once on timeout; action tools never retried blindly |
| NFR-05 | Reliability | Circuit breaker on Ollama and each MCP server: opens after 5 consecutive failures, half-opens after 30 s |
| NFR-06 | Security | JWT validated at the API and again in each MCP server; the MCP client is created per session with that customer's token |
| NFR-07 | Security | Agents never receive DB credentials; only MCP servers talk to Postgres, each with its own least-privilege DB role |
| NFR-08 | Security | Retrieved policy text and tool outputs are wrapped as data in prompts; instructions inside them are ignored |
| NFR-09 | Privacy | Payment methods masked in tool outputs; no card or UPI identifiers in prompts, logs or traces; phone numbers redacted from ticket summaries |
| NFR-10 | Observability | Every turn has a correlation ID across API, graph nodes, MCP calls and model calls; traces keep prompt version, model, tokens, latency, tool results, decision and fallback path |
| NFR-11 | Auditability | Audit rows are append-only (no UPDATE or DELETE grants) and retained for the length of the practical |
| NFR-12 | Maintainability | Policy thresholds, model names and prompts live in config or versioned files; changing any of them requires the eval suite to pass |
| NFR-13 | Portability | `docker compose up` starts Postgres, Qdrant, the three MCP servers, API and UI; Ollama runs on the host |

## 8. Data model

Postgres tables (LangGraph checkpoint tables are created by the checkpointer itself).

| Table | Key fields | Owned by |
| --- | --- | --- |
| `customers` | id, name, city, phone (redacted in outputs), created\_at | orders-mcp |
| `orders` | id, customer\_id, restaurant, status, total, promised\_at, picked\_at, delivered\_at | orders-mcp |
| `order_items` | id, order\_id, name, qty, unit\_price | orders-mcp |
| `payments` | id, order\_id, method\_masked, amount, status | payments-mcp |
| `cases` | id, customer\_id, order\_id, intent, status (open, awaiting\_confirmation, awaiting\_approval, resolved, handed\_off, expired), thread\_id, created\_at | API |
| `decisions` | id, case\_id, eligible, amount, method (refund, wallet\_credit, none), reason\_codes\[\], requires\_approval, policy\_clause\_id, policy\_version, status (proposed, confirmed, approved, rejected, executed) | API (rules engine) |
| `approvals` | id, decision\_id, approver\_id, action (approve, reduce, reject), final\_amount, reason, decided\_at | API (console) |
| `refunds` | id, decision\_id, idempotency\_key UNIQUE, amount, status, created\_at | payments-mcp |
| `wallet_credits` | id, decision\_id, idempotency\_key UNIQUE, amount | payments-mcp |
| `tickets` | id, case\_id UNIQUE, category, summary, status | support-mcp |
| `audit_log` | id, correlation\_id, actor (customer, agent, system), tool\_or\_event, input\_summary, result\_code, latency\_ms, ts | all (append-only) |

Qdrant collection `refund_policy`: one point per policy clause, with payload `{clause_id, title, city, version, status, effective_from, intents[]}`. Chunk by clause, not by token count.

## 9. API endpoints (FastAPI)

All endpoints need a bearer JWT. Customer endpoints check `role=customer`; console endpoints check `role=agent`.

| Method | Path | Who | Purpose |
| --- | --- | --- | --- |
| POST | `/chat/sessions` | Customer | Start a session; returns `session_id` (= LangGraph `thread_id`) |
| POST | `/chat/sessions/{id}/messages` | Customer | Send a message; response is an SSE stream of `token`, `status`, `confirmation_request` and `done` events |
| POST | `/chat/sessions/{id}/confirm` | Customer | Confirm or decline a proposed refund (`{decision_id, accept}`) and resume the graph |
| GET | `/chat/sessions/{id}` | Customer | Session history and current case status |
| GET | `/console/approvals?status=pending` | Agent | Approval queue, oldest first, with time left on the SLA |
| GET | `/console/cases/{case_id}` | Agent | Full case card: transcript, facts, policy clause, decision, audit trail |
| POST | `/console/approvals/{decision_id}` | Agent | `{action: approve, reduce or reject, final_amount?, reason}` → resumes the graph with `Command(resume=...)` |
| POST | `/console/cases/{case_id}/takeover` | Agent | Agent takes over the chat; the bot stops replying |
| GET | `/health` | Any | Liveness, plus reachability of Ollama, Qdrant, Postgres and each MCP server |
| GET | `/metrics` | Ops | Auto-resolution rate, refunds by status, fallback rate, P95 by step |

## 10. Error handling and fallbacks

The fallback depends on what failed and how risky the step is. A retry is only one option.

| Failure | Detected by | Behaviour | Customer sees |
| --- | --- | --- | --- |
| Ollama timeout or 5xx (read-only step) | Client timeout | Retry once with backoff; then circuit breaker | "Give me a moment…", then handoff if still failing |
| Invalid tool arguments or JSON | Pydantic validation | One repair attempt with the validation error fed back; then handoff | Nothing unusual, or a handoff |
| MCP read tool timeout | 3 s timeout | Retry once | Same as above |
| MCP action tool timeout | 5 s timeout | **No blind retry.** Query refund status by idempotency key, then act on the true state | "Checking your refund status…" |
| Policy not found or low retrieval score | Score below threshold | Don't answer from model memory; offer a human | "I'll connect you with a specialist for this one." |
| Order not found or not the caller's | `NOT_FOUND` | Ask the customer to pick from their recent orders | List of their recent orders |
| Rules engine says not eligible | Decision | Explain the reason code with the cited clause; offer a ticket | Clear reason + option to escalate |
| Guard blocks the input | Guard node | Polite refusal, no tool calls, audit row written | "I can only help with QuickBite orders." |
| Approval SLA expires | Scheduler | Create a ticket, set the case to `expired`, notify the customer | "Our team will reach you within 24 h." |
| Graph hits the recursion limit or an agent exceeds 3 tool calls | Recursion limit / counter | Stop and hand off | Handoff message |

## 11. Evaluation plan and quality gates

The eval suite runs on every change to a prompt, model, policy doc, rule or tool schema, and compares the result with the last passing baseline.

**Golden set: 50 scripted conversations in `evals/golden.jsonl`**

| Category | Cases | What each case asserts |
| --- | --- | --- |
| Happy path (J1, J2) | 15 | Intent, tools called, decision fields, refund executed once |
| Approval path (J3) | 8 | Interrupt raised; resume with approve, reduce or reject produces the right final state |
| Not eligible (late claim, over monthly limit) | 7 | No action tool called; reason cited |
| Ambiguous order | 5 | Asks which order; never guesses |
| Safety (injection, other customer, out of scope) | 10 | Blocked before any tool call |
| Failure injection (Ollama down, MCP timeout) | 5 | Correct fallback; no duplicate refund |

**Metrics and gates**

| Metric | How measured | Gate |
| --- | --- | --- |
| Wrong refund (amount or eligibility) | Decision vs expected | Hard: 0 |
| Duplicate refund | Rows per idempotency key | Hard: 0 |
| Safety case leaked to a tool | Tool-call log | Hard: 0 |
| Intent accuracy | Exact match | Soft: ≥ 90% |
| Tool-selection accuracy | Expected vs actual tool sequence | Soft: ≥ 95% |
| Policy Recall@3 | Expected clause in top 3 | Soft: ≥ 95% |
| Groundedness of policy answers | LLM judge, spot-checked by you on 10 cases | Soft: ≥ 90% |
| P95 turn latency | Trace timings | Soft: no more than 15% worse than baseline |

The rules engine also gets plain pytest unit tests (target 100% branch coverage), because deterministic code should be tested deterministically.

## 12. Repository layout and local setup

```text
quickbite-copilot/
├── docker-compose.yml          # postgres, qdrant, 3 MCP servers, api, ui (Ollama on host)
├── config/
│   ├── models.yaml             # model per role, timeouts
│   └── policy_rules.yaml       # ₹200 limit, 24 h window, 2 per 30 days
├── mcp_servers/
│   ├── orders/server.py        # FastMCP: list_recent_orders, get_order, get_delivery_timeline
│   ├── payments/server.py      # FastMCP: get_payment, get_refund_history, issue_refund, issue_wallet_credit
│   └── support/server.py       # FastMCP: create_ticket, policy:// resource
├── app/
│   ├── api/                    # FastAPI routers: chat, console, health, metrics
│   ├── graph/
│   │   ├── state.py            # CaseState TypedDict
│   │   ├── nodes/              # guard, classify, investigator, retrieve_policy, rules, resolver, respond, handoff
│   │   └── build.py            # StateGraph wiring + Postgres checkpointer
│   ├── rules/engine.py         # pure functions, no LLM
│   ├── rag/ingest.py, retrieve.py
│   ├── mcp_client.py           # per-session MultiServerMCPClient with the user's JWT
│   ├── prompts/                # versioned prompt files (v1, v2…)
│   └── observability.py        # correlation IDs, tracing, audit writer
├── ui/                         # React + Vite: /chat and /console
├── data/seed.py, policies/*.md # synthetic customers, orders, policy clauses
├── evals/golden.jsonl, run_evals.py, baseline.json
└── tests/                      # rules unit tests, MCP contract tests, graph tests
```

**Local setup, in order**

1. Install Ollama and pull the three models from section 5.
2. `docker compose up -d postgres qdrant`, then `python data/seed.py` and `python -m app.rag.ingest`.
3. Start the MCP servers and check each with the MCP Inspector.
4. `uvicorn app.api.main:app`, then `npm run dev` in `ui/`.
5. `python evals/run_evals.py` and commit `baseline.json` once all hard gates pass.
