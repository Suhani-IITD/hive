# Codebase Review

Context: local source review + focused test run in this repo

## 1) What I validated from code

- Quickstart configures LLM provider/API keys, but does not connect Gmail OAuth integrations directly in setup flow.
  - quickstart.sh
- Gmail and email tooling depend on GOOGLE_ACCESS_TOKEN from Aden-backed credentials (google credential spec), not direct Gmail API-key auth.
  - tools/src/aden_tools/credentials/email.py
  - tools/src/aden_tools/tools/gmail_tool/gmail_tool.py
  - tools/src/aden_tools/tools/email_tool/email_tool.py
- LiteLLM provider already has explicit retry/backoff handling for 429 and empty responses (including Gemini-like empty-response cases).
  - core/framework/llm/litellm.py
- LLM key validation script currently treats HTTP 429 as a valid key for Gemini and other providers.
  - scripts/check_llm_key.py

## 2) Free-ties API testing issues 

### A) Free-tier Gemini rate limit hits early

Why this happens:
- Key validation is reachability/auth check, not sustained quota/capacity check.
- check_gemini() accepts 200 or 429 as valid key. Setup can pass even when runtime workload is too large for free tier.
  - scripts/check_llm_key.py
- Runtime retries and backoff exist, but they do not solve low quota.
  - core/framework/llm/litellm.py

Suggested fix:
- Add optional capacity-probe mode in scripts/check_llm_key.py:
  - tiny completion with strict token cap
  - parse quota/rate-limit metadata where available
  - return status tiers: valid_low_capacity, valid, invalid, inconclusive
- In quickstart.sh, if using free-tier API key and status=valid_low_capacity, show guidance:
  - recommend smaller worker model settings
  - suggest OpenRouter/Hive/other fallback for multi-node templates
  - display explicit best-effort warning for heavy agents

Feasibility: high.

### B) Gmail integration blocked for Aden app

Why this happens:
- Gmail tools require OAuth token (GOOGLE_ACCESS_TOKEN) via Aden-backed integration.
  - tools/src/aden_tools/credentials/email.py
  - tools/src/aden_tools/tools/gmail_tool/gmail_tool.py
- Error messaging is mostly connect via hive.adenhq.com, but provider-side reason classes are not surfaced clearly (admin block, revoked consent, scope mismatch).
  - tools/src/aden_tools/tools/gmail_tool/gmail_tool.py
  - core/framework/credentials/aden/client.py

Suggested fix:
- Improve actionable auth diagnostics:
  - preserve requires_reauthorization and reauthorization_url end-to-end in UI/API
  - classify failures into reauthorize_required, access_blocked_by_admin, invalid_scope, token_expired
  - show next-step matrix in credential setup modal/CLI output
- Add a Gmail connectivity test command:
  - token exists
  - Gmail profile endpoint reachable
  - expected scope available

Feasibility: medium.

## 3) Additional issues found

### Issue 1: Duplicate tool registration in verified path

Evidence:
- _register_verified() registers many credentialed tools twice.
  - tools/src/aden_tools/tools/__init__.py
- Runtime check:
  - registered_count 281
  - unique_count 281
  - duplicates 0
- But FastMCP emits many Tool already exists warnings during registration.

Impact:
- noisy logs
- unnecessary startup work
- harder production debugging

Fix plan:
- remove duplicated second credentialed registration block in _register_verified()
- add regression test asserting no duplicate-registration warnings on register_all_tools()

Feasibility: high.

### Issue 2: Template tests are environment-sensitive to ADEN_API_KEY

Reproduced command:
- uv run pytest examples/templates/email_reply_agent/tests/test_email_reply_agent.py -q

Observed result:
- 1 failed, 9 passed, 3 errors
- 3 errors come from AgentRunner.load() running preload credential validation when ADEN_API_KEY is present but Gmail integration is not connected.
  - core/framework/runner/runner.py
  - core/framework/runner/preload_validation.py
  - examples/templates/email_reply_agent/tests/conftest.py

Impact:
- structure-only tests fail due local credential state
- contributor onboarding friction

Fix plan:
- in fixture, call AgentRunner.load(..., skip_credential_validation=True) for structure tests
- alternatively set module-level skip_credential_validation=True for template CI context

