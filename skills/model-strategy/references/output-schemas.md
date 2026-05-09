# Output Schemas & Templates

This reference defines the exact structure for the two output files the model-strategy skill
generates. Follow these schemas precisely — downstream orchestrators and agents parse these
files programmatically.

---

## 1. model-config.json Schema

```json
{
  "$schema": "model-config-schema",
  "version": "1.0",
  "last_updated": "YYYY-MM-DD",
  "notes": "Brief description of what this config covers and how to use it.",

  "budget": {
    "total_monthly_cap_usd": 100,
    "cap_type": "hard | soft",
    "metered": {
      "<model-id>": {
        "monthly_usd": 10,
        "notes": "Pricing details, promo windows, etc."
      }
    },
    "fixed_subscriptions": {
      "<subscription-name>": {
        "monthly_usd": 20,
        "models": ["model-a", "model-b"],
        "plan_tier": "Pro | Team | Enterprise | etc.",
        "notes": "Capacity description"
      }
    }
  },

  "models": {
    "<model-id>": {
      "provider": "provider-name",
      "display_name": "Human-Readable Model Name",
      "api_base": "https://api.example.com",
      "params_active": "49B",
      "architecture": "MoE (1.6T total) | Dense | etc.",
      "context_window": 1000000,
      "intelligence_index": 52,
      "swe_bench": 80.6,
      "cost_model": "metered | plan | free",
      "cost_per_mtok_blended": 0.55,
      "strengths": ["list", "of", "strengths"],
      "limitations": ["list", "of", "known", "limitations"],
      "openai_compatible": true,
      "multimodal": false,
      "tool_calling": true,
      "license": "Apache 2.0 | Proprietary | etc.",
      "usage_caps": {
        "max_tokens": 4096,
        "mode": "instant | thinking",
        "daily_limit": null,
        "notes": "Any usage restrictions"
      }
    }
  },

  "routing": {
    "orchestrator": {
      "description": "What the orchestrator does",
      "primary": "<model-id>",
      "fallback": "<model-id>",
      "fallback_trigger": "When to switch to fallback",
      "notes": "Why this model was chosen for orchestration"
    },

    "roles": {
      "<role_id>": {
        "title": "Human-Readable Role Title",
        "description": "What this role does — 1-2 sentences",
        "domain": ["software", "finance"],
        "primary": "<model-id>",
        "fallback": "<model-id>",
        "escalation": "<model-id or null>",
        "escalation_trigger": "When to use the escalation model",
        "iteration_engine": "<model-id or null>",
        "utility": "<model-id or null>",
        "utility_scope": "What the utility model handles",
        "usage_pattern": "How often, how to use",
        "when_to_fallback": "Criteria for switching to fallback",
        "budget_allocation_usd": 3.00,
        "notes": "Additional context"
      }
    },

    "quick_reference": {
      "<task_description_snake_case>": "<model-id>"
    }
  },

  "plan_tier_mapping": {
    "_description": "Maps abstract capability tiers to concrete models from this config.",
    "tier_frontier": "<model-id>",
    "tier_strong": "<model-id>",
    "tier_worker": "<model-id>",
    "tier_fast": "<model-id>",
    "tier_cheap": "<model-id>",
    "tier_utility": "<model-id>"
  },

  "cost_control": [
    "Strategy 1 — specific to user's model mix",
    "Strategy 2 — reference actual pricing and models",
    "Strategy 3 — etc."
  ],

  "api_providers": {
    "<model-id>": {
      "provider": "Provider Name",
      "endpoint": "https://api.example.com",
      "auth_type": "api_key | oauth | plan_based",
      "notes": "Failover options, alternative providers"
    }
  },

  "risks": [
    {
      "risk": "Description of risk",
      "impact": "What happens if it materializes",
      "mitigation": "How to handle it",
      "likelihood": "low | medium | high"
    }
  ]
}
```

### Schema Notes

- **model-id**: Use lowercase-kebab-case derived from the model name (e.g., `deepseek-v4-pro`,
  `gpt-4.1`, `claude-sonnet-4.5`, `gemini-2.5-pro`). This ID is referenced everywhere in the
  config and must be consistent.

- **Nullable fields**: `escalation`, `iteration_engine`, `utility`, `usage_caps` can be `null`
  if not applicable.

- **Domain values**: Use lowercase single words from this list (extend as needed):
  `software`, `finance`, `legal`, `marketing`, `support`, `ecommerce`, `healthcare`,
  `education`, `data`, `sales`, `hr`, `operations`, `creative`, `executive`, `general`

- **No framework namespaces**: The config must NOT contain fields prefixed with any orchestrator
  name (no `hermes_`, `openclaw_`, `autogen_`, `crewai_`, etc.). It is a neutral description
  of model capabilities and role assignments.

---

## 2. Tier Definitions

