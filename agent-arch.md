# Scheduled Agents — Design Spec

**Status:** Draft
**Date:** 2026-04-21
**Owner:** manojn@thinklabs.io

---

## Problem

The DevOps chatbot today only runs agents in response to user chat messages. We want **autonomous scheduled agents** that run on a cron schedule without a user in the loop — e.g., "every hour, check for unassigned Jira tickets on the DEVOPS board and post a summary."

This requires:
1. A place to define and manage these agents (admin UI)
2. A scheduler to fire them on cadence
3. Persistence for agent definitions and run history
4. A way for scheduled agents to reuse the multi-agent specialist architecture (Jira/Bitbucket/Confluence/AWS/Terraform agents)

## Non-Goals

- Not implementing the first concrete scheduled agent (Jira triage) in this spec — that's a follow-up once the platform is in place.
- Not migrating existing chat flow to AgentCore (separate plan).
- Not building RBAC / role separation for admins vs users (deferred; POC reuses chat password auth).
- Not building a visual workflow builder. Agents are defined by picking a specialist + writing a prompt + setting a schedule.

## Decisions Summary

| Topic | Decision |
|---|---|
| Admin UI location | Separate web page at `/agents`, not a chat flow |
| Scheduler | EventBridge Scheduler → Lambda (one schedule per agent) |
| Storage | DynamoDB for both agent definitions and run history |
| Agent definition model | Free-form: pick a specialist + write a task prompt + set schedule |
| Admin UI stack | FastAPI + Jinja2 + HTMX + Tailwind CSS + daisyUI |
| Admin UI auth | Shared password auth with Chainlit (`AUTH_USERS`) — no role separation |
| External system auth (Jira/Bitbucket/Confluence) | Service account tokens in AWS Secrets Manager |
| Deployment | Same ECS task as Chainlit, two ports, ALB path routing |

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  User's Browser                                                   │
│  ┌──────────────────┐         ┌────────────────────────────────┐ │
│  │  Chainlit Chat    │         │  Admin UI (new)                │ │
│  │  (existing)        │         │  /agents — list, create, edit  │ │
│  │                    │         │  /agents/:id/runs — history    │ │
│  └──────────────────┘         └────────────────────────────────┘ │
└────────┬────────────────────────────────┬───────────────────────┘
         │                                │
         ▼                                ▼
┌────────────────────────┐       ┌────────────────────────┐
│  ECS Task              │       │  ECS Task (same)       │
│  Chainlit :8000        │       │  FastAPI admin :8001    │
│                        │       │  /agents CRUD + views   │
└────────────┬───────────┘       └───────────┬────────────┘
             │                               │
             └───────────────┬───────────────┘
                             │
                             ▼
                ┌────────────────────────┐
                │  DynamoDB              │
                │  • agents              │  definitions
                │  • agent_runs          │  run history
                └────────────────────────┘
                             ▲
                             │ writes run results
                             │
                ┌────────────┴───────────┐
                │  Lambda: agent-runner  │
                │  • Loads agent def     │
                │  • Instantiates        │
                │    specialist agent    │
                │  • Runs prompt         │
                │  • Writes run record   │
                └────────────┬───────────┘
                             ▲
                             │ invokes
                             │
                ┌────────────┴───────────┐
                │  EventBridge Scheduler │
                │  One schedule per      │
                │  agent (cron)          │
                └────────────────────────┘
