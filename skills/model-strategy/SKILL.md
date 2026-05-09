---
name: model-strategy
description: >
  Generate a model routing strategy and configuration for an agentic company — mapping AI models
  to staffing roles based on budgets, subscriptions, business domains, and model strengths.
  Use this skill when the user says "set up model routing", "configure agent models",
  "create a model strategy", "staff my AI agents", "assign models to roles",
  "model config", "agent staffing plan", "which model for which task",
  "set up my agentic company", "configure sub-agents", or any request to decide
  which AI models should handle which types of work across an organization.
  Also use when updating an existing model-config.json or model-strategy-guide.md,
  or when the user adds new model subscriptions and needs to re-optimize routing.
  This skill is domain-agnostic — it works for software, finance, e-commerce, legal,
  marketing, healthcare, education, or any combination of business domains.
---

# Model Strategy & Sub-Agent Staffing

This skill interviews the user about their AI model subscriptions, budget, and business domains,
then generates two portable output files that any agentic orchestrator can consume:

1. **`model-config.json`** — Machine-readable routing config (models, roles, budget, quick-reference table)
2. **`model-strategy-guide.md`** — Human-readable strategy document with rationale, role descriptions, budget allocation, risks, and implementation priorities

These files are orchestrator-agnostic. They work with any multi-agent system (Claude Cowork, Hermes,
OpenClaw, AutoGen, CrewAI, custom orchestrators, etc.) because they describe *what* each model is
good at and *which role* it should fill — not *how* a specific framework should call it.

---

## Phase 1: Interview the User

Gather all the information needed to make informed model-to-role assignments. Use the AskUserQuestion
tool or conversational questions. Do not skip any category — each one materially affects the output.

### 1.1 Business Domains & Scope

Ask the user which domains their agentic company will operate in. Present common options but
make it clear they can combine freely or add custom domains:

- Software Engineering & Delivery
- Finance & Accounting
- Legal & Compliance
- Marketing & Content
- Customer Support
- E-Commerce & Retail Operations
- Healthcare / Medical
- Education & Training
- Data Science & Analytics
- Sales & CRM
- HR & Recruiting
- Operations & Supply Chain
- Creative / Design
- Executive / Strategy
- Other (let them describe)

For each selected domain, ask a brief follow-up about the *scope* within that domain. For example:
- Software: "Full-stack web? Mobile? Infrastructure? ML/AI? All of the above?"
- Finance: "Bookkeeping? FP&A? Audit? Tax? Treasury?"
- Legal: "Contract review? Compliance? IP? Litigation support?"

This scope information determines which specialized roles to create within each domain.

### 1.2 Monthly Budget & Spending Cap

Ask:
- "What is your total monthly spending cap on AI model subscriptions and API usage?"
- "Is this a hard cap or a soft target?"

This number constrains everything downstream. If the user doesn't know yet, help them
think through it: "A typical solo agentic startup might spend $30–$100/month on metered
APIs plus fixed subscriptions. A team of 5 might spend $200–$500. What range feels right?"

### 1.3 Model Subscriptions & API Access

For each AI provider the user subscribes to, collect:

| Field | Why it matters |
|---|---|
| **Provider name** | e.g., DeepSeek, OpenAI, Google, Anthropic, Moonshot, Z.ai, Mistral, xAI, Cohere, etc. |
| **Model name(s)** | Specific models available (e.g., GPT-4.1, Claude Sonnet 4.5, Gemini 2.5 Pro, DeepSeek V4 Pro) |
| **Pricing model** | Metered (per-token), fixed subscription, free tier, or hybrid |
| **Monthly budget or plan tier** | Dollar amount for metered; plan name for subscriptions (e.g., "Pro", "Team", "$20/month") |
| **Context window** | If known — otherwise the skill will use publicly available specs |
| **Known strengths** | What the user (or benchmarks) considers this model good at |
| **Known limitations** | Anything the user has noticed — verbosity, hallucination, slow inference, etc. |
| **API compatibility** | OpenAI-compatible? Custom SDK? Platform-only (chat UI, no API)? |
| **Usage caps or quotas** | Rate limits, daily/monthly token caps, session limits |

If the user has many models, help them enumerate by provider: "Let's go provider by provider.
First — which providers do you subscribe to?" Then drill into each.

### 1.4 Orchestrator Preferences

Ask:
- "Do you have a preference for which model acts as the orchestrator (the 'brain' that plans,
  routes, and quality-checks)? Or should I recommend one based on your subscriptions?"
