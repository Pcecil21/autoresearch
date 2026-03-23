# AutoEvolve — Self-Improving Agent Framework

## Adapted from Karpathy's AutoResearch for Peter Cecil's Portfolio

**Date:** March 22, 2026
**Author:** Claude (with Peter)
**Status:** SPEC — awaiting review

---

## What This Is

A shared framework that applies the AutoResearch "keep/discard" loop to **agent prompt optimization** across Peter's projects. Instead of optimizing ML training code, we optimize the system prompts, parameters, and configurations that power AI agents in production.

**The Karpathy Loop (adapted):**
```
Define success metric →
Generate prompt variant →
Run against test data →
Measure result →
Keep if better, discard if worse →
Repeat overnight →
Human reviews winners in the morning
```

**No GPU required.** This runs on Claude API + Supabase — the same stack every project already uses.

---

## Core Architecture

### Three Components (shared across all projects)

#### 1. Experiment Engine (`autoevolve-engine`)
- Supabase Edge Function triggered by cron (nightly) or manual
- Pulls current "production" prompt for a given agent
- Generates N variants (Claude proposes mutations)
- Runs each variant against test cases
- Scores results
- Logs everything
- Promotes winner IF it beats baseline AND passes safety checks

#### 2. Test Harness (per-project)
- Each project defines:
  - **Test cases** — synthetic or historical interactions
  - **Success metric** — what "better" means (numeric score)
  - **Safety validator** — what disqualifies a variant (PII leak, wrong info, harmful tone)
  - **Budget cap** — max API spend per experiment run

#### 3. Dashboard & Human Gate
- Supabase table: `autoevolve_experiments`
- Shows: variant, score, diff from baseline, safety pass/fail
- **Nothing goes to production without human approval**
- One-click promote or discard

---

## Database Schema (shared Supabase)

```sql
-- Shared across all projects
CREATE TABLE autoevolve_agents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project TEXT NOT NULL,           -- 'north-shore-ai', 'fittrack', etc.
  agent_name TEXT NOT NULL,        -- 'medical-receptionist', 'daily-insight', etc.
  current_prompt TEXT NOT NULL,    -- production system prompt
  current_version INT DEFAULT 1,
  metric_name TEXT NOT NULL,       -- 'appointment_booking_rate', 'engagement_score'
  metric_direction TEXT DEFAULT 'higher', -- 'higher' or 'lower'
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE autoevolve_experiments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID REFERENCES autoevolve_agents(id),
  run_tag TEXT NOT NULL,           -- 'mar22-overnight'
  variant_prompt TEXT NOT NULL,
  mutation_description TEXT,       -- what changed and why
  baseline_score NUMERIC,
  variant_score NUMERIC,
  improvement_pct NUMERIC,
  safety_passed BOOLEAN DEFAULT false,
  status TEXT DEFAULT 'pending',   -- pending, running, keep, discard, crash, promoted
  test_cases_run INT,
  api_cost_usd NUMERIC,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE autoevolve_test_cases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID REFERENCES autoevolve_agents(id),
  input_data JSONB NOT NULL,       -- the test scenario
  expected_behavior TEXT,          -- what good looks like
  weight NUMERIC DEFAULT 1.0,     -- some test cases matter more
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE autoevolve_prompt_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID REFERENCES autoevolve_agents(id),
  version INT NOT NULL,
  prompt TEXT NOT NULL,
  promoted_from_experiment UUID REFERENCES autoevolve_experiments(id),
  promoted_at TIMESTAMPTZ DEFAULT now(),
  promoted_by TEXT DEFAULT 'human' -- 'human' always, no auto-promote
);
```

---

## Security Guardrails

### 1. No Auto-Deploy (Human Gate)
- Winning variants are flagged for review, NEVER auto-promoted
- Peter reviews diffs in the morning
- One-click approve/reject in dashboard

### 2. Sandbox Only
- Experiments run against SYNTHETIC test cases, never live interactions
- Test cases are anonymized — no real names, no real medical info, no real financial data
- Production agents are never modified during experiments

### 3. Budget Caps
- Per-run cap: configurable per project (default $5/night)
- Per-variant cap: max tokens per test case evaluation
- Kill switch: if cumulative cost exceeds cap, stop and report