```

### Flow: Create Agent

1. User opens `/agents` in browser → clicks "Create"
2. Fills form (name, specialist, prompt, cron, enabled) → submits
3. Admin API writes to `agents` table in DynamoDB
4. Admin API creates EventBridge schedule targeting `agent-runner` Lambda with payload `{agent_id: "..."}`

### Flow: Scheduled Run

1. EventBridge fires on cron → invokes Lambda with `{agent_id: "..."}`
2. Lambda reads agent definition from DynamoDB
3. Lambda instantiates the specialist agent (same code path as Chainlit's multi-agent router)
4. Lambda runs the agent's prompt, captures output
5. Lambda writes run record to `agent_runs` table (status, duration, output, error)

### Flow: View Run History

1. User opens `/agents/:id/runs`
2. Admin API queries `agent_runs` by `agent_id`, most recent first, paginated
3. Admin UI renders table. Click a row to expand full output.

## Data Model

### Table: `agents`

- **Partition key:** `agent_id` (string, UUID)
- **Attributes:**
  - `name` (string) — human-readable name
  - `description` (string)
  - `specialist` (string) — one of: `jira`, `bitbucket`, `confluence`, `aws`, `terraform`
  - `prompt` (string) — the task prompt given to the specialist at runtime
  - `schedule_expression` (string) — EventBridge cron/rate expression, e.g., `rate(1 hour)` or `cron(0 9 * * ? *)`
  - `schedule_arn` (string) — ARN of the EventBridge schedule (for updates/deletes)
  - `enabled` (bool) — if false, schedule is paused
  - `owner` (string) — username from auth
  - `created_at`, `updated_at` (ISO8601 strings)
  - `output_destination` (string, future) — where to send results. For POC: always `dynamodb` (just store in run history)

### Table: `agent_runs`

- **Partition key:** `agent_id` (string)
- **Sort key:** `run_id` (string, ULID so sorts chronologically descending)
- **Attributes:**
  - `started_at`, `finished_at` (ISO8601)
  - `duration_ms` (number)
  - `status` (string) — one of: `running`, `success`, `failed`, `timeout`
  - `output` (string) — the final assistant message from the specialist
  - `error` (string, nullable) — error message if failed
  - `trigger` (string) — `scheduled`, `manual`, or `test`
  - `triggered_by` (string, nullable) — username if manual/test
  - `ttl` (number) — Unix timestamp, auto-delete after 90 days

## Admin UI

### Stack

- **FastAPI** — backend, serves HTML pages and JSON endpoints
- **Jinja2** — server-side templates
- **HTMX** — swap HTML fragments for interactivity without a JS framework
- **Tailwind CSS** (CDN) — utility-first styling
- **daisyUI** (CDN) — component presets (buttons, cards, tables, modals) on top of Tailwind
- **Heroicons** — inline SVG icons
- **No build pipeline for POC** — all CDN-loaded. Add build step later if bundle size or CSP requires it.
- **Alpine.js is NOT included** initially — add later only if we hit client-state cases that daisyUI/CSS-only components can't handle.

### Pages

| Route | Purpose |
|---|---|
| `GET /agents` | List view — table of all agents with name, specialist, schedule, last run, status |
| `GET /agents/new` | Create form |
| `GET /agents/:id` | Edit form (same fields as create, prefilled) |
| `GET /agents/:id/runs` | Run history table with expandable rows |

### Form fields (create/edit)

- Name (required)
- Description (optional)
- Specialist (dropdown: Jira, Bitbucket, Confluence, AWS, Terraform)
- Task prompt (textarea, 5+ rows)
- Schedule (cron/rate expression with preset buttons: hourly, daily 9am, weekdays 9am, weekly)
- Enabled toggle
- Buttons: `[Test Run]` (synchronous, returns output to the UI), `[Save]`

### Detail page extras

- `[Run Now]` — triggers Lambda asynchronously (fire-and-forget), refreshes the run history below
- `[Disable]` / `[Enable]` — pauses/resumes EventBridge schedule
- `[Delete]` — with confirmation modal

### API endpoints

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/agents` | Create — write DDB + create EventBridge schedule |
| `PUT` | `/api/agents/:id` | Update — write DDB + update schedule |
| `DELETE` | `/api/agents/:id` | Delete — remove DDB item + delete schedule |
| `POST` | `/api/agents/:id/run` | Invoke Lambda asynchronously, return run_id |
| `POST` | `/api/agents/:id/test` | Invoke Lambda synchronously (RequestResponse), return output |
| `GET` | `/api/agents/:id/runs` | JSON list of runs (for HTMX refresh after Run Now) |

## Lambda: `agent-runner`

### Responsibilities

1. Parse event payload: `{agent_id, trigger, triggered_by}`
2. Create a `running` run record in `agent_runs` immediately (so the UI can show "in progress")
3. Load agent definition from `agents`
4. Construct the specialist agent using the same factory code as Chainlit (reuse `devops_chatbot.agents.<specialist>` module)
5. Run `agent(prompt)` — Strands handles the tool-use loop
6. Update the run record with `success` + output, or `failed` + error
7. Send structured logs to CloudWatch for debugging