- "Does your orchestrator framework have any model requirements? (e.g., must support tool calling,
  must have OpenAI-compatible API, needs long context)"

### 1.5 Existing Configuration

Ask:
- "Do you already have a model-config.json or model-strategy-guide.md from a previous setup?
  If so, point me to the file and I'll use it as a starting point."
- "Is there a CLAUDE.md or Agents.md in this project that I should update with routing pointers?"

### 1.6 Priority & Constraints

Ask:
- "What matters most: minimizing cost, maximizing quality, or balancing both?"
- "Are there any models you do NOT want to use, or any providers you want to avoid?"
- "Any geopolitical or data-residency constraints? (e.g., avoid China-hosted APIs, must use
  EU-based providers, data must not leave a certain region)"
- "Do you need offline / self-hosted fallback capability?"

---

## Phase 2: Research & Model Assessment

After collecting user answers, build a model landscape table. For each model the user has access to:

1. **Look up current specs** — Use web search if available to verify context window sizes,
   intelligence benchmarks (Artificial Analysis Index, LMSYS Elo, SWE-bench scores, MMLU, etc.),
   and current pricing. User-provided info takes precedence for subjective assessments, but
   verify hard specs.

2. **Score each model** across dimensions relevant to the user's domains:

| Dimension | What to assess |
|---|---|
| Intelligence / reasoning | Benchmark scores, user feedback |
| Code generation | SWE-bench, HumanEval, practical coding ability |
| Long-context handling | Context window size, effective retrieval at depth |
| Tool calling | Structured output, function calling reliability |
| Instruction following | Precision on complex multi-step prompts |
| Speed / latency | Inference speed, time-to-first-token |
| Cost efficiency | Tokens per dollar at the user's pricing tier |
| Sustained execution | Can it maintain quality over long sessions (8hr+)? |
| Multimodal | Vision, image understanding, document analysis |
| Domain expertise | Strength in specific domains (legal, medical, financial, etc.) |
| Verbosity | Does it over-generate? Important for budget control |

Not every dimension matters for every role — use the user's domains to weight appropriately.

3. **Identify the tier** each model falls into. Read `references/output-schemas.md` for the
   standard tier definitions (frontier, strong, worker, fast, cheap, utility).

---

## Phase 3: Role Generation

Based on the user's domains and scope, generate the list of roles their agentic company needs.
Read `references/domain-roles.md` for domain-specific role templates and adapt them to the
user's specific scope answers from Phase 1.

For each role, determine:

| Field | Description |
|---|---|
| `role_id` | snake_case identifier (e.g., `senior_fullstack_dev`, `chief_accountant`) |
| `title` | Human-readable title |
| `description` | What this role does — 1-2 sentences |
| `domain` | Which business domain(s) this role serves |
| `primary_model` | Best-fit model from the user's available models |
| `fallback_model` | Backup model if primary is unavailable or budget-exhausted |
| `escalation_model` | (optional) A stronger model for hard sub-tasks within this role |
| `iteration_model` | (optional) A cheaper model for rapid iteration / brainstorming |
| `usage_pattern` | How often and how to use this role (e.g., "3-5 calls/week, strategic only") |
| `when_to_escalate` | Criteria for escalating to a stronger model |
| `when_to_fallback` | Criteria for falling back to the backup |

### Model-to-Role Assignment Logic

When assigning models to roles, follow these principles (in priority order):

1. **Match model strengths to role requirements.** A role that needs deep reasoning gets
   the highest-intelligence model. A role that needs cheap volume gets the lowest-cost model.

2. **Respect budget constraints.** Metered models have finite monthly budgets. Assign them to
   roles where their unique strengths justify the cost. Use plan-based or cheap models for
   high-volume roles.

3. **Avoid wasting expensive capabilities on routine work.** If a plan-based model handles 80%
   of a role's tasks adequately, make it primary and reserve the frontier model for escalation.

4. **Consider session patterns.** Long coding marathons suit models with sustained execution.
   Quick config tasks suit fast/cheap models. Strategic decisions suit frontier models used sparingly.

5. **Build in fallback chains.** Every role should have at least a primary and fallback. The
   fallback should be on a different provider or pricing model to ensure resilience.

---

## Phase 4: Orchestrator Selection

Choose the orchestrator brain from the user's available models. The orchestrator doesn't do the
work itself — it plans, decomposes tasks, routes to the right sub-agent, and quality-checks results.

Prioritize:
- **Long context window** — the orchestrator needs to hold full project plans, agent outputs, and
  conversation history simultaneously