### 4. Safety Validator
Every variant output is checked by a separate Claude call:
- Does it contain PII? → FAIL
- Does it give medical/financial advice it shouldn't? → FAIL
- Does it hallucinate facts? → FAIL
- Does it match the brand tone? → PASS/FAIL
- Does it comply with regulatory requirements? → PASS/FAIL (RIA: fiduciary, FINRA)

### 5. Prompt Diff Review
- Every variant stores a human-readable diff from the baseline
- Dashboard shows exactly what changed, not just the score
- No "black box" improvements

### 6. Rollback
- Full version history in `autoevolve_prompt_history`
- One-click rollback to any previous version
- Production prompt is always explicit, never computed

### 7. API Key Isolation
- AutoEvolve uses its own Anthropic API key with spend limits
- Separate from production agent API keys
- Rate-limited at the key level

---

## Project Applications

### 1. NORTH SHORE AI SERVICES — Highest Leverage

**Why:** This IS the product. Self-improving agents are the moat.

#### Agent: Medical/Dental Receptionist
- **What it does:** Handles incoming calls, books appointments, answers FAQs
- **Success metric:** `appointment_booking_rate` (% of calls that result in a booking)
- **Test cases:** 50 synthetic patient call transcripts with varied scenarios:
  - New patient wanting to schedule
  - Existing patient rescheduling
  - Patient asking about insurance
  - After-hours inquiry
  - Urgent/emergency triage
  - Confused elderly caller
  - Non-English speaker
- **What the loop optimizes:**
  - Greeting warmth and professionalism
  - Question sequencing (what to ask first)
  - Objection handling ("I'll call back later")
  - Urgency detection accuracy
  - Hold/transfer scripting
- **Safety validator:** Must never give medical advice, must always offer to connect with staff for clinical questions
- **Budget:** $3/night (50 test cases × ~$0.06/case)
- **Expected outcome:** 10-25% improvement in booking rate over 2 weeks of nightly runs

#### Agent: Insurance Agency Intake
- **Success metric:** `lead_qualification_accuracy` (correct segmentation of prospect intent)
- **Test cases:** 40 synthetic intake conversations
- **What the loop optimizes:** Question flow, needs identification, handoff timing
- **Safety validator:** Must not make coverage promises or binding quotes

#### Agent: Restaurant Voice AI
- **Success metric:** `reservation_completion_rate` + `upsell_acceptance_rate`
- **Test cases:** 30 synthetic reservation calls
- **What the loop optimizes:** Greeting, availability communication, party size handling, special requests, waitlist management
- **Safety validator:** Must not confirm reservations that don't exist in the system

---

### 2. FITTRACK / HEALTH CEO

**Why:** Peter is the only user. His behavior IS the feedback signal.

