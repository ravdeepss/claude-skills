# Hermes Agent — Model Strategy & Sub-Agent Staffing Plan (v2)

**Date:** May 2026  
**Focus:** Software Engineering & Delivery only  
**Constraint:** No Anthropic/Claude models permitted  
**Budget Model:** Fixed subscription plans + metered DeepSeek usage

---

## 1. Budget & Plan Summary

| Model / Provider | Plan Type | Monthly Budget | Effective Capacity |
|---|---|---|---|
| **DeepSeek V4 Pro** | Metered API | **$10 USD** | ~18M tokens blended (promo) / ~72M tokens (promo, cached) |
| **Gemma 4 31B** | Metered API | **$20 USD** | ~105M tokens blended at $0.19/MTok |
| **Z.ai / GLM-5.1 + GLM-5-Turbo + GLM-4.7** | Coding Plan (monthly) | Fixed subscription | ~5x Claude Pro equivalent usage (~250+ heavy coding sessions/month) |
| **Kimi K2.6** | Moderato Plan (basic) | Fixed subscription | Limited monthly allowance — use strategically |

**Total metered spend:** $30 USD/month  
**Fixed subscriptions:** Z.ai Coding Plan + Kimi Moderato (pricing outside the metered budget)

---

## 2. Model Landscape — Engineering-Focused Reference

| Model | Params (Active) | Context | Blended Cost¹ | Intelligence Index² | Key Engineering Strength |
|---|---|---|---|---|---|
| **DeepSeek V4 Pro** | 1.6T (49B) MoE | 1M | $0.55 ($0.14 promo) | 52 | Cheapest frontier reasoning, 1M context, 80.6% SWE-bench |
| **GLM-5.1** | 744B (40B) MoE | 200K | Plan-based | 51 | 8hr sustained execution, 655+ iterations, SWE-Bench Pro #1 |
| **GLM-5-Turbo** | Proprietary | 200K | Plan-based | ~45 | Fast inference, supervised agent tasks |
| **GLM-4.7** | Smaller | 200K | Plan-based | ~40 | Cheapest Z.ai model, solid for routine infra/config tasks |
| **Kimi K2.6** | 1T (32B) MoE | 256K | Plan-based | **54** | Highest raw intelligence, agent swarm native, multimodal |
| **Gemma 4 31B** | 31B Dense | 256K | $0.19 | ~35 | Apache 2.0, multimodal, great math, dirt cheap iteration |

> ¹ Blended = 3:1 input:output ratio at standard pricing  
> ² Artificial Analysis Intelligence Index v4.0 (higher = better)

---

## 3. Orchestrator Brain — Recommendation

### Winner: **DeepSeek V4 Pro**

The orchestrator doesn't write code itself — it *plans, decomposes, routes, and quality-checks*. What matters is reasoning depth, tool-calling reliability, and cost efficiency for continuous agentic loops.

| Criterion | DeepSeek V4 Pro | Kimi K2.6 | GLM-5.1 |
|---|---|---|---|
| Intelligence Index | 52 | **54** | 51 |
| Context window | **1M** | 256K | 200K |
| Tool calling | ✅ Excellent | ✅ Good | ✅ Good |
| Agentic design | Strong | **Native swarm** | 8hr sustained |
| Verbosity | Moderate | **Very verbose** | Verbose |
| Budget model | Metered ($10) | Fixed (Moderato) | Fixed (Coding Plan) |
| OpenAI-compatible API | ✅ | ✅ | ✅ |

**Why DeepSeek V4 Pro wins for orchestration:**

- **1M context window** lets the orchestrator hold the entire project plan, agent outputs, and conversation history in a single pass — critical for multi-agent coordination
- **Moderate verbosity** means orchestrator meta-reasoning doesn't blow the $10 budget
- **Kimi K2.6 is dangerously verbose** — 170M output tokens during Intelligence Index evaluation vs 42M average. On a Moderato plan with limited allowance, burning tokens on orchestration wastes the plan's value
- **GLM-5.1's 8hr sustained execution is wasted on orchestration** — that capability shines in a *worker* role doing deep coding, not plan-and-dispatch
- At promo pricing (~$0.14/MTok blended), $10 buys ~72M tokens/month with aggressive prompt caching — sufficient for orchestration duties
- Post-promo ($0.55/MTok blended): $10 buys ~18M tokens — tighter but viable if system prompts are cached aggressively

