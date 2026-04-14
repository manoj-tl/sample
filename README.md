# DevOps Chatbot — Performance & Quality Improvement Plan

> **Created:** 2026-04-14
> **Status:** Proposed
> **Scope:** Speed, result quality, reliability, and architectural improvements

---

## Table of Contents

- [Summary](#summary)
- [P0 — Critical Speed Fixes](#p0--critical-speed-fixes)
  - [1. MCP Server Startup Blocking Every Session](#1-mcp-server-startup-blocking-every-session)
  - [2. Synchronous DynamoDB Writes Block Every Turn](#2-synchronous-dynamodb-writes-block-every-turn)
  - [3. System Prompt Token Bloat](#3-system-prompt-token-bloat)
- [P1 — High-Impact Quality & Reliability](#p1--high-impact-quality--reliability)
  - [4. Router + Specialist Agent Architecture](#4-router--specialist-agent-architecture)
  - [5. Silent MCP Server Failures](#5-silent-mcp-server-failures)
  - [6. Knowledge Base Retrieval Tuning](#6-knowledge-base-retrieval-tuning)
- [P2 — Medium-Impact Improvements](#p2--medium-impact-improvements)
  - [7. Improve Tool Descriptions](#7-improve-tool-descriptions)
  - [8. Explicit Model Parameters](#8-explicit-model-parameters)
  - [9. Rate Limiting & Cost Controls](#9-rate-limiting--cost-controls)
  - [10. Structured Error Handling](#10-structured-error-handling)
  - [11. DynamoDB Pagination](#11-dynamodb-pagination)
- [P3 — Low-Impact Cleanup](#p3--low-impact-cleanup)
  - [12. KB Index Blocks First Tool Call](#12-kb-index-blocks-first-tool-call)
  - [13. AWS Boto3 Client Caching](#13-aws-boto3-client-caching)
  - [14. Global State Refactor](#14-global-state-refactor)
- [Priority Roadmap](#priority-roadmap)

---

## Summary

This document captures all identified performance bottlenecks, quality issues, and reliability gaps in the DevOps Chatbot. Fixes are organized by priority (P0–P3) based on user-facing impact.

**Key metrics to improve:**

| Metric | Current | Target |
|--------|---------|--------|
| Session startup time | 5–15s | < 2s |
| Per-turn DynamoDB overhead | 500ms–2s | < 50ms (async) |
| System prompt tokens | ~2,200 | ~500 |
| Tool count per agent | 24+ (scaling to 40+) | 5–10 per specialist |
| Tool selection accuracy | Degraded at 24+ tools | Improved with fewer tools |

---

## P0 — Critical Speed Fixes

### 1. MCP Server Startup Blocking Every Session

**File:** `chatbot/src/devops_chatbot/strands_agent.py` (lines 66–76)

**Problem:**
All 3 MCP servers (knowledge-base, aws-devops, terraform-helper) are spawned as child processes via stdio on every `on_chat_start` call. Each spawn waits for the subprocess to be ready. This adds **5–15 seconds** before the user can interact.

**Root Cause:** No connection pooling or reuse of MCP client instances across sessions.

**Fix:** Pool and reuse MCP client connections at the module level with a TTL-based cache.

```python
import time
from threading import Lock

_mcp_pool: dict[str, tuple[MCPClient, float]] = {}
_pool_lock = Lock()
_POOL_TTL = 300  # 5 minutes

def get_or_create_mcp_client(name: str, params) -> MCPClient:
    with _pool_lock:
        if name in _mcp_pool:
            client, created_at = _mcp_pool[name]
            if time.time() - created_at < _POOL_TTL:
                return client
        client = MCPClient(lambda p=params: stdio_client(p), prefix=name)
        _mcp_pool[name] = (client, time.time())
        return client
```

**Expected Impact:** Session startup drops from 5–15s to < 2s.

**Effort:** 1 day

---

### 2. Synchronous DynamoDB Writes Block Every Turn

**File:** `chatbot/src/devops_chatbot/app.py` (lines 209–235)

**Problem:**
Each message triggers 3 synchronous DynamoDB calls in sequence:
1. `save_turn()` — save user message
2. `save_turn()` — save assistant response
3. `update_session_metadata()` — update title, message count

These block the message processing thread and add **500ms–2s per turn**.

**Fix:** Fire DynamoDB writes asynchronously. The user does not need to wait for persistence to complete before seeing the response.

```python
async def on_message(message: cl.Message):
    # Save user message in background
    asyncio.create_task(asyncio.to_thread(
        chat_history.save_turn, session_id, turn_index, "user", message.content
    ))

    # Run agent (this is the main blocking call — expected)
    result = agent(message.content)
    response_text = str(result)

    # Send response immediately
    await cl.Message(content=response_text).send()

    # Persist assistant response and metadata in background
    asyncio.create_task(asyncio.to_thread(
        chat_history.save_turn, session_id, turn_index + 1, "assistant", response_text
    ))
    asyncio.create_task(asyncio.to_thread(
        chat_history.update_session_metadata, user_id, session_id, title, msg_count
    ))
```

**Expected Impact:** Eliminates 500ms–2s per turn from the user's perceived latency.

**Effort:** 1 day

---

### 3. System Prompt Token Bloat

**File:** `chatbot/src/devops_chatbot/system_prompt.txt`

**Problem:**
The system prompt is 158 lines (~2,200 tokens) with:
- "NEVER discuss chatbot internals" repeated 3 times (lines 16, 30, 34, 152)
- A full "Available Tools" section (lines 36–66) that duplicates tool schemas
- 80 lines of prescriptive dos/don'ts in "Important Rules"
- A verbose escalation flowchart (lines 100–124)

This is added to **every turn**, costing ~44K tokens on a 20-turn conversation.

**Fix:**
1. Trim to ~500 tokens with core scope, boundary cases, and 2–3 few-shot examples
2. Move tool guidance into individual tool descriptions (where the agent actually reads them)
3. Remove all duplication
4. Rely on Bedrock Guardrails for boundary enforcement instead of prompt-based restrictions

**Before (abbreviated):**
```
You are a DevOps Infrastructure Assistant...
NEVER discuss chatbot internals...
Available Tools:
  - search_confluence_knowledge: ...
  - search_codebase: ...
  [60 more lines of tool docs]
Important Rules:
  - NEVER skip KB search...
  - NEVER discuss chatbot internals...  [again]
  [50 more lines]
```

**After (target):**
```
You are a DevOps Infrastructure Assistant for [Company]. You help developers with:
- Infrastructure changes (Terraform, IAM, Secrets Manager, Lambda, EventBridge)
- CI/CD pipelines and deployment processes
- Internal runbooks and operational procedures

Always search the knowledge base before answering. Provide exact file paths,
diff blocks, and git commands when recommending changes. Never execute
infrastructure changes directly — generate code for human review.

If you cannot answer from available tools, suggest creating a Jira ticket
for the DevOps team.
```

**Expected Impact:** ~30% reduction in token cost per conversation. Faster inference.

**Effort:** 0.5 day

---

## P1 — High-Impact Quality & Reliability

### 4. Router + Specialist Agent Architecture

**Files:** `chatbot/src/devops_chatbot/strands_agent.py`, `docs/architecture/multi-agent-design.md`

**Problem:**
The current design loads **24+ tools** into a single agent (scaling to 40+ when Bitbucket/Confluence are enabled). Research shows tool selection accuracy drops ~50% going from 10 to 40 tools. The agent must reason over all tool schemas every turn, increasing latency and cost.

**Fix:** Implement the Router + Specialist pattern already documented in `docs/architecture/multi-agent-design.md`:

```
Router Agent (no MCP tools, only agent-dispatch tools)
├── AWS Specialist Agent
│   └── Tools: describe_iam_role, describe_secret, describe_lambda,
│              describe_eventbridge, generate_iam_snippet, generate_secrets_snippet
├── Terraform Specialist Agent
│   └── Tools: generate_terraform_snippet, explain_terraform_code,
│              validate_terraform_syntax, get_terraform_workflow_guide
├── Knowledge Specialist Agent
│   └── Tools: search_confluence_knowledge, search_codebase, search_jira,
│              search_knowledge_base, get_runbook, list_runbooks
└── (Future: Jira, Bitbucket, Confluence specialists)
```

**Benefits:**
- Each specialist sees only 3–7 tools (better selection accuracy)
- Router adds minimal overhead (just classification, no tool schemas)
- Specialists can have tailored system prompts
- Easier to add new domains without impacting existing ones

**Expected Impact:** ~50% improvement in tool selection accuracy. Lower per-turn token cost.

**Effort:** 2–3 days

---

### 5. Silent MCP Server Failures

**File:** `chatbot/src/devops_chatbot/strands_agent.py` (lines 69–76)

**Problem:**
If any MCP server fails to start, it is logged as a warning and silently skipped:
```python
except Exception as e:
    logger.warning("Could not register MCP server %s: %s", name, e)
```
The agent still believes those tools are available and will attempt to call them, resulting in runtime failures with no useful error message to the user.

**Fix:**
1. Retry with exponential backoff (3 attempts: 1s, 2s, 4s)
2. Track which servers failed to start
3. Inject a system message telling the agent which tools are unavailable

```python
MAX_RETRIES = 3
BACKOFF_BASE = 1  # seconds

failed_servers: list[str] = []

for name, params in mcp_servers.items():
    for attempt in range(MAX_RETRIES):
        try:
            client = MCPClient(lambda p=params: stdio_client(p), prefix=name)
            clients.append(client)
            break
        except Exception as e:
            if attempt < MAX_RETRIES - 1:
                await asyncio.sleep(BACKOFF_BASE * (2 ** attempt))
            else:
                logger.error("MCP server %s failed after %d retries: %s", name, MAX_RETRIES, e)
                failed_servers.append(name)

if failed_servers:
    system_msg = f"WARNING: The following tool servers are unavailable: {', '.join(failed_servers)}. Do not attempt to use tools from these servers."
    # Prepend to agent system prompt or inject as first message
```

**Expected Impact:** Eliminates confusing runtime tool failures. Agent routes around unavailable tools.

**Effort:** 1 day

---

### 6. Knowledge Base Retrieval Tuning

**File:** `chatbot/src/devops_chatbot/kb_tools.py` (lines 38–86)

**Problem:**
- Confluence and Jira KBs return only 5 results — too few for broad queries
- Codebase KB returns 7 results — slightly better but still hardcoded
- Score threshold of `0.3` is only applied to the codebase tool, not the others
- No way to tune without code changes

**Fix:**
1. Make `max_results` and `score_threshold` configurable via environment variables
2. Apply score filtering consistently across all 3 KB tools
3. Increase defaults to 10 results
4. Log retrieval stats for ongoing tuning

```python
# Environment-driven configuration
CONFLUENCE_KB_MAX_RESULTS = int(os.getenv("CONFLUENCE_KB_MAX_RESULTS", "10"))
CONFLUENCE_KB_SCORE_THRESHOLD = float(os.getenv("CONFLUENCE_KB_SCORE_THRESHOLD", "0.3"))
CODEBASE_KB_MAX_RESULTS = int(os.getenv("CODEBASE_KB_MAX_RESULTS", "10"))
CODEBASE_KB_SCORE_THRESHOLD = float(os.getenv("CODEBASE_KB_SCORE_THRESHOLD", "0.3"))
JIRA_KB_MAX_RESULTS = int(os.getenv("JIRA_KB_MAX_RESULTS", "10"))
JIRA_KB_SCORE_THRESHOLD = float(os.getenv("JIRA_KB_SCORE_THRESHOLD", "0.3"))

def _retrieve(kb_id: str, query: str, max_results: int = 10, score_threshold: float = 0.3):
    response = client.retrieve(
        knowledgeBaseId=kb_id,
        retrievalQuery={"text": query},
        retrievalConfiguration={
            "vectorSearchConfiguration": {"numberOfResults": max_results}
        },
    )
    results = response.get("retrievalResults", [])
    filtered = [r for r in results if r.get("score", 0) >= score_threshold]
    logger.info("KB %s: %d candidates, %d after filtering (threshold=%.2f)",
                kb_id, len(results), len(filtered), score_threshold)
    return filtered
```

**Expected Impact:** Better answer relevance. Operators can tune retrieval without code deploys.

**Effort:** 0.5 day

---

## P2 — Medium-Impact Improvements

### 7. Improve Tool Descriptions

**File:** `chatbot/src/devops_chatbot/kb_tools.py` (lines 89–177)

**Problem:**
Tool descriptions conflate Bedrock KB semantic search with live service search. The agent does not know when to use KB search vs. live MCP search, leading to tool misselection.

**Fix:** Make descriptions explicit about data source, freshness, limitations, and ideal use cases.

**Before:**
```python
def search_confluence_knowledge(query: str) -> str:
    """Semantic search of Confluence institutional knowledge.
    Searches processes, policies, runbooks, and team norms stored in Confluence.
    Use this FIRST for any "how do I..." or process question."""
```

**After:**
```python
def search_confluence_knowledge(query: str) -> str:
    """Semantic vector search over a SNAPSHOT of Confluence pages (synced nightly to Bedrock KB).
    Returns top results ranked by relevance score.
    Best for: process documentation, policies, runbooks, team norms, onboarding guides.
    NOT for: real-time page content, recent edits (< 24h), page comments, or attachments.
    Query tips: use natural language questions, not keyword lists."""
```

Apply the same pattern to `search_codebase` and `search_jira`.

**Expected Impact:** Better tool selection by the agent.

**Effort:** 0.5 day

---

### 8. Explicit Model Parameters

**File:** `chatbot/src/devops_chatbot/strands_agent.py` (lines 94–147)

**Problem:**
No `temperature`, `max_tokens`, or `top_p` are set. Defaults may vary by model and are not reproducible. No cost control on response length.

**Fix:**
```python
AGENT_TEMPERATURE = float(os.getenv("AGENT_TEMPERATURE", "0.3"))
AGENT_MAX_TOKENS = int(os.getenv("AGENT_MAX_TOKENS", "2048"))

model = BedrockModel(
    model_id=model_id,
    region_name=region,
    temperature=AGENT_TEMPERATURE,
    max_tokens=AGENT_MAX_TOKENS,
    **guardrail_kwargs,
)
```

- `temperature=0.3` — lower for deterministic DevOps answers
- `max_tokens=2048` — caps response length and cost
- Both configurable via environment variables

**Expected Impact:** Reproducible behavior. Cost control.

**Effort:** 0.5 day

---

### 9. Rate Limiting & Cost Controls

**Files:** `chatbot/src/devops_chatbot/app.py`

**Problem:**
No controls on conversation speed or depth. A user can send unlimited messages, each spawning multiple LLM calls and tool invocations. No circuit breaker or quota system.

**Fix:**
1. Add per-user rate limiting (e.g., 30 messages/minute)
2. Set max conversation depth (e.g., 100 turns per session)
3. Track cumulative token usage per session in DynamoDB

```python
from collections import defaultdict
import time

_rate_limits: dict[str, list[float]] = defaultdict(list)
RATE_LIMIT = 30  # messages per minute
MAX_TURNS = 100

async def check_rate_limit(user_id: str) -> bool:
    now = time.time()
    timestamps = _rate_limits[user_id]
    # Remove timestamps older than 60 seconds
    _rate_limits[user_id] = [t for t in timestamps if now - t < 60]
    if len(_rate_limits[user_id]) >= RATE_LIMIT:
        return False
    _rate_limits[user_id].append(now)
    return True
```

**Expected Impact:** Prevents abuse and runaway costs.

**Effort:** 1 day

---

### 10. Structured Error Handling

**File:** `chatbot/src/devops_chatbot/app.py` (lines 239–247)

**Problem:**
On exception, the code assumes `agent.messages` is a list of dicts and pops the last entry. No distinction between transient errors (throttling, timeout) vs. permanent ones. No retry mechanism.

**Fix:**
```python
from botocore.exceptions import ClientError

try:
    result = agent(message.content)
except ClientError as e:
    error_code = e.response["Error"]["Code"]
    if error_code in ("ThrottlingException", "TooManyRequestsException"):
        logger.warning("Throttled by AWS: %s", e)
        await cl.Message(
            content="The service is temporarily busy. Please wait a moment and try again."
        ).send()
        return
    logger.error("AWS error: %s", e, exc_info=True)
    await cl.Message(content="An AWS service error occurred. Please try again.").send()
    return
except TimeoutError:
    logger.warning("Agent timed out")
    await cl.Message(content="The request timed out. Please try a simpler question.").send()
    return
except Exception as e:
    logger.error("Unexpected error in agent", exc_info=True)
    await cl.Message(
        content="An unexpected error occurred. Please try again or contact the DevOps team."
    ).send()
    return
```

**Expected Impact:** Better user experience on failures. Easier debugging.

**Effort:** 0.5 day

---

### 11. DynamoDB Pagination

**File:** `chatbot/src/devops_chatbot/chat_history.py` (lines 111–119, 149–158)

**Problem:**
`load_history()` and `list_sessions()` only read the first page of DynamoDB results (1MB limit). Long conversations or active users will silently lose data.

**Fix:**
```python
def load_history(session_id: str) -> list[dict]:
    items = []
    last_key = None
    while True:
        kwargs = {
            "KeyConditionExpression": Key("session_id").eq(session_id),
            "ScanIndexForward": True,
        }
        if last_key:
            kwargs["ExclusiveStartKey"] = last_key
        response = _messages_table().query(**kwargs)
        items.extend(response.get("Items", []))
        last_key = response.get("LastEvaluatedKey")
        if not last_key:
            break
    return items
```

Apply the same pattern to `list_sessions()`.

**Expected Impact:** Prevents silent data loss for active users.

**Effort:** 0.5 day

---

## P3 — Low-Impact Cleanup

### 12. KB Index Blocks First Tool Call

**File:** Knowledge Base MCP server (`server.py` lines 34–44)

**Problem:**
The knowledge base index is lazily built on the first tool call, causing a 5–10 second hang on the agent's first query.

**Fix:** Either pre-build the index at server startup, or cache the built index to disk:
```python
INDEX_CACHE_PATH = Path("/tmp/kb_index.pkl")

def get_index() -> KnowledgeBaseIndex:
    global _index
    if _index is None:
        if INDEX_CACHE_PATH.exists():
            _index = pickle.loads(INDEX_CACHE_PATH.read_bytes())
        else:
            _index = KnowledgeBaseIndex(config.knowledge_base_path)
            INDEX_CACHE_PATH.write_bytes(pickle.dumps(_index))
    return _index
```

**Effort:** 0.5 day

---

### 13. AWS Boto3 Client Caching

**File:** `servers/aws_devops/src/devops_mcp_aws/server.py` (lines 27–37)

**Problem:**
Each tool call creates a new `session.client("iam")` even though the session itself is cached. Client creation has ~100–200ms overhead.

**Fix:**
```python
_clients: dict[str, Any] = {}

def get_client(service_name: str):
    if service_name not in _clients:
        _clients[service_name] = get_session().client(service_name)
    return _clients[service_name]
```

**Effort:** 0.5 day

---

### 14. Global State Refactor

**Files:** `kb_tools.py`, `aws_devops/server.py`, `chat_history.py`

**Problem:**
Module-level globals (`_kb_client`, `_session`, `_dynamodb`, etc.) are not thread-safe and are difficult to test.

**Fix:** Refactor to a config class with lazy-initialized properties:
```python
class KBConfig:
    def __init__(self):
        self._client = None
        self.confluence_kb_id = os.getenv("CONFLUENCE_KB_ID", "")
        self.codebase_kb_id = os.getenv("CODEBASE_KB_ID", "")
        self.jira_kb_id = os.getenv("JIRA_KB_ID", "")

    @property
    def client(self):
        if self._client is None:
            self._client = boto3.client("bedrock-agent-runtime")
        return self._client

kb_config = KBConfig()
```

**Effort:** 1 day

---

## Priority Roadmap

| Priority | Fix | Effort | Impact | Category |
|----------|-----|--------|--------|----------|
| **P0** | #1 MCP client pooling | 1 day | -10s per session | Speed |
| **P0** | #2 Async DynamoDB writes | 1 day | -1s per turn | Speed |
| **P0** | #3 Trim system prompt | 0.5 day | -30% token cost | Speed / Cost |
| **P1** | #4 Router + Specialist agents | 2–3 days | +50% tool accuracy | Quality |
| **P1** | #5 MCP retry + health reporting | 1 day | Eliminates silent failures | Reliability |
| **P1** | #6 KB retrieval tuning | 0.5 day | Better answer relevance | Quality |
| **P2** | #7 Improve tool descriptions | 0.5 day | Better tool selection | Quality |
| **P2** | #8 Explicit model parameters | 0.5 day | Reproducibility + cost | Quality |
| **P2** | #9 Rate limiting | 1 day | Abuse prevention | Reliability |
| **P2** | #10 Structured error handling | 0.5 day | Better UX on failures | Reliability |
| **P2** | #11 DynamoDB pagination | 0.5 day | Prevents data loss | Reliability |
| **P3** | #12 KB index caching | 0.5 day | Faster first query | Speed |
| **P3** | #13 Boto3 client caching | 0.5 day | -100ms per AWS call | Speed |
| **P3** | #14 Global state refactor | 1 day | Testability | Quality |

**Total estimated effort:** ~11–12 days

### Suggested Phases

- **Phase 1 (Week 1):** P0 fixes — MCP pooling, async DynamoDB, system prompt trim
- **Phase 2 (Week 2):** P1 fixes — Router + Specialist architecture, MCP retries, KB tuning
- **Phase 3 (Week 3):** P2 fixes — Tool descriptions, model params, rate limiting, error handling
- **Phase 4 (Ongoing):** P3 cleanup — Index caching, client caching, global state refactor