### Configuration

- **Runtime:** Python 3.12
- **Memory:** 1024 MB (initial — tune based on observed usage)
- **Timeout:** 10 minutes (leaves buffer under Lambda's 15-min hard limit)
- **Concurrency:** Reserved concurrency = 5 (so runaway scheduling can't DoS Bedrock)
- **Environment variables:** Same as Chainlit task (Bedrock region, KB IDs, etc.)
- **IAM permissions:**
  - `bedrock:InvokeModel` (Claude)
  - `dynamodb:GetItem` on `agents`; `PutItem`/`UpdateItem` on `agent_runs`
  - `secretsmanager:GetSecretValue` for Jira/Bitbucket/Confluence service tokens
  - `bedrock-agent-runtime:Retrieve` for KB tools
  - CloudWatch Logs default permissions

### Packaging

- Share the specialist agent code with Chainlit. Either:
  - **Option 1 (recommended):** Package `devops_chatbot` as an installable wheel, include in both Chainlit's container and Lambda's deployment package. One source of truth.
  - **Option 2:** Lambda container image built from the same base image as Chainlit. Heavier but simpler dependency management.

## EventBridge Scheduler Integration

- **One schedule per agent.** Created/updated/deleted by the admin API using the EventBridge Scheduler SDK.
- **Schedule name:** `devops-chatbot-agent-{agent_id}` — deterministic so we can find/update it.
- **Target:** `agent-runner` Lambda, with payload `{agent_id, trigger: "scheduled"}`.
- **Flexible time window:** `OFF` (run at exact scheduled time — no jitter).
- **Pause/resume:** Use `UpdateSchedule` with `State: DISABLED`/`ENABLED` rather than delete.
- **DLQ:** Schedules configured with a dead-letter SQS queue; failed invocations land there for debugging. Admin UI surfaces "Schedule errors" badge on affected agents.

## Authentication

### Admin UI access

- Same `AUTH_USERS` env var pattern as Chainlit. No role distinction — any authenticated user can create, edit, delete agents.
- FastAPI dependency validates a session cookie set at login.
- Login page is shared with Chainlit (or duplicated — TBD during implementation).
- **Deferred:** Admin role separation (Phase 2).

### Scheduled agent → external systems

- Service account tokens (`JIRA_TOKEN`, `BITBUCKET_TOKEN`, `CONFLUENCE_TOKEN`) stored in AWS Secrets Manager.
- Lambda reads secrets at runtime via `boto3`. Cached per-invocation (not across invocations — Lambda execution contexts are reused but we don't rely on it).
- MCP servers consume tokens via environment variables or by reading from Secrets Manager directly (existing pattern).
- **Audit trail:** All actions by scheduled agents are attributed to the bot service account in external systems. The `owner` field on `agents` records which human created the agent.
- **Deferred:** Per-user OAuth via AgentCore Identity (Phase 3, after AgentCore migration).

## Deployment

### POC Deployment (Phase 1)

- **Same ECS task as Chainlit.** Two processes in one container:
  - Chainlit on port 8000
  - FastAPI admin on port 8001
  - Supervised by a process manager (e.g., `supervisord` or a multi-process entrypoint script)
- **ALB path routing:**
  - `/` and `/ws` → port 8000 (Chainlit)
  - `/agents*`, `/api/agents*`, `/static/*` → port 8001 (admin)
- **Lambda + EventBridge** deployed via Terraform alongside existing infra.

### Why not separate ECS service?

Cleaner isolation but overkill for POC. One container means:
- One deployment pipeline
- Shared code for specialist factory (no packaging problem)
- Simpler networking
Revisit if admin UI traffic ever competes with Chainlit for resources.

## Safety Considerations

1. **Rate limiting.** Admin API enforces max 50 agents per tenant to prevent accidental bill explosions.
2. **Concurrency cap.** Lambda reserved concurrency = 5. If more than 5 scheduled agents fire simultaneously, later ones queue (EventBridge handles retries).
3. **Idempotency.** `agent-runner` creates the `running` record BEFORE invoking the agent. If the Lambda retries (e.g., network glitch talking to EventBridge), we can detect by checking run_id.
4. **Read-only by default.** Current specialists are read-only (no write capabilities to Jira/Bitbucket/Confluence yet). Scheduled agents therefore cannot cause external harm at the POC stage.
5. **Delete protection.** Admin UI requires typing the agent name to confirm deletion (prevents fat-finger).
6. **Cost guardrails.** CloudWatch alarms on Bedrock token spend (configured separately). If a scheduled agent's prompt spirals (loop of tool calls), the Lambda timeout (10 min) bounds the damage.

## Observability

- **Structured logs** from Lambda: every invocation logs `agent_id`, `run_id`, `status`, `duration_ms`, `tool_call_count`. CloudWatch Logs for debugging.
- **CloudWatch metrics** on Lambda: `Invocations`, `Errors`, `Duration`, `Throttles`.
- **Admin UI "Runs" page** for product-level observability — status, output, error per run.
- **Deferred:** Dashboard with graphs (success rate, avg duration per agent) — can query DDB aggregations if we add a GSI later.

## Testing Strategy

- **Unit tests** on the Lambda handler with a mocked specialist agent (patch the `create_agent` factory to return a stub).
- **Integration test** against a dev DynamoDB table + a stubbed specialist agent that returns a fixed string.
- **End-to-end test** in a dev AWS account: create agent via admin API, verify schedule exists in EventBridge, wait for it to fire, verify run record appears in DDB.
- **UI smoke test** via Playwright: load `/agents`, create an agent, verify it appears in list, click "Test Run", verify output.

## Open Questions (to resolve during implementation)

1. **Shared login with Chainlit** — can FastAPI read the same session cookie Chainlit sets? If not, do we build a separate login page for admin, or put both behind a shared SSO later?
2. **Process supervision in ECS task** — which tool? (`supervisord`, `honcho`, shell script with `wait -n`?)
3. **Specialist agent packaging for Lambda** — wheel vs container image. Depends on current packaging of `devops_chatbot`.
4. **Cron preset UI** — how to help non-technical users write cron? Options: preset buttons, or a tiny cron builder widget. Start with presets only.

## Phases

### Phase 1: Platform (this spec)
Build the infrastructure to define, schedule, run, and view scheduled agents. Deliverables: admin UI, DynamoDB tables, Lambda, EventBridge integration, deployment.

### Phase 2: First real agent — Jira Triage
Design and implement the first concrete scheduled agent. Separate spec. Requires Jira MCP write capabilities.

### Phase 3: Notifications
Add output destinations beyond DynamoDB — Slack channels, email, Jira comments. Extends the `output_destination` field.

### Phase 4: AgentCore migration
Move the Lambda-based runner to AgentCore Runtime. The admin UI, DynamoDB tables, and EventBridge schedules remain — only the target changes from Lambda ARN to AgentCore endpoint.

### Phase 5: Role separation + per-user OAuth
Admin vs user roles. AgentCore Identity for per-user Jira/Bitbucket/Confluence auth instead of shared service accounts.

---

## Appendix: Rejected Alternatives

- **APScheduler in Chainlit process** — In-process scheduler. Rejected because ECS task restarts kill in-flight jobs and we'd need leader election for multi-task deployments.
- **CloudWatch Logs for run history** — Zero-storage approach. Rejected: slow UI (Logs Insights takes seconds), per-query cost, no live "running" status, no aggregations.
- **Chainlit chat profiles / action buttons for admin** — Keep everything in chat. Rejected: a proper admin surface is needed for tables, forms, status dashboards.
- **React SPA for admin UI** — Rejected: introduces a build toolchain and JS state management for a CRUD app where HTMX is sufficient.
- **Bootstrap / Pico.css** — Rejected in favor of Tailwind + daisyUI for modern look without custom CSS.
- **Template-only agent definitions** — Locked-down prebuilt agent types. Rejected because multi-agent specialists already encode the safety; free-form prompts reuse that work.
- **Visual workflow builder** — YAGNI for current stage; revisit if users need multi-step DAGs.
