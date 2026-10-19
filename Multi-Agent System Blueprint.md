Reference architecture · production

# Multi-Agent System Blueprint

How to build a production system where a supervisor agent runs a team of specialist agents, and each specialist has its own skills and tools. It uses two projects as references: **marketingskills**, which supplies the skills and tools, and **agency-agents**, which supplies the agent roster and the orchestrator prompt.

1 supervisor, in the main conversation workers one level deep every outside action goes through one tool gateway limits enforced in code, not prompts

01 · The architecture

## Layers and how they talk

Blue boxes are the two places the system's judgement and safety concentrate: the supervisor decides what happens, and the tool gateway decides what is allowed to touch the outside world. The amber gate stops anything outward-facing until a person approves it. Workers never talk to each other directly; they get tasks from the queue and return summaries to the supervisor.

02 · Lifecycle

## How one request moves through it

Example request: *"Our pricing page traffic dropped 30%. Find out why and draft fixes."*

1. **Arrives.** The gateway checks who is asking and which workspace (tenant) they belong to, applies rate limits, and strips personal data that shouldn't reach the model.
2. **Gets routed.** The router reads the request and picks agents and skills. It matches against skill descriptions and a search index. If confidence is low, it asks the user instead of guessing. This closes the "no skill found" gap you saw in marketingskills.
3. **Gets planned.** The supervisor (a strong model) writes a plan: SEO agent checks rankings, Content agent reviews the page copy, Verifier checks both. The plan is saved as tasks in the workflow engine, so a crash doesn't lose progress.
4. **Fans out.** The queue sends tasks to workers in parallel. Each worker starts with a clean context holding only its persona, the shared product context, and its own skills.
5. **Workers loop.** Each worker runs think, call a tool, observe. Every tool call passes the tool gateway, which checks the agent's allowlist, adds credentials the model never sees, and blocks writes in dry-run mode.
6. **Gets verified.** Output is checked against a schema, then the Verifier agent checks facts. A failure goes back to the same worker with the reason. After 3 failed attempts, the task escalates to a person.
7. **Waits for approval.** Anything outward-facing, such as publishing a page, sending email or changing ad spend, stops at the approval gate.
8. **Gets merged.** The supervisor combines the workers' summaries into one answer. Every step has already been traced with tokens, cost and latency.

03 · Mapping

## What each project gives you, and what you still build

| Layer | marketingskills | agency-agents | You build |
| --- | --- | --- | --- |
| Agent definitions | None (single-agent design) | Persona files, e.g. `marketing-seo-specialist.md` | Agent card: model, skills, tools, limits |
| Orchestrator | None | `agents-orchestrator.md`, NEXUS playbooks, handoff templates | Workflow engine that actually enforces the plan |
| Skills | 50 `SKILL.md` files with references, loaded on demand | None | Registry with versions and a search index |
| Shared context | `.agents/product-marketing.md` | None | Per-tenant memory store |
| Tools | 67 CLIs, 104 integration guides, MCP list | None | Tool gateway, secrets vault, permissions |
| Retries / quality gates | None | "Max 3 retries", Dev↔QA loop, written as prompt rules | The same rules, enforced in code |
| Evals | `evals.json` per skill, version checks in CI | Lint and test scripts for agent files | Routing evals, regression suite, live monitoring |
| Observability | None | None | Tracing, cost, audit log |

**The key difference from both repos.** Both projects are content: prompts that *describe* orchestration. In production, code has to *enforce* it. A prompt saying "maximum 3 retries" is a request the model may ignore. A workflow engine with `max_attempts: 3` is a guarantee.

04 · Agent card

## One file per agent ties it all together

This is the production version of an agency-agents persona. It points to the persona for *who* the agent is, to marketingskills for *what it knows*, and to the tool gateway for *what it may touch*.