| Tier | Purpose | Typical Characteristics |
|---|---|---|
| `tier_frontier` | Hardest problems, architectural decisions, novel design | Highest intelligence index, may be expensive or quota-limited |
| `tier_strong` | Core work requiring good reasoning and code generation | Near-frontier intelligence, moderate cost, reliable tool calling |
| `tier_worker` | Sustained coding sessions, long builds, marathon execution | Good code gen, sustained execution capability, plan-based preferred |
| `tier_fast` | Quick supervised tasks, simple routing, fallback orchestration | Fast inference, lower intelligence OK, plan-based preferred |
| `tier_cheap` | High-volume iteration, brainstorming, first-pass review, tests | Lowest cost per token, adequate quality for non-critical work |
| `tier_utility` | Routine config tasks, boilerplate, simple transformations | Cheapest option, no frontier intelligence needed |

---

## 3. model-strategy-guide.md Template

Use this structure for the human-readable strategy document. Adapt section depths and content
to the user's specific setup — don't include empty sections.

```markdown
# Model Strategy & Sub-Agent Staffing Plan

**Date:** {date}
**Focus:** {comma-separated list of business domains}
**Constraints:** {any constraints — excluded providers, data residency, etc.}
**Budget Model:** {brief budget summary}

---

## 1. Budget & Plan Summary

| Model / Provider | Plan Type | Monthly Budget | Effective Capacity |
|---|---|---|---|
| **{Model Name}** | {Metered API / Fixed subscription} | **${amount}** | {token estimate or session count} |

**Total metered spend:** ${total}/month
**Fixed subscriptions:** {list with pricing if known}

---

## 2. Model Landscape — Reference

| Model | Params (Active) | Context | Blended Cost | Intelligence Index | Key Strength |
|---|---|---|---|---|---|
| **{Model}** | {params} | {context} | ${cost} | {index} | {strength} |

> Footnotes explaining index sources, cost calculation method, etc.

---

## 3. Orchestrator Brain — Recommendation

### Winner: **{Model Name}**

{2-3 paragraph rationale explaining why this model was chosen for orchestration,
comparing it against alternatives on context window, cost, verbosity, and tool calling.}

| Criterion | {Model A} | {Model B} | {Model C} |
|---|---|---|---|
| Intelligence Index | ... | ... | ... |
| Context window | ... | ... | ... |
| Tool calling | ... | ... | ... |
| Budget model | ... | ... | ... |

**Fallback orchestrator:** {model} — {when and why to use it}

---

## 4. Sub-Agent Staffing Plan

### Architecture

{ASCII diagram showing orchestrator at top, role groups below, with escalation arrows}

---

### 4.1 {Role Title}

**Model: {Primary Model}** ({plan type})
**Fallback: {Fallback Model}**

| What they do | Why this model |
|---|---|
| {responsibility 1} | {rationale} |
| {responsibility 2} | {rationale} |

**Usage pattern:** {frequency, session type}
**Budget allocation:** ~${amount} of the ${total} {model} budget.

{Repeat for each role}

---

## 5. Task-to-Model Quick Reference

{Formatted as a code block or table mapping task descriptions to model names}

---

## 6. Budget Allocation — Metered Models

### {Model Name} (${budget}/month)

| Role | Est. Token Usage | Est. Spend |
|---|---|---|
| {role} | {tokens} | ${amount} |
| **Total** | **{tokens}** | **${amount}** |

{Repeat per metered model}

### {Subscription Name} (Fixed)

| Model | Role | Usage Pattern |
|---|---|---|
| {model} | {role} | {pattern} |

---

## 7. Cost Control Levers

1. **{Strategy}** — {specific implementation details referencing the user's models and pricing}
2. ...

---

## 8. Key Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| {risk} | {impact} | {mitigation} |

---

## 9. Recommended API Providers

| Model | Provider | Why |
|---|---|---|
| {model} | **{provider}** | {rationale} |

---

## 10. Implementation Priorities

1. **Week 1:** {setup tasks}
2. **Week 2:** {integration tasks}
3. **Week 3:** {expansion tasks}
4. **Week 4:** {optimization and stress testing}

---

## 11. Contingency Planning

{What to do when pricing changes, plans expire, providers go down, or the company's
domain focus shifts. Specific scenarios with concrete action plans.}

---

*Generated by model-strategy skill. All pricing verified as of {date}.
Focus: {domains}. Orchestrator-agnostic configuration.*
```

---

## 4. CLAUDE.md / Agents.md Routing Entry

When updating project routing files, add entries in the style that matches the existing file.
If the file uses a table format:

```markdown
| Configure model routing | / (root) | model-config.json, model-strategy-guide.md |
| View available AI models | / (root) | model-config.json |
| Understand role assignments | / (root) | model-strategy-guide.md |
```

If the file uses a prose or list format, adapt accordingly. The key information to convey:

- These two files exist at the project root
- `model-config.json` is the machine-readable config for agent routing
- `model-strategy-guide.md` is the human-readable rationale and implementation guide
- Any agent that needs to decide which model to use for a task should consult these files