- **Strong tool calling** — reliable structured output and function invocation
- **Moderate cost** — the orchestrator runs continuously, so verbosity and per-token cost matter
- **Good reasoning** — but doesn't need to be the absolute frontier; "strong" tier suffices
- **API availability** — must have a programmable API, not just a chat UI

Designate a **fallback orchestrator** on a different pricing model (ideally plan-based / zero
metered cost) for when the primary orchestrator's budget runs thin.

---

## Phase 5: Generate Output Files

### 5.1 model-config.json

Generate a JSON file following the schema in `references/output-schemas.md`. The file must include:

- `version`, `last_updated`, `notes`
- `budget` section with metered and fixed subscription breakdowns
- `models` section with specs for every model the user has access to
- `routing.orchestrator` with primary and fallback
- `routing.roles` with every role from Phase 3
- `routing.quick_reference` — a flat lookup table mapping task descriptions to model IDs
- `plan_tier_mapping` — maps abstract tiers (frontier, strong, worker, etc.) to concrete models
- `cost_control` — array of budget-saving strategies specific to the user's model mix
- `api_providers` — endpoint and provider info for each model

Do NOT include any orchestrator-specific fields (no "hermes_*", no "openclaw_*", no framework
namespaces). Keep the schema generic.

### 5.2 model-strategy-guide.md

Generate a markdown document following the template in `references/output-schemas.md`. Structure:

1. **Header** — Date, focus domains, constraints, budget summary
2. **Budget & Plan Summary** — Table of all models with monthly costs
3. **Model Landscape** — Reference table with params, context, cost, intelligence, strengths
4. **Orchestrator Brain** — Recommendation with comparison table and rationale
5. **Sub-Agent Staffing Plan** — Architecture diagram (ASCII) + one section per role with:
   - What they do (table)
   - Why this model (rationale)
   - Usage pattern
   - Budget allocation (for metered models)
6. **Task-to-Model Quick Reference** — Flat routing table
7. **Budget Allocation** — Per-model breakdown by role
8. **Cost Control Levers** — Numbered list of budget-saving strategies
9. **Key Risks & Mitigations** — Table of risks with impact and mitigation
10. **API Providers** — Table of recommended endpoints
11. **Implementation Priorities** — Week-by-week rollout plan
12. **Post-Promo / Future Contingency** — What to do when pricing changes

Write the strategy guide in a direct, opinionated style. Don't hedge — make clear recommendations
with explicit rationale. The user is relying on this document to make real staffing decisions.

### 5.3 File Placement

Save both files to the project root (or the user's preferred location if specified):
- `model-config.json`
- `model-strategy-guide.md`

---

## Phase 6: Update Project Files

After generating the output files, check whether the project has any of these files:

- `CLAUDE.md`
- `Agents.md`
- `agents.md`

If found, update them to include a routing entry pointing to the new config files. Add an entry
like this to the routing table (adapt format to match the existing file's style):

```
| Configure model routing | / (root) | model-config.json, model-strategy-guide.md |
```

If the file has a "## Files" or similar section, add entries describing the two new files.

If neither CLAUDE.md nor Agents.md exists, mention to the user that they may want to create one
with a pointer to the config files so that agents scanning the project know where to look.

---

## Phase 7: Verification

Before presenting the final output to the user, verify:

- [ ] Every model the user listed appears in `models` section of the config
- [ ] Every role has a `primary_model` that exists in the `models` section
- [ ] Every role has at least a `fallback_model`
- [ ] Budget allocations for metered models sum to ≤ the model's monthly budget
- [ ] The orchestrator is not assigned the most expensive model unless justified
- [ ] `quick_reference` covers all domains the user selected
- [ ] `plan_tier_mapping` maps every tier to a concrete model
- [ ] No orchestrator-specific or framework-specific field names appear in the config
- [ ] The strategy guide's role sections match the config's role definitions exactly
- [ ] Cost control strategies reference specific models and pricing from the user's setup

If any check fails, fix it before presenting. Then summarize what was generated and offer
to adjust any assignments the user disagrees with.

---

## Updating an Existing Configuration

If the user points to existing `model-config.json` and/or `model-strategy-guide.md` files:

1. Read and parse the existing files
2. Ask what changed — new models added? Budget changed? Domains expanded? Models deprecated?
3. Preserve role IDs and structure where possible — downstream systems may reference them
4. Re-run the model assessment (Phase 2) with the updated model list
5. Re-optimize assignments and regenerate both files
6. Highlight what changed in a summary diff for the user to review