Feasibility: high.

### Issue 3: Test/agent drift in email_reply_agent edge condition

Evidence:
- agent uses condition_expr: batch_complete == True
  - examples/templates/email_reply_agent/agent.py
- test expects older stricter expression with send_started/send_count/sent_message_ids
  - examples/templates/email_reply_agent/tests/test_email_reply_agent.py

Impact:
- failing CI/test noise
- unclear intended safety contract

Fix plan:
- choose intended behavior:
  - Option A: keep simple condition and update tests
  - Option B: restore strict condition and update node outputs/prompts
- align flowchart.json + tests after decision

Feasibility: high.

## 4) Specialized agent ideas with implementation plans

### Proposal A: Inbox SLA Triage (upgrade of email_inbox_management + email_reply_agent)

Positioning:
- it is a reliability/ops upgrade layer on top of existing email templates
- preserves "draft-only by default" behavior

Use case:
- prioritize urgent customer/vendor/investor emails against SLA windows
- create draft replies and escalation summaries only

Implementation sketch:
1. add required baseline config: VIP domains, SLA windows, escalation policy, timezone
2. classify incoming inbox threads by urgency + stakeholder class
3. apply labels/priority flags in Gmail
4. create draft responses for urgent threads only
5. generate daily SLA-risk report

Feasibility: high.

### Proposal B: Follow-up Queue Orchestrator (composite: email_reply_agent + sdr_agent + sheets logging)

Positioning:
- composite pipeline template
- optimized for founders/operators running weekly follow-up loops

Use case:
- detect stale conversations and unanswered threads
- maintain actionable follow-up queue with owner/priority/next-date
- draft personalized follow-ups in batches

Implementation sketch:
1. required baseline config: follow-up cadence, stale threshold, owner map, do-not-contact rules
2. pull and rank follow-up candidates from Gmail
3. sync queue state to Sheets (or local datastore fallback)
4. generate draft follow-ups with a weekly digest

Feasibility: high.

### Proposal C: Scheduling Concierge v2 (upgrade of meeting_scheduler + email parsing)

Positioning:
- extends existing meeting_scheduler template into email-first workflow

Use case:
- convert scheduling emails into proposed slots, confirmations, and briefing packet links

Implementation sketch:
1. required baseline config: meeting duration defaults, booking windows, preferred hours, attendee roles
2. parse scheduling intent from inbound email threads
3. check calendar availability and propose options
4. create event after approval and generate structured briefing notes

Feasibility: medium-high.

### Proposal D: Limit-Aware Research Mode (upgrade of deep_research_agent + tech_news_reporter)

Positioning:
- framework/template optimization mode rather than a separate disconnected template

Use case:
- keep output quality stable under low-quota/free-tier model limits

Implementation sketch:
1. preflight budget node estimates token/tool budget before heavy steps
2. switch strategy by budget tier (light/standard/deep)
3. enforce chunking and citation-first extraction paths
4. fallback on repeated 429/empty-response signals with graceful degradation

Feasibility: medium.

### Proposal E: Credential Health Sentinel v2 (complement to credential_tester)

Positioning:
- credential_tester is interactive testing; this adds scheduled health monitoring and proactive remediation

Use case:
- detect expired/revoked integrations before user workflows fail

Implementation sketch:
1. scheduled daily health checks per provider/account
2. classify failures (reauthorize_required, access_blocked_by_admin, invalid_scope, token_expired)
3. produce actionable reauth checklist + deep links
4. optional Slack/Telegram digest for ops owners

Feasibility: medium-high.

### Framework feature proposal: Agent Blend Pipelines with required baseline parameters

Problem:
- many workflows are a blend of existing templates, but current runs may ask repeated setup questions that cost extra turns/tokens

Proposal:
1. allow composing 2-3 templates into a pipeline with explicit handoff contract (state keys + output schema)
2. support `required_baseline_params` per entry point (must be provided once; cached per workspace/user)
3. allow policy defaults (`draft_only=true`, `max_batch_size`, `approval_required_actions`) to be enforced at runtime
4. provide fail-fast validation before first model/tool call if required params are missing

Expected impact:
- fewer unnecessary clarification turns
- lower token/tool cost
- more predictable production behavior for repeat workflows