#### Agent: Daily Insight Generator
- **What it does:** Generates the morning brief insight on the Health CEO homepage
- **Current model:** Claude Haiku via Supabase Edge Function
- **Success metric:** `engagement_score` (composite of: did Peter tap to expand? did he act on it? did he dismiss it?)
- **Test cases:** 30 historical days of Peter's health data + what he actually did that day
- **What the loop optimizes:**
  - Insight specificity (vague vs. actionable)
  - Priority ordering (sleep vs. nutrition vs. training vs. bloodwork)
  - Tone (clinical vs. coaching vs. motivational)
  - Length (too long = skipped, too short = unhelpful)
  - Correlation surfacing (connecting WHOOP recovery to yesterday's training)
- **Safety validator:** Must not contradict bloodwork results, must not give medical diagnoses
- **Budget:** $2/night
- **Expected outcome:** Insights that Peter actually acts on 3x more often

#### Agent: Bloodwork Analyzer
- **What it does:** PhD-level analysis of Function Health biomarker panels
- **Current model:** Claude Sonnet via Vercel serverless
- **Success metric:** `clinical_accuracy_score` (evaluated by a separate Claude instance with medical knowledge)
- **Test cases:** Peter's 7 historical blood draws + known clinical interpretations
- **What the loop optimizes:**
  - Accuracy of trend identification
  - Quality of cross-biomarker correlation
  - Actionability of recommendations
  - Appropriate hedging (when to say "discuss with your doctor")
- **Safety validator:** Must not diagnose conditions, must flag when values are critically out of range
- **Budget:** $3/night (larger prompts)

---

### 3. RIA CLIENT TOOLS (Blue Line Advisory)

**Why:** Client-facing AI that represents Peter's practice. Quality = trust = retention.

#### Agent: Portfolio Commentary Generator
- **What it does:** Generates personalized portfolio commentary for client review meetings
- **Success metric:** `relevance_score` (does the commentary address the client's actual concerns and holdings?)
- **Test cases:** 10 anonymized client profiles with known portfolios and concerns
- **What the loop optimizes:**
  - Personalization depth
  - Market context integration
  - Risk communication clarity
  - Action item specificity
  - Compliance-safe language
- **Safety validator:** Must not make performance guarantees, must include appropriate disclaimers, must match ADV Part 2A fee disclosure
- **Budget:** $3/night

#### Agent: Financial Planning Nudge
- **What it does:** Proactive suggestions (Roth conversion window, 529 overfunding, tax-loss harvesting)
- **Success metric:** `action_rate` (did the client or Peter act on the nudge?)
- **Test cases:** 15 anonymized household scenarios with known optimal actions
- **What the loop optimizes:**
  - Timing relevance (right nudge at right time)
  - Urgency calibration
  - Explanation clarity for non-financial audience
- **Safety validator:** Must not make specific tax promises, must recommend consulting CPA for tax decisions
- **Budget:** $2/night

---

### 4. ICE SCHEDULER

**Why:** The solver has parameters that affect schedule quality. The loop can tune them.

#### Agent: Schedule Quality Evaluator
- **What it does:** Evaluates generated schedules against fairness, balance, and preference criteria
- **Success metric:** `schedule_quality_score` (composite: fairness index + preference satisfaction + prime time equity)
- **Test cases:** 5 historical WHA season configurations with known "good" and "bad" schedules
- **What the loop optimizes:**
  - Not the prompt — the **solver constraint weights** (how much to prioritize fairness vs. preference vs. travel)
  - The schedule explanation text (how the system communicates trade-offs to the hockey director)
- **Safety validator:** Must not produce schedules that violate hard constraints (CSDHL game durations, end times)
- **Budget:** $2/night (solver runs are CPU, only evaluation uses Claude)

---

### 5. BIRKIN BROCCOLI

**Why:** Kelly's business could benefit from smarter menu recommendations and ordering flow optimization — but ONLY if Kelly adopts the app.

#### Agent: Menu Recommendation Engine (FUTURE — contingent on Kelly adoption)
- **What it does:** Suggests weekly menus based on customer preferences, season, inventory
- **Success metric:** `order_conversion_rate` (% of menu views that become orders)
- **Test cases:** 20 synthetic customer profiles with dietary preferences
- **What the loop optimizes:**
  - Menu description copy (appetizing language)
  - Dietary match accuracy
  - Seasonal relevance
  - Cross-sell suggestions ("customers who ordered X also loved Y")
- **Safety validator:** Must accurately flag allergens, must not recommend items that conflict with stated dietary restrictions
- **Budget:** $1/night
- **STATUS:** BLOCKED — not buildable until Kelly is actively using the app and generating real order data

---

### 6. FORA TRAVEL

**Why:** Travel itinerary quality is subjective but measurable. The loop can optimize how the Explorer agent builds trips.

#### Agent: Itinerary Generator (Explorer Agent)
- **What it does:** Generates multi-day trip itineraries based on client preferences
- **Success metric:** `itinerary_quality_score` (composite: variety, pacing, budget fit, preference match, logistics feasibility)
- **Test cases:** 10 synthetic trip briefs with known "excellent" itineraries for comparison
- **What the loop optimizes:**
  - Activity variety and pacing
  - Budget allocation
  - Local vs. tourist experience balance
  - Family-friendliness calibration
  - Logistics feasibility (travel times, opening hours)
- **Safety validator:** Must not recommend unsafe destinations, must respect budget constraints
- **Budget:** $3/night
- **STATUS:** PARKED — Fora Travel is in Idea stage, build this when it moves to Prototype

---

## Implementation Plan

### Phase 0: Shared Infrastructure (Week 1)
- [ ] Create `autoevolve` Supabase project (or add tables to existing shared project)
- [ ] Run database migrations (schema above)
- [ ] Build the Experiment Engine edge function
- [ ] Build the Safety Validator edge function
- [ ] Build the Mutation Generator (Claude generates prompt variants with explanations)
- [ ] Build the Scorer (runs variant against test cases, returns numeric score)
- [ ] Build simple dashboard page (list experiments, approve/reject)
- [ ] Set up isolated API key with spend limits

### Phase 1: FitTrack Daily Insight (Week 2) — Easiest Win
**Why start here:** Single user (Peter), clear feedback signal, low risk, fast iteration.
- [ ] Define 30 test cases from Peter's historical health data
- [ ] Register the daily-insight agent in `autoevolve_agents`
- [ ] Write the current Haiku system prompt as v1 baseline
- [ ] Run first overnight experiment batch
- [ ] Peter reviews results, promotes winner
- [ ] Measure: does the new prompt produce insights Peter acts on more?

### Phase 2: North Shore AI — Medical Receptionist (Week 3-4) — Highest Revenue Impact
**Why second:** This is the product. Needs the safety validator to be solid first.
- [ ] Build 50 synthetic patient call test cases
- [ ] Define medical safety rules for the validator
- [ ] Register the medical-receptionist agent
- [ ] Run experiments against synthetic calls
- [ ] A/B test winning prompt against baseline on first pilot client
- [ ] Measure: appointment booking rate improvement

### Phase 3: RIA Client Tools (Week 5-6) — Compliance-Critical
**Why third:** Compliance layer needs to be bulletproof.
- [ ] Build anonymized client profile test cases
- [ ] Define FINRA/fiduciary safety rules
- [ ] Register portfolio-commentary and planning-nudge agents
- [ ] Run experiments
- [ ] Peter reviews for compliance before any client exposure

### Phase 4: Ice Scheduler + Others (Week 7+) — As Needed
- [ ] Extend to solver parameter tuning
- [ ] Birkin Broccoli only if Kelly adopts
- [ ] Fora Travel only if it moves to Prototype

---

## Cost Projections

| Project | Agents | Nightly Cost | Monthly Cost |
|---------|--------|-------------|-------------|
| FitTrack | 2 | $5 | $150 |
| North Shore AI | 3 | $9 | $270 |
| RIA Client Tools | 2 | $5 | $150 |
| Ice Scheduler | 1 | $2 | $60 |
| **Total** | **8** | **$21** | **$630** |

**ROI:** If North Shore AI's medical receptionist agent improves booking rate by 15%, and each booking is worth $150-300 to the practice, one extra appointment per day = $3K-6K/month. $270/month to get there is a no-brainer.

---

## What This Is NOT

- **NOT auto-deploying anything.** Human gate always.
- **NOT touching real customer data.** Synthetic test cases only.
- **NOT training ML models.** No GPU needed. This optimizes prompts via the Claude API.
- **NOT replacing human judgment.** Peter decides what's good enough. The system just proposes and measures.

---

## Relationship to Karpathy's AutoResearch

| Aspect | Karpathy Original | AutoEvolve (ours) |
|--------|-------------------|-------------------|
| What's optimized | Python training code (train.py) | Agent system prompts |
| Metric | val_bpb (lower = better) | Per-project (booking rate, engagement, accuracy) |
| Runtime | 5 min GPU training | 30 sec Claude API call per test case |
| Hardware | H100 GPU | None — runs on Supabase Edge Functions |
| Cost per experiment | GPU compute time | ~$0.06-0.30 Claude API |
| Loop | NEVER STOP | Budget-capped, nightly cron |
| Deploy | Auto-keep on branch | Human approval required |
| Safety | Sandboxed to one file | Safety validator + PII firewall + compliance check |
| Coordination | Single agent (or agenthub) | Single agent per project (multi-agent later via agenthub pattern) |

---

## Future: Multi-Agent via AgentHub Pattern

Karpathy's `agenthub` branch adds multi-agent coordination via a shared commit log and message board. Once AutoEvolve is running single-agent per project, we could:

1. Run multiple agents per project exploring different optimization strategies simultaneously
2. Share learnings across projects (e.g., tone improvements in FitTrack could inform RIA tools)
3. Implement a "research board" where agents post what they've tried, avoiding redundant experiments

**This is Phase 5+ — don't build it until single-agent is proven and generating value.**