**Fallback orchestrator:** GLM-5-Turbo (fast, part of Z.ai plan, no metered cost) if DeepSeek budget runs thin late in the month.

---

## 4. Sub-Agent Staffing Plan — Engineering & Delivery

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    HERMES ORCHESTRATOR                      │
│                    DeepSeek V4 Pro                          │
│              (Planning · Routing · QA Gate)                 │
└──────────┬──────────┬──────────┬──────────┬────────────────┘
           │          │          │          │
     ┌─────▼───┐ ┌───▼────┐ ┌──▼───┐ ┌───▼────┐
     │ Arch &  │ │  Dev   │ │  UX  │ │  Ops   │
     │ Design  │ │  Pool  │ │  &   │ │  &     │
     │ Review  │ │        │ │ Front│ │  Infra │
     └─────────┘ └────────┘ └──────┘ └────────┘
           │          │                    │
     ┌─────▼───┐ ┌───▼────┐         ┌───▼────┐
     │  Code   │ │Marathon│         │  QA    │
     │ Review  │ │Sessions│         │ Review │
     └─────────┘ └────────┘         └────────┘
```

---

### 4.1 Principal Engineer / Architect

**Model: Kimi K2.6** (Moderato plan)  
**Fallback: GLM-5.1**

| What they do | Why this model |
|---|---|
| System design decisions | K2.6 has the highest Intelligence Index (54) — strongest pure reasoner for architectural trade-offs |
| Architecture reviews | Multimodal — can analyze diagrams, whiteboards, existing system screenshots |
| Complex trade-off analysis | Verbosity is a *feature* for architecture docs — you want exhaustive reasoning |
| MCP schema design | Deep structured thinking about API contracts |

**Usage pattern:** Strategic escalation only — 3-5 calls/week max. Reserve Kimi's limited Moderato allowance for genuinely hard decisions where the 2-point Intelligence Index advantage over DeepSeek matters.

**Prompt pattern:** Frame the architectural question with full context, constraints, and 2-3 options. Request structured output (decision + rationale + risks). Set `max_tokens` to 4096 to control verbosity within plan limits.

**When to use GLM-5.1 instead:** For 80% of architectural questions that are "solid engineering judgment" rather than "genuinely novel design problems" — GLM-5.1 handles these well and doesn't burn Kimi plan allowance.

---

### 4.2 Senior Full Stack Developer

**Model: DeepSeek V4 Pro** (standard tasks)  
**Heavy sessions: GLM-5.1** (Z.ai Coding Plan)

| What they do | Why this model |
|---|---|
| Feature implementation | V4 Pro scores 80.6% SWE-bench — near frontier at fraction of cost |
| MCP server/client code | Strong tool-calling and structured output |
| API development | OpenAI-compatible, understands modern patterns |
| Database work (Postgres, Supabase) | 1M context handles full schema + migration context |
| Code generation (Node/Python/TypeScript) | Strong across all three stacks |

**For marathon coding sessions** (building an entire MCP server, complex feature end-to-end, multi-file refactors), escalate to **GLM-5.1** — its 8-hour sustained execution with 655+ iteration capability is purpose-built for this. The Z.ai Coding Plan makes this effectively unlimited for sustained development work.

**Budget allocation:** ~$4-6 of the $10 DeepSeek budget goes here (shared with orchestrator).

---

### 4.3 Senior Front End Developer

**Model: GLM-5.1** (primary)  
**Rapid iteration: Gemma 4 31B**

| What they do | Why this model |
|---|---|
| React/Next.js components | GLM-5.1's sustained execution handles multi-component builds in one session |
| Tailwind/CSS implementation | Solid pattern matching for design system implementations |
| Responsive layouts | Long context + iteration = gets responsive right |
| Component architecture | Good at composing reusable, well-typed components |
| Full page builds | 8hr session capability means entire pages built in one shot |

**Gemma 4 31B as the iteration engine:** For rapid prototyping, trying 3-4 layout variations, quick component sketches, and boilerplate generation (forms, tables, CRUD), Gemma at $0.19/MTok is effectively unlimited within the $20 budget. Use it for the "throw stuff at the wall" phase, then have GLM-5.1 polish and integrate.

**Why GLM-5.1 over DeepSeek for frontend:** Frontend work is iterative and session-heavy — you tweak, preview, tweak again. GLM-5.1's sustained execution model (no timeout, 655+ iterations) fits this workflow better than DeepSeek's metered per-token model. And it's on a fixed plan, so no budget anxiety during long UI sessions.

---

### 4.4 UX Designer

**Model: Kimi K2.6** (for UX strategy/audit)  
**Wireframes/flows: Gemma 4 31B**

| What they do | Why this model |
|---|---|
| UX audit & recommendations | K2.6's multimodal capability can analyze existing UI screenshots |
| User flow design | Highest Intelligence Index = strongest reasoning about user journeys |
| Design system specs | Thorough, exhaustive output for token/spacing/color systems |
| Accessibility review | Deep, doesn't skip WCAG considerations |

**Gemma 4 31B as the sketch pad:** Quick wireframe descriptions, flow diagrams in text/Mermaid, and initial brainstorming at $0.19/MTok. Iterate cheaply, refine the winner, then send to Kimi for a UX audit pass.

**Usage pattern:** 2-4 Kimi calls/week for UX review. Gemma handles all exploration.

---

### 4.5 Senior Cloud Engineer / DevOps

**Model: DeepSeek V4 Pro** (architecture-level infra)  
**Routine tasks: GLM-4.7** (Z.ai plan)

| What they do | Why this model |
|---|---|
| AWS CDK/CloudFormation | V4 Pro's 1M context handles full IaC stacks |
| Supabase config & RLS policies | Strong SQL and policy reasoning |
| Vercel deployment configs | Understands edge functions, middleware patterns |
| CI/CD pipelines | Good at GitHub Actions, Docker workflows |
| Cost optimization | Can analyze billing patterns with full context |

**GLM-4.7 as the utility player:** Handles routine infra tasks on the Z.ai plan at no metered cost — Dockerfile tweaks, env configs, simple Terraform, nginx configs, docker-compose files. Think of it as the "junior DevOps engineer" handling tickets while V4 Pro handles architecture.

**Budget allocation:** ~$2-3 of DeepSeek budget for infra architecture calls.

---

### 4.6 QA Engineer / Code Reviewer

**Model: Gemma 4 31B** (first pass)  
**Escalation: DeepSeek V4 Pro**

| What they do | Why this model |
|---|---|
| Automated test generation | Gemma's math strength helps with edge case identification |
| Code review (style/patterns) | Cheap enough to review every PR |
| Test coverage analysis | Can process test files + source together in 256K context |
| Linting/formatting checks | Simple pattern matching — doesn't need frontier intelligence |

**Escalation to DeepSeek V4 Pro:** For security reviews, complex logic verification, race condition analysis, and architectural code review where 1M context is needed to understand the full system.

**Budget allocation:** ~$8-12 of the $20 Gemma budget. Gemma is the workhorse for QA — cheap enough to run on every commit.

---

### 4.7 Legacy Java/Spring Boot Specialist

**Model: GLM-5.1** (Z.ai Coding Plan)  
**Fallback: DeepSeek V4 Pro**

| What they do | Why this model |
|---|---|
| Spring Boot services | GLM-5.1's sustained execution handles Java's verbosity well |
| Migration work | Long sessions for refactoring large codebases |
| API integration | REST/GraphQL endpoint generation |
| JPA/Hibernate | Solid ORM pattern knowledge |

**No separate metered budget needed** — runs entirely on the Z.ai Coding Plan. For quick questions or lookups, DeepSeek V4 Pro's 1M context can hold entire Spring projects.

---

## 5. Model-to-Role Quick Reference

```
When Hermes needs to...              → Route to...
─────────────────────────────────────────────────────
Plan & decompose a project           → DeepSeek V4 Pro (orchestrator)
Make an architecture decision        → Kimi K2.6
Get a second opinion on design       → GLM-5.1
Write backend code (Node/Python/TS)  → DeepSeek V4 Pro (short) / GLM-5.1 (long)
Write Java/Spring Boot code          → GLM-5.1
Build React/Next.js UI               → GLM-5.1 (build) + Gemma 4 (iterate)
Design a user experience             → Kimi K2.6 (audit) + Gemma 4 (explore)
Write MCP server code                → GLM-5.1 (marathon session)
Configure AWS/Supabase/Vercel        → DeepSeek V4 Pro
Write a Dockerfile or CI/CD          → GLM-4.7 (free on plan)
Do a long coding marathon (8hr+)     → GLM-5.1
Review code quality                  → Gemma 4 31B (first pass) → V4 Pro (escalation)
Generate tests                       → Gemma 4 31B
Run security review                  → DeepSeek V4 Pro
Brainstorm or rough-draft anything   → Gemma 4 31B
Quick config/env/simple task         → GLM-4.7
Rapid UI prototyping                 → Gemma 4 31B
```

---

## 6. Budget Allocation — Metered Models

### DeepSeek V4 Pro ($10/month)

| Role | Est. Token Usage | Est. Spend |
|---|---|---|
| Orchestrator (planning, routing, QA gate) | 8-12M tokens | $4-6 |
| Full Stack Dev (short coding tasks) | 4-6M tokens | $2-3 |
| Cloud Engineer (infra architecture) | 2-4M tokens | $1-2 |
| Code Review escalation | 1-2M tokens | $0.50-1 |
| **Total** | **~15-24M tokens** | **~$8-10** |

*Assumes promo pricing through May 31. Post-promo: reduce orchestrator verbosity, lean harder on GLM-5-Turbo as fallback orchestrator.*

### Gemma 4 31B ($20/month)

| Role | Est. Token Usage | Est. Spend |
|---|---|---|
| QA/Test generation (primary) | 40-60M tokens | $8-12 |
| Frontend rapid iteration | 20-30M tokens | $4-6 |
| UX exploration/wireframes | 10-15M tokens | $2-3 |
| Brainstorming/rough drafts | 10-20M tokens | $2-4 |
| **Total** | **~80-125M tokens** | **~$16-20** |

*Gemma is the high-volume workhorse. At $0.19/MTok, 100M+ tokens/month is achievable within budget.*

### Z.ai Coding Plan (Fixed subscription)

| Model | Role | Usage Pattern |
|---|---|---|
| GLM-5.1 | Marathon coding, frontend builds, Java work | 10-20 long sessions/week |
| GLM-5-Turbo | Fallback orchestrator, supervised tasks | 5-10 calls/week |
| GLM-4.7 | Utility infra tasks, Dockerfiles, configs | 15-25 quick calls/week |

*The 5x Claude Pro equivalent means ~250+ heavy coding sessions/month — more than enough for a solo startup.*

### Kimi K2.6 Moderato Plan (Fixed subscription)

| Role | Usage Pattern | Notes |
|---|---|---|
| Architecture decisions | 3-5 calls/week | Set max_tokens=4096 |
| UX audit & strategy | 2-4 calls/week | Use Instant mode (temp 0.6) |
| Complex trade-off analysis | 1-2 calls/week | Only genuinely hard problems |

*Moderato plan has limited allowance — every Kimi call should be high-value. Never use Kimi for anything GLM-5.1 or DeepSeek can handle.*

---

## 7. Cost Control Levers

1. **Prompt caching on DeepSeek** — Cache-hit pricing is 1/10th of input price (since April 26, 2026). The Hermes system prompt + tool definitions + project context repeats on every orchestrator call. Caching alone could cut orchestrator costs by 60-80%, stretching the $10 further
2. **GLM-5-Turbo as overflow orchestrator** — When DeepSeek budget gets tight late in the month, route simple orchestration (task dispatch, status checks) to GLM-5-Turbo on the Z.ai plan at zero metered cost
3. **Gemma for everything that doesn't need frontier intelligence** — At $0.19/MTok, Gemma is 3x cheaper than DeepSeek. Any task that's pattern-matching rather than novel reasoning goes to Gemma first
4. **Kimi verbosity control** — Always set `max_tokens` to 4096 and use Instant mode (not Thinking mode) to stay within Moderato limits
5. **Batch exploration on Gemma** — When brainstorming approaches, generate 3-4 options on Gemma, then only escalate the best candidate to a frontier model for refinement
6. **DeepSeek promo window** — Burn through heavy orchestration setup work before May 31 while the 75% discount is active

---

## 8. Key Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| DeepSeek promo ends May 31 | Orchestrator cost ~4x increase within $10 budget | Aggressive prompt caching + GLM-5-Turbo as fallback orchestrator for simple routing |
| DeepSeek API instability (China-hosted) | Orchestrator downtime | GLM-5-Turbo as hot-standby orchestrator. OpenRouter as multi-provider failover |
| Kimi Moderato allowance exhausted mid-month | No architecture advisor | GLM-5.1 handles 80% of architectural questions. Reserve Kimi for the 20% that truly need highest intelligence |
| Z.ai plan usage cap hit | No marathon coding sessions | Unlikely at 5x Claude Pro equivalent, but monitor. DeepSeek V4 Pro can handle shorter coding tasks as overflow |
| Gemma quality ceiling on complex tasks | QA misses subtle bugs, frontend iteration produces mediocre results | Always escalate complex reviews to DeepSeek or GLM-5.1. Gemma is first-pass only |
| Geopolitical API access risk (Chinese models) | DeepSeek, GLM, Kimi APIs could face restrictions | All three have open weights on HuggingFace for self-hosting. Gemma (Google) is US-based. OpenRouter provides multi-region failover |
| No Claude = weaker instruction-following | Some tasks may need more iteration | Compensate with more explicit prompts, structured output schemas, and validation loops in the orchestrator |

---

## 9. Recommended API Providers

| Model | Provider | Why |
|---|---|---|
| DeepSeek V4 Pro | **DeepSeek direct** (api.deepseek.com) | Cheapest, promo pricing, 1M context, prompt caching |
| GLM-5.1 | **Z.ai Coding Plan** | Fixed subscription, sustained execution |
| GLM-5-Turbo | **Z.ai Coding Plan** | Same subscription, fast inference |
| GLM-4.7 | **Z.ai Coding Plan** | Same subscription, utility tasks |
| Kimi K2.6 | **Moonshot AI** (Moderato plan) | Direct subscription, predictable allowance |
| Gemma 4 31B | **OpenRouter** or **Fireworks** | $0.13/$0.38, some providers offer free tier |

**Single-router fallback:** OpenRouter gives one API key for DeepSeek and Gemma if direct APIs have downtime. Slight markup but good reliability insurance.

---

## 10. Implementation Priorities

1. **Week 1:** Set up Hermes with DeepSeek V4 Pro as orchestrator brain. Configure system prompt with MCP tool definitions. Test orchestration loop. Validate prompt caching is working (critical for $10 budget)
2. **Week 2:** Integrate Z.ai Coding Plan models (GLM-5.1, GLM-5-Turbo, GLM-4.7) as sub-agents. Test marathon coding sessions with GLM-5.1. Implement GLM-5-Turbo as fallback orchestrator
3. **Week 3:** Add Gemma 4 31B for QA, iteration, and exploration tasks. Set up code review pipeline (Gemma first-pass → DeepSeek escalation). Test frontend rapid iteration workflow
4. **Week 4:** Add Kimi K2.6 for architecture decisions. Implement budget tracking and alerting. Tune routing logic to minimize Kimi usage for maximum value. Stress-test the full pipeline on a real feature delivery

---

## 11. Post-Promo Contingency (June 2026+)

When DeepSeek's 75% discount expires:

| Scenario | Action |
|---|---|
| $10 still covers orchestration with caching | Continue as-is, monitor usage |
| $10 becomes tight | Route 50% of orchestration to GLM-5-Turbo (plan-based, zero metered cost). Reserve DeepSeek for complex planning that needs 1M context |
| $10 insufficient | Reduce DeepSeek to architecture-level infra only ($3-4). Promote GLM-5-Turbo to primary orchestrator. Accept slightly lower routing quality |

The key insight: the Z.ai Coding Plan is the safety net. If metered budgets get squeezed, GLM models absorb overflow at no additional cost.

---

*Generated for Hermes Agent architecture planning. All pricing verified as of May 2026. Focus: Software Engineering & Delivery. No Anthropic/Claude models in the stack.*
