# Diff Intelligence Graph Specification

> **Note:** This is a thinking artifact, not a complete design spec. It captures the core ideas and schema direction for capturing decision traces in AI-assisted software development. Expect iterations when adapting with validations against real-world usage.

**Version:** 0.1.0-draft  
**Status:** Request for Comments  
**Last Updated:** January 2026

---

## Abstract

The Diff Intelligence Graph (DIG) is a specification for capturing decision traces in AI-assisted software development. It connects AI interactions, code changes, rollout strategies, and production outcomes into a queryable graph that enables systematic learning from shipping.

Most organizations log the *what* (PR merged, deploy happened, incident occurred) but not the *why* (developer intent, AI suggestions, rollout rationale, decision reasoning). Those "why" traces are scattered across code comments, dashboards, Slack threads, and incident reviews. They're not queryable, not reusable, not compounding. DIG makes them durable.

This specification defines the core event types, their schemas, and linking conventions. It is intentionally domain-agnostic: domain context is captured as metadata, enabling domain-specific analysis without domain-specific schemas.

A key metric enabled by DIG is **Production Survival Rate (PSR)**: the percentage of changes assisted by a given model that reach full rollout without rollback or incident. PSR can be computed per model, per domain, per change type, and per rollout strategy, enabling contextual model selection and agent evaluation grounded in production reality.

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Design Principles](#2-design-principles)
3. [Core Concepts](#3-core-concepts)
4. [Event Types](#4-event-types)
   - [4.1 Session](#41-session)
   - [4.2 AI Interaction](#42-ai-interaction)
   - [4.3 Change](#43-change)
   - [4.4 Rollout](#44-rollout)
   - [4.5 Outcome](#45-outcome)
   - [4.6 Learning](#46-learning)
5. [Linking Conventions](#5-linking-conventions)
6. [Domain and Work Type Tagging](#6-domain-and-work-type-tagging)
7. [Production Survival Rate (PSR)](#7-production-survival-rate-psr)
8. [Common Fields](#8-common-fields)
9. [Example: Full Trace](#9-example-full-trace)
10. [Query Patterns](#10-query-patterns)
11. [Implementation Notes](#11-implementation-notes)
12. [Extensibility](#12-extensibility)
13. [Appendix: Field Reference](#appendix-field-reference)

---

## 1. Motivation

AI tools accelerate code creation, but most organizations don't systematically learn from what happens after code ships. 

We can usually reconstruct *what* happened:
- PR merged
- Deploy happened  
- Metrics moved
- Incident occurred

But we rarely capture *why*:
- What the developer intended
- What the AI suggested
- What context was used
- Why a rollout strategy was chosen
- What guardrails were expected
- Why someone decided to proceed, pause, or rollback
- What ultimately happened

Those "why" traces are scattered across code comments, Slack threads, dashboards, and incident reviews. They're not queryable. They're not reusable. They don't compound.

**So organizations repeat the same mistakes, just faster.**

The Diff Intelligence Graph creates a structured system of record for these traces, transforming shipping from "a series of isolated deploys" into a closed-loop learning system.

### What DIG Enables

Once you have this graph, you can answer questions that feel impossible today:

- What's the production survival rate for AI-generated code by model?
- Which rollout strategies reduce rollback rates for reranking changes?
- What patterns predict incidents before they happen?
- Which guardrails catch real issues vs. create noise?
- Which model is safest for ranking model changes? For UI refactors? For data pipelines?

### Why This Matters for Agentic AI

Agentic AI assistants are getting better at autonomous actions: writing code, opening PRs, even shipping changes. But they still lack the most important training signal: **outcomes**.

Today's agents optimize for task completion, user approval, and benchmark performance. They don't optimize for production stability, rollback rates, incident risk, or long-term code health.

**Agents don't need more tokens. They need outcomes.**

A Diff Intelligence Graph provides the grounding signal agents need:
- Which recommendations led to safe rollouts?
- Which patterns caused regressions?
- Which mitigations actually reduced incident risk?
- Which code changes survived production vs. got rolled back?

Without outcome data, agents are flying blind. With it, they can learn what actually works, not in benchmarks, but in your production environment.

### DIG Benefits Multiple Consumers

DIG is valuable **even if you never deploy agents**. It's fundamentally about organizational learning from production. Agents are one beneficiary, not the reason DIG exists.

| Consumer | What DIG Provides |
|----------|-------------------|
| **Engineers** | "Changes like this caused issues before" warnings at code-time |
| **Platform teams** | Data-driven rollout policies, guardrail tuning |
| **Leadership** | PSR trends, release quality metrics, DORA-adjacent insights |
| **CI/CD systems** | Automated risk scoring, recommended rollout strategies |
| **Agents** | Outcome grounding for self-improvement |

The common thread: **systematic learning from production, made queryable and reusable**.

---

## 2. Design Principles

### 2.1 Append-Only Events

All events are immutable once written. This provides an audit trail and enables replay. Updates are modeled as new events that reference previous ones.

Note: Some events include an `updated_at` field. This represents the latest projection state for convenience, not mutation of the underlying event store. Implementations may compute `updated_at` from the most recent related event.

### 2.2 Links Over Nesting

Events reference each other via IDs rather than nesting. This enables flexible traversal and avoids duplication.

### 2.3 Domain-Agnostic Core

The schema captures the *shape* of decision traces universally. Domain (backend, frontend, data, ML) is metadata, not structure. This enables cross-domain analysis and avoids schema fragmentation.

### 2.4 Extensible Metadata

Every event includes a `metadata` field for organization-specific or domain-specific data. The core schema remains stable while extensions evolve.

### 2.5 Correlation by Default

Every event that relates to a code change carries a `change_id`. This is the primary correlation key that links AI traces → code → rollout → outcome.

Note: `session` and `ai_interaction` events occur *before* a change exists, so they don't carry `change_id`. Instead, the `change` event back-references them via `session_ids` and `ai_interaction_ids`. This reverse linking is intentional.

### 2.6 Human-Readable IDs

IDs are prefixed by type (e.g., `session_abc123`, `change_def456`) to make logs and queries more readable.

---

## 3. Core Concepts

### The Graph Structure

```
┌─────────┐
│ Session │ (development context grouping)
└────┬────┘
     │ contains 1..n
     ▼
┌────────────────┐
│ AI Interaction │ (model suggestions, human decisions)
└───────┬────────┘
        │ produces 0..n
        ▼
   ┌────────┐
   │ Change │ (code diff, PR, commit)
   └───┬────┘
       │ deployed via 1..n
       ▼
  ┌─────────┐
  │ Rollout │ (deployment strategy, guardrails)
  └────┬────┘
       │ produces 1..n
       ▼
  ┌─────────┐
  │ Outcome │ (telemetry, incidents, decisions)
  └────┬────┘
       │ aggregates into
       ▼
  ┌──────────┐
  │ Learning │ (patterns, recommendations)
  └──────────┘
```

### Event Lifecycle

1. **Session** begins when a developer starts working on a task
2. **AI Interactions** are logged as the developer uses AI tools
3. **Change** is created when code is committed/PR opened
4. **Rollout** is created when the change is deployed
5. **Outcome** events stream in as production telemetry arrives
6. **Learning** is derived from aggregated outcomes

### Minimum Viable DIG

The full spec describes six event types, but **you don't need all of them to start**.

The smallest useful DIG captures just three things:

```
change → outcome → survival
```

With just these fields, you can compute PSR:

```json
// Minimal change event
{
  "id": "change_abc123",
  "type": "change",
  "version": "0.1.0",
  "created_at": "2026-01-15T11:00:00Z",
  "source_control": {
    "repository": "acme/feed-service",
    "commit_sha": "abc123"
  },
  "tags": {
    "domain": "ml",
    "work_type": "refactoring"
  }
}

// Minimal outcome event
{
  "id": "outcome_xyz789",
  "type": "outcome",
  "version": "0.1.0",
  "created_at": "2026-01-15T18:00:00Z",
  "change_id": "change_abc123",
  "survival": {
    "survived": false,
    "rolled_back": true
  }
}
```

**That's it.** From this minimal data, you can already answer:
- What's our overall PSR?
- Which domains have the highest rollback rates?
- Are refactors riskier than bugfixes?

Even in MV-DIG, treat these as **append-only events in a single event store**, keyed by `change_id`. This keeps you aligned with the full spec's design principles (2.1, 2.5) and makes upgrading seamless.

**Layer additional events as you mature:**

| Stage | Add | Enables |
|-------|-----|---------|
| **Start here** | `change` + `outcome` | PSR by domain, work type, team |
| **+Rollout context** | `rollout` | PSR by strategy, guardrail effectiveness |
| **+AI tracing** | `ai_interaction` | PSR by model, acceptance rate correlation |
| **+Learning loops** | `learning` | IDE-time warnings, pattern reuse |
| **+Session context** | `session` | Intent capture, full decision traces |

Start capturing outcomes now. The rest is a roadmap, not a prerequisite.

---

## 4. Event Types

### 4.1 Session

A session groups related AI interactions during a development task. It captures the human intent and context available to the developer/AI.

```json
{
  "id": "session_a1b2c3d4",
  "type": "session",
  "version": "0.1.0",
  "created_at": "2026-01-15T10:30:00Z",
  "updated_at": "2026-01-15T11:45:00Z",
  
  "developer": {
    "id": "user_xyz789",
    "anonymous_id": "anon_hash_abc"
  },
  
  "intent": {
    "description": "Add item deduplication to the reranking service",
    "source": "manual",
    "ticket_id": "JIRA-1234",
    "ticket_url": "https://jira.example.com/JIRA-1234"
  },
  
  "context": {
    "repository": "acme/reranking-service",
    "branch": "feature/item-deduplication",
    "files_open": ["src/reranking/deduplication.py", "src/reranking/scorer.py"],
    "tools_active": ["vscode", "claude-code"]
  },
  
  "tags": {
    "domain": "ml",
    "team": "feed-ranking",
    "priority": "high"
  },
  
  "metadata": {}
}
```

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier, prefixed with `session_` |
| `type` | string | Always `"session"` |
| `version` | string | Schema version |
| `created_at` | ISO 8601 | When the session started |

#### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `updated_at` | ISO 8601 | Last activity timestamp |
| `developer.id` | string | Developer identifier (may be omitted for privacy) |
| `developer.anonymous_id` | string | Hashed/anonymized identifier |
| `intent.description` | string | What the developer is trying to accomplish |
| `intent.source` | enum | `manual`, `ticket`, `incident`, `agent` |
| `intent.ticket_id` | string | External ticket reference |
| `context.repository` | string | Repository identifier |
| `context.branch` | string | Working branch |
| `context.files_open` | array[string] | Files in context |
| `context.tools_active` | array[string] | Active development tools |
| `tags` | object | Arbitrary key-value tags for filtering |
| `metadata` | object | Extension point for custom data |

---

### 4.2 AI Interaction

An AI interaction captures a request-response cycle with an AI model, including what the human accepted, rejected, or modified.

```json
{
  "id": "ai_e5f6g7h8",
  "type": "ai_interaction",
  "version": "0.1.0",
  "created_at": "2026-01-15T10:35:00Z",
  
  "session_id": "session_a1b2c3d4",
  
  "request": {
    "prompt_hash": "sha256_abc123",
    "prompt_summary": "Implement sliding-window item deduplication for reranking",
    "context_files": ["src/reranking/scorer.py"],
    "context_tokens": 4500
  },
  
  "response": {
    "model_id": "claude-sonnet-4-20250514",
    "model_provider": "anthropic",
    "response_hash": "sha256_def456",
    "response_summary": "Generated DeduplicationFilter class with configurable time window and similarity threshold",
    "tokens_generated": 850,
    "latency_ms": 2340,
    "finish_reason": "complete"
  },
  
  "human_action": {
    "action": "partial_accept",
    "accepted_pct": 0.75,
    "modifications": "Adjusted similarity threshold, added logging",
    "time_to_decision_ms": 45000
  },
  
  "tags": {
    "domain": "ml",
    "interaction_type": "implementation",
    "language": "python"
  },
  
  "metadata": {
    "ide": "vscode",
    "extension_version": "1.2.3"
  }
}
```

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier, prefixed with `ai_` |
| `type` | string | Always `"ai_interaction"` |
| `version` | string | Schema version |
| `created_at` | ISO 8601 | When the interaction occurred |
| `response.model_id` | string | Model identifier |

#### Human Action Values

| Value | Description |
|-------|-------------|
| `accept` | Accepted as-is |
| `partial_accept` | Accepted with modifications |
| `reject` | Rejected entirely |
| `ignore` | No action taken |
| `regenerate` | Requested new response |

#### Privacy Considerations

- `prompt_hash` and `response_hash` allow correlation without storing full content
- `prompt_summary` and `response_summary` are optional human-readable descriptions
- Full prompts/responses can be stored separately and linked if desired

---

### 4.3 Change

A change represents a code modification, typically a commit or pull request. It links AI interactions to version control.

```json
{
  "id": "change_i9j0k1l2",
  "type": "change",
  "version": "0.1.0",
  "created_at": "2026-01-15T11:00:00Z",
  "updated_at": "2026-01-15T14:30:00Z",
  
  "session_ids": ["session_a1b2c3d4"],
  "ai_interaction_ids": ["ai_e5f6g7h8", "ai_m3n4o5p6"],
  
  "source_control": {
    "provider": "github",
    "repository": "acme/reranking-service",
    "commit_sha": "abc123def456",
    "pr_number": 892,
    "pr_url": "https://github.com/acme/reranking-service/pull/892",
    "base_branch": "main",
    "head_branch": "feature/item-deduplication"
  },
  
  "diff_stats": {
    "files_changed": 3,
    "lines_added": 145,
    "lines_removed": 12,
    "files": [
      {"path": "src/reranking/deduplication.py", "added": 120, "removed": 0},
      {"path": "src/reranking/scorer.py", "added": 15, "removed": 8},
      {"path": "tests/test_deduplication.py", "added": 10, "removed": 4}
    ]
  },
  
  "authorship": {
    "human_authored_pct": 0.25,
    "ai_authored_pct": 0.75,
    "ai_models_used": ["claude-sonnet-4-20250514"]
  },
  
  "classification": {
    "change_type": "feature",
    "risk_level": "medium",
    "risk_signals": ["ranking_impact", "feed_diversity_change"],
    "estimated_blast_radius": "feed_reranking"
  },
  
  "review": {
    "reviewers": ["user_abc", "user_def"],
    "approvals": 2,
    "review_comments": 3,
    "time_to_merge_hours": 4.5
  },
  
  "tags": {
    "domain": "ml",
    "work_type": "feature",
    "team": "feed-ranking",
    "services_affected": ["reranking-service"]
  },
  
  "metadata": {}
}
```

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier, prefixed with `change_` |
| `type` | string | Always `"change"` |
| `version` | string | Schema version |
| `created_at` | ISO 8601 | When the change was created |
| `source_control.repository` | string | Repository identifier |
| `source_control.commit_sha` | string | Commit or merge commit SHA |

#### Change Type Values

| Value | Description |
|-------|-------------|
| `feature` | New functionality |
| `bugfix` | Bug fix |
| `refactor` | Code restructuring without behavior change |
| `config` | Configuration change |
| `dependency` | Dependency update |
| `hotfix` | Emergency production fix |
| `rollback` | Reverting a previous change |

#### Risk Level Values

| Value | Description |
|-------|-------------|
| `low` | Minimal risk, well-tested path |
| `medium` | Moderate risk, standard review |
| `high` | High risk, requires extra scrutiny |
| `critical` | Critical path, requires sign-off |

---

### 4.4 Rollout

A rollout captures how a change was deployed: the strategy, guardrails, and progression.

```json
{
  "id": "rollout_q7r8s9t0",
  "type": "rollout",
  "version": "0.1.0",
  "created_at": "2026-01-15T15:00:00Z",
  "updated_at": "2026-01-15T16:30:00Z",
  
  "change_id": "change_i9j0k1l2",
  
  "deployment": {
    "environment": "production",
    "target": "reranking-service",
    "deploy_tool": "argocd",
    "deploy_id": "deploy_uvw123",
    "artifact_version": "v1.2.3"
  },
  
  "strategy": {
    "type": "progressive",
    "stages": [
      {"pct": 1, "duration_minutes": 15},
      {"pct": 5, "duration_minutes": 30},
      {"pct": 25, "duration_minutes": 60},
      {"pct": 100, "duration_minutes": null}
    ],
    "auto_promote": true,
    "auto_rollback": true
  },
  
  "guardrails": {
    "metrics": [
      {"name": "error_rate", "threshold": 0.01, "comparison": "lt"},
      {"name": "p99_latency_ms", "threshold": 200, "comparison": "lt"},
      {"name": "engagement_rate", "threshold": 0.95, "comparison": "gte_baseline"},
      {"name": "dedupe_rate", "threshold": 0.30, "comparison": "lt"}
    ],
    "alerts": ["pagerduty:feed-ranking-oncall"],
    "manual_approval_required": false
  },
  
  "progression": [
    {
      "timestamp": "2026-01-15T15:00:00Z",
      "stage_pct": 1,
      "status": "healthy",
      "metrics_snapshot": {"error_rate": 0.002, "p99_latency_ms": 45, "dedupe_rate": 0.12}
    },
    {
      "timestamp": "2026-01-15T15:15:00Z",
      "stage_pct": 5,
      "status": "healthy",
      "metrics_snapshot": {"error_rate": 0.003, "p99_latency_ms": 48, "dedupe_rate": 0.14}
    },
    {
      "timestamp": "2026-01-15T15:45:00Z",
      "stage_pct": 25,
      "status": "healthy",
      "metrics_snapshot": {"error_rate": 0.002, "p99_latency_ms": 46, "dedupe_rate": 0.13}
    },
    {
      "timestamp": "2026-01-15T16:30:00Z",
      "stage_pct": 100,
      "status": "complete",
      "metrics_snapshot": {"error_rate": 0.002, "p99_latency_ms": 47, "dedupe_rate": 0.13}
    }
  ],
  
  "final_status": "success",
  
  "tags": {
    "domain": "ml",
    "team": "feed-ranking"
  },
  
  "metadata": {}
}
```

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier, prefixed with `rollout_` |
| `type` | string | Always `"rollout"` |
| `version` | string | Schema version |
| `created_at` | ISO 8601 | When the rollout started |
| `change_id` | string | The change being deployed |
| `deployment.environment` | string | Target environment |
| `final_status` | enum | Terminal status |

#### Strategy Type Values

| Value | Description |
|-------|-------------|
| `all_at_once` | Immediate full deployment |
| `progressive` | Staged percentage rollout |
| `canary` | Canary deployment with comparison |
| `blue_green` | Blue-green deployment |
| `feature_flag` | Feature flag controlled |

#### Final Status Values

| Value | Description |
|-------|-------------|
| `success` | Fully deployed successfully |
| `rolled_back` | Rolled back due to issues |
| `paused` | Paused mid-rollout |
| `failed` | Deployment failed |
| `cancelled` | Manually cancelled |

---

### 4.5 Outcome

An outcome captures what happened after deployment: telemetry changes, incidents, and human decisions.

**Note on telemetry overlap:** Rollout `progression.metrics_snapshot` captures point-in-time metrics used for promote/rollback decisions. Outcome `telemetry` captures windowed post-hoc analysis used for learning. Both are valuable: one is the decision trace, the other is the learning signal.

**Note on multiple rollouts:** A change may have multiple rollout attempts (e.g., initial rollout, rollback, re-deploy). Each rollout attempt gets its own outcome event. For PSR computation, use the outcome from the final production rollout attempt.

```json
{
  "id": "outcome_u1v2w3x4",
  "type": "outcome",
  "version": "0.1.0",
  "created_at": "2026-01-15T18:00:00Z",
  
  "change_id": "change_i9j0k1l2",
  "rollout_id": "rollout_q7r8s9t0",
  
  "observation_window": {
    "start": "2026-01-15T15:00:00Z",
    "end": "2026-01-15T18:00:00Z",
    "duration_hours": 3
  },
  
  "telemetry": {
    "baseline_period": {
      "start": "2026-01-14T15:00:00Z",
      "end": "2026-01-15T15:00:00Z"
    },
    "metrics": [
      {
        "name": "error_rate",
        "baseline": 0.002,
        "observed": 0.0025,
        "change_pct": 25,
        "significant": false
      },
      {
        "name": "p99_latency_ms",
        "baseline": 42,
        "observed": 47,
        "change_pct": 11.9,
        "significant": false
      },
      {
        "name": "engagement_rate",
        "baseline": 0.156,
        "observed": 0.162,
        "change_pct": 3.8,
        "significant": true
      },
      {
        "name": "dedupe_rate",
        "baseline": 0.0,
        "observed": 0.13,
        "change_pct": null,
        "significant": true
      }
    ]
  },
  
  "incidents": [],
  
  "decisions": [
    {
      "timestamp": "2026-01-15T16:30:00Z",
      "decision": "proceed",
      "actor": "automation",
      "rationale": "All guardrails passed, auto-promoted to 100%"
    }
  ],
  
  "survival": {
    "survived": true,
    "rolled_back": false,
    "time_to_rollback_minutes": null,
    "hotfix_required": false
  },
  
  "tags": {
    "domain": "ml",
    "team": "feed-ranking"
  },
  
  "metadata": {}
}
```

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier, prefixed with `outcome_` |
| `type` | string | Always `"outcome"` |
| `version` | string | Schema version |
| `created_at` | ISO 8601 | When the outcome was recorded |
| `change_id` | string | The change this outcome relates to |
| `survival.survived` | boolean | Whether the change survived production |

#### Telemetry Metrics

- `change_pct`: Relative percentage change, computed as `100 * (observed - baseline) / baseline` when baseline > 0. Use `null` when baseline is 0 or undefined.
- `significant`: Boolean indicating statistical significance. Computation method is org-specific (e.g., t-test, bootstrap CI, sequential testing). Consider documenting your method in metadata.

#### Decision Values

| Value | Description |
|-------|-------------|
| `proceed` | Continue with rollout |
| `pause` | Pause rollout for investigation |
| `rollback` | Rollback the change |
| `hotfix` | Apply a hotfix |
| `investigate` | Flag for investigation |

#### Incident Object

When incidents occur, they are captured as:

```json
{
  "incident_id": "inc_abc123",
  "severity": "sev2",
  "title": "Feed engagement drop due to over-aggressive deduplication",
  "detected_at": "2026-01-15T16:45:00Z",
  "resolved_at": "2026-01-15T17:30:00Z",
  "duration_minutes": 45,
  "attributed_to_change": true,
  "root_cause": "dedupe_threshold_too_strict",
  "mitigation_applied": "rollback",
  "incident_url": "https://pagerduty.example.com/inc_abc123"
}
```

The `root_cause` field enables pattern detection across incidents (e.g., "deduplication threshold issues are 3x more common in reranking changes").

---

### 4.6 Learning

A learning is derived from aggregated outcomes. It represents a pattern or recommendation that feeds forward into future development.

```json
{
  "id": "learning_y5z6a7b8",
  "type": "learning",
  "version": "0.1.0",
  "created_at": "2026-01-20T09:00:00Z",
  
  "derived_from": {
    "outcome_ids": ["outcome_u1v2w3x4", "outcome_c2d3e4f5", "..."],
    "change_ids": ["change_i9j0k1l2", "change_g3h4i5j6", "..."],
    "sample_size": 47,
    "observation_period": {
      "start": "2025-10-01T00:00:00Z",
      "end": "2026-01-20T00:00:00Z"
    }
  },
  
  "pattern": {
    "name": "reranking_dedupe_engagement_risk",
    "description": "Deduplication changes in reranking have 3x rollback rate when dedupe_rate guardrail isn't monitored",
    "conditions": {
      "tags": {
        "domain": "ml",
        "services_affected": ["reranking-service"]
      },
      "classification": {
        "change_type": "feature"
      }
    },
    "observation": {
      "metric": "rollback_rate",
      "without_mitigation": 0.15,
      "with_mitigation": 0.05,
      "mitigation": "dedupe_rate_guardrail"
    },
    "confidence": 0.87,
    "statistical_significance": 0.02
  },
  
  "recommendation": {
    "trigger_conditions": "change.tags.services_affected CONTAINS 'reranking-service' AND change.diff CONTAINS 'dedupe'",
    "action": "suggest_dedupe_guardrails",
    "severity": "warning",
    "message": "Reranking deduplication changes historically benefit from progressive rollout with dedupe_rate and engagement_rate guardrails. Similar changes without dedupe_rate monitoring had 15% rollback rate vs 5% with monitoring."
  },
  
  "model_insights": [
    {
      "model_id": "claude-sonnet-4-20250514",
      "survival_rate": 0.94,
      "sample_size": 23,
      "comparison": {
        "vs_human_only": 0.02,
        "vs_other_models": 0.05
      }
    }
  ],
  
  "tags": {
    "domain": "ml",
    "category": "guardrail_recommendation"
  },
  
  "metadata": {}
}
```

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier, prefixed with `learning_` |
| `type` | string | Always `"learning"` |
| `version` | string | Schema version |
| `created_at` | ISO 8601 | When the learning was generated |
| `derived_from.sample_size` | integer | Number of outcomes analyzed |
| `pattern.name` | string | Machine-readable pattern name |
| `pattern.description` | string | Human-readable description |
| `pattern.confidence` | float | Confidence score 0-1 |

#### Recommendation Severity Values

| Value | Description |
|-------|-------------|
| `info` | Informational, no action required |
| `warning` | Suggested action, proceed with awareness |
| `error` | Strong recommendation, requires justification to override |
| `block` | Blocks action, requires explicit override |

#### Note on `trigger_conditions`

The `trigger_conditions` field may reference non-canonical signals (e.g., diff heuristics, file path patterns) and should be treated as best-effort hints, not guaranteed matches. Canonical applicability must be expressible via `pattern.conditions`. Consuming systems should use `pattern.conditions` for reliable matching and `trigger_conditions` for additional filtering or ranking.

#### Enforcement Model

DIG does not enforce behavior. It is a system of record, not a policy engine. Enforcement (blocking PRs, requiring approvals, gating deploys) is delegated to consuming systems such as CI pipelines, CD tools, and IDEs. The `severity` field expresses intent, not capability.

---

## 5. Linking Conventions

### 5.1 ID Format

All IDs follow the pattern: `{prefix}_{random_id}`

| Event Type | ID Prefix | Example |
|------------|-----------|---------|
| `session` | `session_` | `session_a1b2c3d4` |
| `ai_interaction` | `ai_` | `ai_e5f6g7h8` |
| `change` | `change_` | `change_i9j0k1l2` |
| `rollout` | `rollout_` | `rollout_q7r8s9t0` |
| `outcome` | `outcome_` | `outcome_u1v2w3x4` |
| `learning` | `learning_` | `learning_y5z6a7b8` |

Note: `ai_interaction` uses the shortened `ai_` prefix for brevity.

Random portion should be URL-safe and at least 8 characters.

### 5.2 Reference Fields

Events reference other events via `*_id` (single) or `*_ids` (multiple):

```
session
    ↑
    └── ai_interaction.session_id
            ↑
            └── change.ai_interaction_ids[]
                    ↑
                    ├── rollout.change_id
                    └── outcome.change_id
                            ↑
                            └── learning.derived_from.outcome_ids[]
```

### 5.3 The `change_id` Correlation Key

The `change_id` is the primary correlation key. Any event related to a code change should carry it, enabling queries like:

```sql
-- Find all events related to a change
SELECT * FROM events 
WHERE change_id = 'change_i9j0k1l2' 
   OR id = 'change_i9j0k1l2';
```

---

## 6. Domain and Work Type Tagging

### 6.1 Domain as Metadata

Domains and work types are captured in the `tags` object, not as structural elements:

```json
{
  "tags": {
    "domain": "ml",
    "work_type": "feature",
    "sub_domain": "reranking",
    "language": "python",
    "framework": "tensorflow"
  }
}
```

This enables:
- Cross-domain analysis (e.g., "compare rollback rates across all domains")
- Domain-specific analysis (e.g., "filter to ML domain")
- Work-type analysis (e.g., "refactors have higher rollback rates than bugfixes")
- Custom taxonomies per organization

### 6.2 Suggested Domain Values

| Value | Description |
|-------|-------------|
| `frontend` | Client-side web/mobile |
| `backend` | Server-side services |
| `data` | Data pipelines, ETL |
| `ml` | Machine learning, models |
| `infrastructure` | Infrastructure as code |
| `platform` | Internal platform/tooling |

### 6.3 Suggested Work Type Values

Work types cut across stack layers. A refactor can happen in frontend or backend:

| Value | Description |
|-------|-------------|
| `product_ux_ui` | Interaction flows, state management, accessibility, design systems |
| `systems_architecture` | Distributed systems, concurrency, failure modes, latency optimization |
| `refactoring` | Preserving behavior while changing structure |
| `migration` | Schema evolution, API versioning, data migrations |
| `data_pipeline` | Correctness, idempotency, freshness, quality checks |
| `experimentation` | Metric design, causal reasoning, A/B test implementation |
| `incident_response` | Debugging under pressure, mitigation, root cause analysis |
| `feature` | New functionality end-to-end |
| `bugfix` | Fixing incorrect behavior |
| `config` | Configuration changes |
| `dependency` | Dependency updates |

Organizations should define their own taxonomy as needed. The key insight: **model performance is contextual and horizontal**. A model that excels at greenfield UI might struggle with distributed systems debugging.

### 6.4 Domain-Specific Guardrails

Guardrails in the `rollout` event can be domain-aware:

```json
{
  "guardrails": {
    "metrics": [
      {"name": "error_rate", "threshold": 0.01, "comparison": "lt"},
      {"name": "p99_latency_ms", "threshold": 500, "comparison": "lt"}
    ]
  }
}
```

The *meaning* of metrics differs by domain, but the *structure* is the same:

| Domain | Typical Guardrails |
|--------|-------------------|
| `frontend` | Crash-free sessions, core journey completion, LCP/FID/CLS |
| `backend` | Error budgets, latency percentiles, throughput |
| `ml` | Drift detection, feature health, prediction distribution |
| `data` | Freshness, row counts, schema validation, null rates |

---

## 7. Production Survival Rate (PSR)

### 7.1 Definition

**Production Survival Rate (PSR)** is the percentage of changes that reach full rollout without rollback or incident.

```
PSR = (changes that survived) / (total changes deployed) × 100
```

A change "survives" if:
- It reached 100% rollout
- It was not rolled back within the observation window
- It did not cause an attributed incident

**`outcome.survival.survived` is the canonical PSR label.** The criteria above describe how organizations typically compute this boolean. Your org may have different rules (e.g., different observation windows, different incident attribution). Define your rules clearly before computing PSR.

### 7.2 Why PSR Matters

Today, we choose AI models based on benchmarks, cost, latency, and vibes. But benchmarks measure capability in isolation. They don't measure what happens when AI-generated code hits production.

**The real question is: which model produces changes that survive production?**

That's not a benchmark problem. It's an operational outcomes problem, and every company's production reality is different.

### 7.3 PSR Dimensions

PSR becomes powerful when computed across dimensions:

| Dimension | Example Questions |
|-----------|-------------------|
| **Per model** | Which model has highest PSR overall? |
| **Per domain** | Which model is safest for reranking changes? |
| **Per work type** | Which model produces fewest rollback-prone refactors? |
| **Per change type** | Do bugfixes have higher PSR than features? |
| **Per rollout strategy** | Does progressive rollout improve PSR? |
| **Per team/codebase** | Which model works best for *our* patterns? |

### 7.4 Computing PSR

PSR is computed from `outcome` events:

```sql
SELECT 
  ai.response.model_id,
  c.tags->>'domain' as domain,
  c.tags->>'work_type' as work_type,
  COUNT(*) as total_changes,
  SUM(CASE WHEN o.survival.survived THEN 1 ELSE 0 END) as survived,
  ROUND(
    100.0 * SUM(CASE WHEN o.survival.survived THEN 1 ELSE 0 END) / COUNT(*), 
    2
  ) as psr
FROM ai_interactions ai
JOIN changes c ON ai.id = ANY(c.ai_interaction_ids)
JOIN outcomes o ON c.id = o.change_id
GROUP BY ai.response.model_id, c.tags->>'domain', c.tags->>'work_type'
ORDER BY psr DESC;
```

### 7.5 PSR for Agent Evaluation

For agentic AI, PSR provides the grounding signal that benchmarks can't:

| What Agents Optimize For Today | What They Should Optimize For |
|-------------------------------|------------------------------|
| Task completion | Production stability |
| User approval (thumbs up) | Rollback rates |
| Benchmark performance | Incident risk |
| (none) | Long-term code health |

By connecting agent actions to outcomes via DIG, agents can learn:
- Which of my recommendations led to safe rollouts?
- Which patterns I suggested caused regressions?
- Which mitigations I proposed actually reduced incident risk?

**This is how agents move from "helpful but risky" to "trusted and reliable."**

### 7.6 PSR Caveats

- **Observation window matters**: Define a consistent window (e.g., 7 days post-deploy)
- **Attribution is hard**: Some incidents have multiple contributing changes
- **Sample size**: PSR is only meaningful with sufficient volume
- **Context differs**: A model's PSR in one org may not transfer to another
- **Incident definition is org-specific**: "Incident" can mean pager incident, attributed SLO violation, guardrail-triggered rollback, or experiment regression. Define this for your org before computing PSR.

### 7.7 PSR as a Trust Signal

In Part 1 of the blog series, we argued that **trust is the scarce resource** when code creation becomes cheap:

> "Can I ship this safely? Will it break production?"

PSR and learning confidence are **trust signals**. They operationalize the answer to that question.

- A high PSR for a change type means you've *earned trust* for that class of changes
- A low PSR means the system is telling you: slow down, add guardrails, or change your approach
- Learning confidence scores indicate how much you should trust a pattern recommendation

Trust isn't binary. It's contextual and earned incrementally. DIG makes that trust **measurable and queryable**, which is how organizations move from "we think this is safe" to "we know this is safe, and here's the data."

---

## 8. Common Fields

All events share these fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier with type prefix |
| `type` | string | Yes | Event type name |
| `version` | string | Yes | Schema version for this event |
| `created_at` | ISO 8601 | Yes | Event creation timestamp |
| `updated_at` | ISO 8601 | No | Last update timestamp |
| `tags` | object | No | Arbitrary key-value tags |
| `metadata` | object | No | Extension data |

### 8.1 Timestamps

All timestamps must be ISO 8601 format with timezone:
- `2026-01-15T10:30:00Z` (UTC)
- `2026-01-15T10:30:00-08:00` (with offset)

UTC is preferred for consistency.

### 8.2 Tags vs. Metadata

- **Tags**: Used for filtering and grouping. Values should be simple strings or arrays of strings.
- **Metadata**: Used for extension data. Can contain arbitrary nested structures.

---

## 9. Example: Full Trace

Here's a complete trace showing how events link together:

```
Developer starts working on item deduplication feature
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ SESSION: session_a1b2c3d4                                   │
│ intent: "Add item deduplication to the reranking service"   │
│ context: reranking-service repo, feature/item-deduplication │
└─────────────────────────────────────────────────────────────┘
    │
    │ Developer asks Claude to implement deduplication
    ▼
┌─────────────────────────────────────────────────────────────┐
│ AI_INTERACTION: ai_e5f6g7h8                                 │
│ session_id: session_a1b2c3d4                                │
│ model: claude-sonnet-4-20250514                             │
│ human_action: partial_accept (75% accepted)                 │
└─────────────────────────────────────────────────────────────┘
    │
    │ Developer opens PR
    ▼
┌─────────────────────────────────────────────────────────────┐
│ CHANGE: change_i9j0k1l2                                     │
│ ai_interaction_ids: [ai_e5f6g7h8]                           │
│ authorship: 75% AI, 25% human                               │
│ classification: feature, medium risk                        │
└─────────────────────────────────────────────────────────────┘
    │
    │ PR merged, deploy triggered
    ▼
┌─────────────────────────────────────────────────────────────┐
│ ROLLOUT: rollout_q7r8s9t0                                   │
│ change_id: change_i9j0k1l2                                  │
│ strategy: progressive (1% → 5% → 25% → 100%)                │
│ guardrails: error_rate < 1%, dedupe_rate < 30%              │
│ final_status: success                                       │
└─────────────────────────────────────────────────────────────┘
    │
    │ 3 hours post-deploy observation
    ▼
┌─────────────────────────────────────────────────────────────┐
│ OUTCOME: outcome_u1v2w3x4                                   │
│ change_id: change_i9j0k1l2                                  │
│ rollout_id: rollout_q7r8s9t0                                │
│ survival: true, no incidents                                │
│ telemetry: engagement_rate +3.8% (significant)              │
└─────────────────────────────────────────────────────────────┘
    │
    │ Weekly pattern analysis
    ▼
┌─────────────────────────────────────────────────────────────┐
│ LEARNING: learning_y5z6a7b8                                 │
│ derived_from: 47 outcomes including outcome_u1v2w3x4        │
│ pattern: "reranking dedupe + dedupe_rate guardrail = safer" │
│ recommendation: add dedupe_rate guardrail for reranking     │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 Example: Rollback with Compounding Learning

This example shows how DIG captures a rollback and creates durable learning that prevents future incidents.

```
AI-assisted Feed Recommendation System refactor is developed
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ CHANGE: change_feed_xyz                                     │
│ classification: refactoring, high risk                      │
│ tags: {domain: "ml", work_type: "refactoring",              │
│        services_affected: ["feed-ranking-service"]}         │
│ authorship: 60% AI, 40% human                               │
└─────────────────────────────────────────────────────────────┘
    │
    │ Progressive rollout begins
    ▼
┌─────────────────────────────────────────────────────────────┐
│ ROLLOUT: rollout_feed_456                                   │
│ strategy: progressive (1% → 5% → 25% → 100%)                │
│ guardrails: error_rate < 0.5%, engagement_rate > baseline   │
│ progression:                                                │
│   - 1%: healthy                                             │
│   - 5%: engagement_rate drops 8% → GUARDRAIL TRIGGERED      │
│ final_status: rolled_back                                   │
└─────────────────────────────────────────────────────────────┘
    │
    │ Outcome captured with incident details
    ▼
┌─────────────────────────────────────────────────────────────┐
│ OUTCOME: outcome_feed_789                                   │
│ survival: false, rolled_back: true                          │
│ time_to_rollback_minutes: 22                                │
│ incidents: [{                                               │
│   severity: "sev3",                                         │
│   title: "Feed engagement drop for cold-start users",       │
│   root_cause: "embedding_version_skew"                      │
│ }]                                                          │
│ decisions: [{                                               │
│   decision: "rollback",                                     │
│   actor: "automation",                                      │
│   rationale: "engagement_rate dropped 8% below baseline"    │
│ }]                                                          │
└─────────────────────────────────────────────────────────────┘
    │
    │ Pattern extracted and stored durably
    ▼
┌─────────────────────────────────────────────────────────────┐
│ LEARNING: learning_feed_emb_001                             │
│ pattern: {                                                  │
│   name: "feed_refactor_embedding_skew_risk",                │
│   description: "Feed ranking refactors have 5x rollback     │
│                 rate when embedding versions aren't pinned  │
│                 across training and serving",               │
│   conditions: {work_type: "refactoring",                    │
│                services_affected: ["feed-ranking-service"]} │
│ }                                                           │
│ recommendation: {                                           │
│   action: "suggest_guardrail",                              │
│   severity: "warning",                                      │
│   message: "Feed ranking refactors: add cold_start_engagement│
│             guardrail and verify embedding version parity   │
│             between training and serving"                   │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
    │
    │ NEXT TIME: Learning surfaces at code-time
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Developer starts new feed ranking refactor...               │
│                                                             │
│ IDE WARNING: "Similar feed ranking refactors had 5x         │
│ rollback rate. Recommended: add cold_start_engagement       │
│ guardrail, verify embedding version parity between          │
│ training and serving. See: learning_feed_emb_001"           │
│                                                             │
│ Suggested rollout: progressive with extra monitoring        │
│ Suggested guardrails: cold_start_engagement, embedding_drift│
└─────────────────────────────────────────────────────────────┘
```

**This is compounding learning.** The incident isn't lost in Slack threads and postmortem docs. It's captured in the graph and surfaces automatically when similar changes appear.

---

## 10. Query Patterns

The graph structure enables queries that are impossible with siloed logs:

### 10.1 Production Survival Rate by Model

```sql
SELECT 
  ai.response.model_id,
  COUNT(*) as total_changes,
  SUM(CASE WHEN o.survival.survived THEN 1 ELSE 0 END) as survived,
  ROUND(100.0 * AVG(CASE WHEN o.survival.survived THEN 1.0 ELSE 0.0 END), 2) as psr
FROM ai_interactions ai
JOIN changes c ON ai.id = ANY(c.ai_interaction_ids)
JOIN outcomes o ON c.id = o.change_id
GROUP BY ai.response.model_id
ORDER BY psr DESC;
```

### 10.2 PSR by Model and Work Type

Which model is best for refactoring vs. new features vs. incident response?

```sql
SELECT 
  ai.response.model_id,
  c.tags->>'work_type' as work_type,
  COUNT(*) as total,
  ROUND(100.0 * AVG(CASE WHEN o.survival.survived THEN 1.0 ELSE 0.0 END), 2) as psr
FROM ai_interactions ai
JOIN changes c ON ai.id = ANY(c.ai_interaction_ids)
JOIN outcomes o ON c.id = o.change_id
GROUP BY ai.response.model_id, c.tags->>'work_type'
HAVING COUNT(*) >= 10  -- minimum sample size
ORDER BY work_type, psr DESC;
```

### 10.3 Rollback Rate by Domain and Change Type

```sql
SELECT 
  c.tags->>'domain' as domain,
  c.classification.change_type,
  COUNT(*) as total,
  SUM(CASE WHEN o.survival.rolled_back THEN 1 ELSE 0 END) as rollbacks,
  ROUND(100.0 * AVG(CASE WHEN o.survival.rolled_back THEN 1.0 ELSE 0.0 END), 2) as rollback_rate
FROM changes c
JOIN outcomes o ON c.id = o.change_id
GROUP BY c.tags->>'domain', c.classification.change_type
ORDER BY rollback_rate DESC;
```

### 10.4 Rollout Strategy Effectiveness

Does progressive rollout actually improve PSR?

```sql
SELECT 
  r.strategy.type,
  COUNT(*) as total,
  ROUND(100.0 * AVG(CASE WHEN o.survival.survived THEN 1.0 ELSE 0.0 END), 2) as psr,
  AVG(COALESCE(o.survival.time_to_rollback_minutes, 0)) as avg_time_to_rollback
FROM rollouts r
JOIN outcomes o ON r.change_id = o.change_id
GROUP BY r.strategy.type;
```

### 10.5 AI Acceptance Rate to PSR Correlation

Do changes where humans heavily modify AI suggestions survive better?

```sql
SELECT 
  CASE 
    WHEN ai.human_action.accepted_pct < 0.5 THEN 'low_accept (<50%)'
    WHEN ai.human_action.accepted_pct < 0.8 THEN 'medium_accept (50-80%)'
    ELSE 'high_accept (>80%)'
  END as acceptance_bucket,
  COUNT(*) as total,
  ROUND(100.0 * AVG(CASE WHEN o.survival.survived THEN 1.0 ELSE 0.0 END), 2) as psr
FROM ai_interactions ai
JOIN changes c ON ai.id = ANY(c.ai_interaction_ids)
JOIN outcomes o ON c.id = o.change_id
WHERE ai.human_action.action IN ('accept', 'partial_accept')
GROUP BY acceptance_bucket
ORDER BY psr DESC;
```

### 10.6 Learning Pattern Effectiveness

Which learned patterns actually reduce incidents?

```sql
SELECT 
  l.pattern.name,
  l.recommendation.action,
  COUNT(DISTINCT c.id) as changes_with_pattern,
  ROUND(100.0 * AVG(CASE WHEN o.survival.survived THEN 1.0 ELSE 0.0 END), 2) as psr_with_mitigation
FROM learnings l
JOIN changes c ON c.tags @> l.pattern.conditions::jsonb
JOIN outcomes o ON c.id = o.change_id
WHERE c.created_at > l.created_at  -- changes after learning was created
GROUP BY l.pattern.name, l.recommendation.action
ORDER BY psr_with_mitigation DESC;
```

---

## 11. Implementation Notes

### 11.1 Storage

DIG is storage-agnostic. The spec defines event shapes, not how to persist them. Common approaches include:
- Document stores with JSONB (PostgreSQL, MongoDB)
- Event stores / append-only logs
- Analytics warehouses for query-heavy workloads

Also, the query examples in this spec are illustrative. Adapt to your storage model.

### 11.2 Privacy Considerations

- `developer.id` can be omitted or hashed
- `prompt_hash`/`response_hash` enable correlation without content storage
- Consider data retention policies for each event type
- Learning events can be derived from anonymized/aggregated data

### 11.3 Schema Evolution

The `version` field enables backward compatibility:
- New optional fields: always safe
- New required fields: bump version
- Removed fields: deprecate first

### 11.4 Edge Cases to Define

Organizations should establish clear rules for these scenarios before computing PSR:

**What counts as "deployed"?**
- Merged to main? Shipped to production? Reached a percentage threshold? Feature flag enabled?

**Multiple changes in one deploy artifact**
- A single artifact version may include multiple PRs. Decide how to attribute outcomes (e.g., all changes share the outcome, or use more granular deploy tracking).

**Incidents with multiple contributing changes**
- Allow incidents to reference multiple `change_id`s, or define primary vs. contributing attribution.

**Rollbacks as changes**
- `change_type: rollback` represents a revert PR. `rollout.final_status: rolled_back` represents an operational rollback action. Decide how these interact in your PSR computation.

**Non-code changes**
- Config changes, feature flag flips, ML model weight updates, and schema migrations may not have traditional source control. Consider making `source_control` optional or allowing alternative change sources.

---

## 12. Extensibility

### 12.1 Custom Event Types

Organizations can define custom event types following the same patterns:

```json
{
  "id": "custom_xyz123",
  "type": "custom:security_scan",
  "version": "0.1.0",
  "created_at": "2026-01-15T12:00:00Z",
  "change_id": "change_i9j0k1l2",
  
  "scan_results": {
    "vulnerabilities_found": 0,
    "scan_tool": "snyk",
    "scan_duration_seconds": 45
  },
  
  "tags": {},
  "metadata": {}
}
```

Custom types should:
- Use `custom:` prefix in the type field
- Include `change_id` if related to a change
- Follow the common fields pattern

### 12.2 Metadata Extensions

The `metadata` field is the primary extension point:

```json
{
  "metadata": {
    "org:cost_center": "eng-platform",
    "org:compliance_review": true,
    "integration:datadog_trace_id": "abc123"
  }
}
```

Use namespaced keys to avoid collisions.

---

## Appendix: Field Reference

### Event Type Summary

| Type | Purpose | Key Links |
|------|---------|-----------|
| `session` | Groups AI interactions for a task | (none) |
| `ai_interaction` | Captures AI request/response cycle | → session |
| `change` | Represents code modification | → ai_interactions, session |
| `rollout` | Captures deployment strategy | → change |
| `outcome` | Records production impact | → change, rollout |
| `learning` | Derived patterns and recommendations | → outcomes, changes |

### Enum Reference

**human_action.action**: `accept`, `partial_accept`, `reject`, `ignore`, `regenerate`

**change.classification.change_type**: `feature`, `bugfix`, `refactor`, `config`, `dependency`, `hotfix`, `rollback`

**change.classification.risk_level**: `low`, `medium`, `high`, `critical`

**rollout.strategy.type**: `all_at_once`, `progressive`, `canary`, `blue_green`, `feature_flag`

**rollout.final_status**: `success`, `rolled_back`, `paused`, `failed`, `cancelled`

**outcome.decisions[].decision**: `proceed`, `pause`, `rollback`, `hotfix`, `investigate`

**learning.recommendation.severity**: `info`, `warning`, `error`, `block`

---

## Changelog

### 0.1.0-draft (January 2026)

- Initial draft specification
- Core event types: session, ai_interaction, change, rollout, outcome, learning
- Domain-agnostic design with tag-based domain classification

---

## License

This specification is released under [CC-BY-SA-4.0].

---

## Contributing

I welcome feedback on this specification. Please open an issue or submit a pull request with:

- Schema clarifications or corrections
- New use cases that aren't well-served by the current design
- Implementation experience reports
- Proposed extensions

The schema is the product. Getting it right matters more than shipping fast.