```
name: seo-agent
persona: agency-agents/marketing/marketing-seo-specialist.md
model: claude-sonnet-5-5            # workers: mid-tier model
skills:                             # loaded on demand, pinned versions
  - seo-audit@2.x
  - schema@2.x
  - ai-seo@2.x
shared_context: tenants/{tenant}/product-marketing.md
tools:                              # enforced by the tool gateway
  - ga4.read
  - search_console.read
  - firecrawl.scrape
write_actions: none                 # read-only agent
limits:
  max_steps: 25
  max_tokens: 200000
  timeout: 10m
output_schema: schemas/seo-findings.json
```

The supervisor gets the same kind of card, with a stronger model (for example `claude-opus-5-5`), no business tools, and one special tool: `delegate(agent, task)`.

05 · Failure handling

## What breaks, and what the system does

| What fails | What happens | Owner |
| --- | --- | --- |
| Tool timeout or 5xx | Retry with backoff. An idempotency key makes sure a retried "send" doesn't send twice. | Tool gateway |
| Rate limited (429) | Wait for the time the API asks, then retry from the queue. | Tool gateway |
| Output fails schema | The validation error goes back to the same agent. Up to 3 attempts. | Workflow engine |
| Verifier rejects | Back to the worker with the reason. After 3 rejections, a person decides. | Supervisor |
| No skill matches | Search index tries a wider match. Still nothing: ask the user, or answer with a "no skill used" flag that is logged. | Router |
| Agent loops or stalls | Step or token budget runs out. The run stops and returns what it has. | Workflow engine |
| Model provider down | Switch to the fallback model or region. | Model gateway |
| Injected instructions in a fetched page | Fetched content is treated as data. Outward actions in that turn need approval. | Guardrails |
| Process crash mid-run | The engine resumes from the last saved step; finished tool calls are not repeated. | Workflow engine |

06 · Components

## Component reference and common options

| Component | Job | Common options |
| --- | --- | --- |
| API gateway | Sign-in, tenants, rate limits | Kong, Cloudflare, or your web framework |
| Agent runtime | The think, act, observe loop | Claude Agent SDK, LangGraph, OpenAI Agents SDK |
| Workflow engine | Saved state, retries, resume after crash | Temporal, Inngest, LangGraph with a checkpointer, AWS Step Functions |
| Task queue | Run workers in parallel | The workflow engine's own, or SQS / Redis |
| Skill registry | Versioned skills and search | Git repo + Postgres + pgvector |
| Tool gateway | MCP servers behind one policy layer | An MCP gateway, Composio, or a thin proxy of your own |
| Secrets | API keys that never enter a prompt | HashiCorp Vault, AWS or GCP Secret Manager |
| Model gateway | Routing, fallback, caching, cost caps | LiteLLM, Portkey, or a thin layer of your own |
| Memory | Run state, shared context, long-term recall | Postgres, Redis, pgvector |
| Observability | Traces, costs, audit trail | Langfuse, LangSmith, OpenTelemetry + Grafana |
| Evals | Catch regressions before release | `evals.json`-style suites in CI, Promptfoo, Braintrust |

07 · Design rules

## Rules that keep it safe

- Keep the hierarchy shallow: one supervisor, one level of workers. Claude Code enforces this; copy it.
- Workers return summaries, not full transcripts, so the supervisor's context stays small.
- Give each agent the fewest tools it needs. Make read and write separate tools.
- Credentials live in the gateway, never in prompts or agent files.
- Enforce retries, steps, tokens and money in code.
- Every outward action passes the approval gate until evals show it is safe to automate.
- Version skills and prompts like code. marketingskills' `check-versions.mjs` is the pattern.
- Treat web pages and tool output as data, never as instructions.
- Use a strong model for the supervisor and cheaper ones for workers.

08 · Build order

## Build it in this order

1. **One agent.** Skills, read-only tools, tracing. This is marketingskills as it is today.
2. **Tool gateway.** Allowlists, secrets vault, dry-run mode.
3. **Team.** Supervisor plus 2–3 workers in parallel, plus a Verifier.
4. **Durable engine.** Move orchestration into the workflow engine, with retries and budgets in code.
5. **Write actions.** Add the approval gate, then allow publishing and sending.
6. **Scale.** Evals in CI, routing evals, cost dashboards, multiple tenants.