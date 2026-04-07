# Jupiter Platform — Design Document

**Version:** 1.0 **Date:** April 2025 **Audience:** Engineering team, new team members, architects

______________________________________________________________________

## Table of Contents

01. [Platform Overview](#1-platform-overview)
02. [Architecture Overview](#2-architecture-overview)
03. [Data Model](#3-data-model)
04. [Authentication & Authorization](#4-authentication--authorization)
05. [Pipeline System](#5-pipeline-system)
06. [Conversation Analysis Engine](#6-conversation-analysis-engine)
07. [Policy Layer (Signal Tuning)](#7-policy-layer-signal-tuning)
08. [Aggregation Pipeline](#8-aggregation-pipeline)
09. [Smart Actions & Workflow Automation](#9-smart-actions--workflow-automation)
10. [Platform-Specific Pipelines](#10-platform-specific-pipelines)
11. [AI Agents & Question Routing](#11-ai-agents--question-routing)
12. [Integrations Hub](#12-integrations-hub)
13. [Background Jobs & Scheduling](#13-background-jobs--scheduling)
14. [Frontend Architecture](#14-frontend-architecture)
15. [API Reference](#15-api-reference)
16. [LLM Prompt System](#16-llm-prompt-system)
17. [Billing & Usage Tracking](#17-billing--usage-tracking)
18. [Infrastructure & DevOps](#18-infrastructure--devops)

______________________________________________________________________

## 1. Platform Overview

Jupiter (PrimeOrbit) is a full-stack AI-powered customer engagement and conversation analysis
platform. It ingests conversations from multiple sources (Slack, Discord, Discourse, HackerNews,
WhatsApp, Google Chat, email, images, audio), analyzes them for sentiment, emotion, topics, intent,
and risk, and provides actionable intelligence through dashboards, alerts, and automated workflows.

### Core Capabilities

- **Multi-source conversation ingestion** — Slack, Discord, Discourse, HackerNews, WhatsApp, Google
  Chat, email, images, audio files
- **AI-powered signal extraction** — Sentiment, emotion, topics, intent, risk classification using
  ML models and LLM fallbacks
- **Policy layer (signal tuning)** — User overrides, organizational rules, and auto-generated
  suggestions to correct and refine AI outputs
- **Aggregation pipeline** — 7-stage pipeline that builds metrics, detects anomalies, generates
  action items, and learns playbooks
- **Smart actions** — Automated workflow execution based on detected signals, with 30+ integrations
- **Real-time intelligence** — Webhook subscribers for alerts and notifications
- **Question routing** — AI-powered routing of questions to the right team or individual
- **Analytics dashboards** — Unified analytics with trend charts, breakdowns, and filtering

### Technology Stack

| Layer           | Technology                                                             |
| --------------- | ---------------------------------------------------------------------- |
| Backend         | Python 3.13, FastAPI, SQLModel, Alembic                                |
| Frontend        | React 19, TypeScript, Vite, Tailwind CSS 4, Shadcn/ui                  |
| Database        | PostgreSQL                                                             |
| ML Models       | HuggingFace Transformers (sentiment, emotion, risk)                    |
| LLM             | Azure OpenAI (GPT-4o, GPT-4o-mini), Claude (Haiku for vision/fallback) |
| Job Scheduler   | APScheduler with SQLAlchemy job store                                  |
| Email           | SendGrid                                                               |
| Payments        | Stripe                                                                 |
| Frontend Server | Express (SSR proxy, token management)                                  |

______________________________________________________________________

## 2. Architecture Overview

### High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Frontend (React 19)                         │
│  Pages · Components · Hooks · Context · Routes · API Client         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ /backend/* proxy
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Express Server (Node.js)                          │
│  Proxy middleware · Token refresh · Static reports · Page tracking   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      FastAPI Backend (Python)                        │
│                                                                     │
│  ┌───────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Routers  │  │Middleware │  │ Services │  │  Background Jobs  │  │
│  │  (33+)    │  │ (3 layers)│  │ (15+)    │  │  (APScheduler)   │  │
│  └─────┬─────┘  └──────────┘  └────┬─────┘  └────────┬─────────┘  │
│        │                           │                  │             │
│        ▼                           ▼                  ▼             │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                    Service Layer                          │      │
│  │  Pipeline · Policy · Aggregation · Classification ·      │      │
│  │  Actions · Routing · Memory · Billing · Auth · Org       │      │
│  └──────────────────────────────┬───────────────────────────┘      │
│                                 │                                   │
│  ┌──────────────────────────────▼───────────────────────────┐      │
│  │              Platform-Specific Services                   │      │
│  │  Slack · Discord · Discourse · HackerNews · WhatsApp     │      │
│  │  Google Chat · Email · Image · Audio                     │      │
│  └──────────────────────────────────────────────────────────┘      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     PostgreSQL Database                              │
│  org_* · conv_* · pipeline_* · policy_* · agg_* · playground_*     │
└─────────────────────────────────────────────────────────────────────┘
```

### Middleware Stack

Three middleware layers process every request in order:

1. **CORSMiddleware** — Allows all origins, credentials, methods, and headers
2. **RateLimitMiddleware** — Sliding window rate limiter (10 req/60s per IP+path) on auth and
   organization creation endpoints
3. **RequestLoggingMiddleware** — Logs user_id (extracted from API key), client IP, method, and path
   for every request

### Request Flow

```
Client → Express Proxy → FastAPI → Middleware → Router → Service → Database
                                                            │
                                                            ├→ External APIs (Slack, Jira, etc.)
                                                            ├→ ML Models (HuggingFace)
                                                            └→ LLM (Azure OpenAI / Claude)
```

______________________________________________________________________

## 3. Data Model

The database uses a strict multi-tenant architecture where every table is scoped by `org_id`. Table
names follow domain prefixes for clarity.

### Table Naming Convention

| Prefix        | Domain                   | Examples                                                |
| ------------- | ------------------------ | ------------------------------------------------------- |
| `org_`        | Organization/user-scoped | `org_organization`, `org_user`, `org_member`, `org_app` |
| `conv_`       | Conversation-scoped      | `conv_history`, `conv_sentiment`, `conv_results_base`   |
| `pipeline_`   | Pipeline config/runs     | `pipeline`, `pipeline_run`, `pipeline_schedule`         |
| `policy_`     | Policy rules             | `policy_rules`                                          |
| `agg_`        | Aggregation pipeline     | `agg_metric`, `agg_action_item`, `agg_alert_event`      |
| `playground_` | Isolated playground      | `playground_history`, `playground_state`                |
| `service_`    | Internal tracking        | `service_job_status`                                    |

### 3.1 Organization & User Management

**org_organization** — Top-level tenant entity.

- `org_id` (PK), `name`, `slug` (unique), `owner_id`, `webhook_url`, `webhook_secret`, `settings`
  (JSON), timestamps

**org_user** — User accounts supporting multiple auth providers.

- `user_id` (UUID PK), `email` (unique), `name`, `google_id`, `github_id`, `auth_provider`,
  `password_hash`, `avatar_url`, `is_active`, `org_id`, timestamps

**org_member** — Organization membership with role-based access.

- `org_id`, `user_id`, `email`, `role` (OWNER/ADMIN/MEMBER), `status` (PENDING/ACTIVE/INACTIVE),
  `invited_by`, `invited_at`, `joined_at`

**org_project** — Projects within an organization for scoping applications.

- `project_id` (UUID PK), `org_id`, `name`, `owner_id`, `description`, `meta` (JSON)

**org_app** — Applications registered under projects.

- `application_id` (UUID PK), `project_id`, `org_id`, `name`, `owner_id`, `status`
  (ACTIVE/BETA/DEPRECATED/SUNSET), `meta` (JSON), engineering/product/support owner configurations
  with integration types and configs

**org_app_apikey** — API keys with role-based scopes.

- `key_id` (PK), `org_id`, `app_id`, `role` (developer/product_manager/admin), `api_key`
  (Fernet-encrypted), `scopes`, `expires_at`, `revoked`

**org_user_preference** — Key-value preferences per user per organization.

- Unique on `(org_id, user_id, category, key)`, value stored as Text

**org_user_memory** — AI-extracted or manual learned context from conversations.

- `source`, `confidence`, `status` (active/confirmed/dismissed), `tags` (JSON)

### 3.2 Conversation Analysis Tables

**conv_history** — Raw conversation data stored as JSON message arrays.

- `org_id`, `pipeline_id`, `run_id`, `conversation_id`, `conv_userid`, `conversation` (JSON array of
  messages)

**conv_summary** — Aggregated conversation metrics.

- `conversation_id` (unique), `message_count`, `turns`, `duration`, `rating`, `summary`
- Agent/model tracking: `models`, `agent_ids`, `tool_calls` (JSON arrays)
- Response quality: `avg_response_time`, `empty_response_count`, `error_count`, `broken_links_rate`

**Signal Analysis Tables** — Each stores per-conversation AI classification results:

| Table            | Key Fields                                           | Model Used                                                 |
| ---------------- | ---------------------------------------------------- | ---------------------------------------------------------- |
| `conv_sentiment` | `sentiment` (neg/neutral/pos), `confidence_score`    | cardiffnlp/twitter-roberta-base-sentiment + Haiku fallback |
| `conv_emotion`   | `emotion` (27 labels), `confidence_score`            | SamLowe/roberta-base-go_emotions + Haiku fallback          |
| `conv_topics`    | `primary_topic`, `secondary_topic`, `is_unsupported` | Azure OpenAI GPT-4o-mini (batched)                         |
| `conv_intent`    | `intent`, `confidence_score`, `extra_metadata`       | Azure OpenAI GPT-4o-mini (batched)                         |
| `conv_risk`      | `risk_category` (8 labels), `confidence_score`       | facebook/bart-large-mnli (zero-shot)                       |

**conv_state** — Real-time conversation state tracking.

- `current_sentiment`, `current_emotion`, `current_topic`, `current_intent`
- `handoff_alerted`, `handoff_scenario_triggered`, `user_type`

**conv_users (EndUser)** — Aggregated end-user profiles.

- `conv_userid` (UUID PK), `org_id`, `first_seen`, `last_seen`, demographics, `extra_metadata`

**conv_user_rawevent** — Normalized incoming SDK events.

- `event_type`, `agent_id`, `message_id`, `experiment_id`, `product_id`, device/platform info

### 3.3 Policy Layer Tables

```
conv_results_base   ← Consolidated AI outputs (materialized from signal tables)
        │
        ▼
policy_rules        ← Natural language policies (guardrails) and rules (preferences)
        │
        ▼
conv_overrides      ← Per-conversation user edits (all fields nullable)
        │
        ▼
conv_results_final  ← Merged output with field_sources provenance tracking
        │
rule_application_logs  ← Audit trail of every rule/override application
        │
rule_suggestions    ← Auto-generated rule suggestions from override patterns
```

**conv_results_base** — Consolidated AI outputs materialized from per-signal analysis tables.

- All classification fields: sentiment, emotion, topic, intent, risk (with confidence scores)
- Flags: `is_resolved`, `needs_action`, `priority`, `severity`, `issue_type`

**conv_overrides** — Per-conversation user edits. Same fields as base, all nullable.

- Unique on `(org_id, conversation_id, pipeline_id)`
- Tracks `user_id` and `notes`

**policy_rules** — Natural language policies and rules.

- `rule_type` (policy = guardrail applied first, rule = preference applied second)
- `target_field` (sentiment, emotion, topics, intent, risk, resolved, action, priority, severity,
  issue_type)
- Pipeline scope: `pipeline_id = ""` means org-wide

**conv_results_final** — Merged output after full policy layer.

- `field_sources` (JSON): maps each field to its provenance — `"base"`, `"policy:<id>"`,
  `"rule:<id>"`, or `"override"`

**rule_application_logs** — Audit trail with before/after values per field.

**rule_suggestions** — Auto-generated from override patterns. Status: `pending` → `approved` |
`rejected`.

### 3.4 Aggregation Pipeline Tables

**agg_conversation_fact** — Unified fact table (input to aggregation).

- Source-agnostic fields: `issue_type`, `sentiment`, `is_resolved`, `needs_action`, `priority`
- Slack-specific: `reply_count`, `is_unanswered`, `first_response_time`, `release_mention`
- WhatsApp-specific: `payment_issue`, `outage`, `compliance_issue`

**agg_metric** — Comprehensive metrics at multiple grains.

- Grains: `global`, `by_issue`, `by_customer`, `by_product`, `by_channel`
- Periods: `daily`, `weekly`, `monthly`
- 50+ metric fields covering volume, sentiment, resolution, severity, trends, scoring

**agg_issue_cluster** — Semantically grouped issues.

- `impact_score`, `is_systemic`, `conversation_count`, `unique_customer_count`

**agg_customer_health / agg_product_health** — Health snapshots (0–100 score).

- `negative_rate`, `unresolved_rate`, `repeat_issue_count`, `health_score`, `trend`

**agg_action_item** — Actionable items generated from detectors.

- `scope_type` (global/product/customer/channel/issue_cluster)
- `recommended_action_type` (15+ types), `owner_role`, `priority`, `due_hint`
- `evidence_metrics` (JSON), `evidence_issue_keys`, `evidence_customers`

**agg_alert_event** — Threshold-based alerts from detectors.

- `detector_name`, `severity`, `title`, `description`, `evidence` (JSON)

**agg_issue_playbook** — Learned resolution patterns.

- `common_root_cause`, `best_owner_role`, `best_first_response_template`
- `historical_resolution_rate`, `time_to_resolution_hours`

**agg_report** — Generated reports (daily/weekly/executive).

### 3.5 Platform-Specific Tables

**Slack:** `conv_slack_channel`, `conv_slack_thread_analysis`, `conv_slack_thread_cluster`,
`conv_slack_thread_cluster_member`, `conv_slack_summary`

**Discourse:** `conv_discourse_site`, `conv_discourse_topic_analysis`, `conv_discourse_summary`

**HackerNews:** `conv_hackernews_site`, `conv_hackernews_thread_analysis`, `conv_hackernews_summary`

**Google Chat:** `conv_google_chat`

**Community Discussions:** `conv_community_discs`, `conv_community_disc_analysis`

**Conversation Segments:** `conv_segment` — WhatsApp/file-based conversation segments with extracted
actors, entities, financial events, system events.

### 3.6 Playground Tables (Isolated)

The playground operates with its own isolated tables, decoupled from the pipeline:

- `playground_history` — Conversation storage
- `playground_state` — Real-time state tracking
- `playground_emotion` — Emotion tracking
- `playground_file` — Uploaded files
- `playground_scenario` — Handoff scenarios
- `playground_event` — Handoff alert events
- `playground_metrics` — Aggregated scenario metrics

### 3.7 Anonymous Access

- `anonymous_user` — Tracks anonymous visitors with fingerprint, visit count, credit balance
- `anonymous_conversation` — Anonymous conversation sessions
- `anonymous_usage_log` — Per-call usage tracking for anonymous users

### Design Patterns

- **UUIDs** for all externally-exposed IDs
- **JSON fields** for flexible data (metadata, config, embeddings, aggregates)
- **UTC timezone-aware datetimes** with `created_at`/`updated_at` on all tables
- **Confidence scores** as 0.0–1.0 floats on all ML predictions
- **Composite unique constraints** for multi-field natural keys
- **Foreign keys with indexes** for referential integrity and query performance

______________________________________________________________________

## 4. Authentication & Authorization

### 4.1 Authentication Providers

Jupiter supports three authentication methods:

**Google OAuth**

1. Frontend obtains Google ID token via `@react-oauth/google`
2. Token sent to `POST /auth/google` (login) or `POST /auth/google/signup` (new users)
3. Backend verifies token against Google's JWKS using `GOOGLE_CLIENT_ID`
4. Returns user object + organizations list

**GitHub OAuth**

1. Frontend redirects to GitHub OAuth flow
2. Callback exchanges code for access token via `POST /auth/github/callback`
3. Backend verifies token against GitHub API (`/user` + `/user/emails`)
4. Returns user object + organizations list

**Email/Password**

1. `POST /auth/email/signup` — Sends 6-digit verification code via SendGrid
2. `POST /auth/email/verify` — Validates the code (15-minute expiry)
3. `POST /auth/email/complete-signup` — Sets password (bcrypt hashed), creates account
4. `POST /auth/email/login` — Validates email + password
5. Password reset: `forgot-password` → `verify-reset-code` → `reset-password`

### 4.2 Token Management

The Express frontend server manages JWT tokens:

- **Token generation**: Custom JWT with issuer `PRIMEORBIT_ACCESS_TOKEN_ISS`
- **Token validity**: 10 hours (configurable via `ACCESS_TOKEN_VALIDITY_PERIOD_SECONDS`)
- **Max lifetime**: 30 hours
- **Proactive refresh**: Frontend schedules refresh 5 minutes before expiration
- **Secret keys**: Stored in `deployment_data/` folder
  - `accessTokenSecretKey.txt` — For access tokens
  - `loginTokenSymmetricKey.txt` — For login tokens

### 4.3 API Key Authentication

For programmatic access (SDK events, external integrations):

- **Generation**: 32-byte URL-safe token, SHA-256 hashed for storage
- **Encryption**: Fernet symmetric encryption for the stored key
- **Roles**: `developer`, `product_manager`, `admin`
- **Scopes**: Comma-separated fine-grained permissions
- **Validation**: `Authorization: Bearer <key>` header, looked up by hash
- **Expiry**: Optional `expires_at` timestamp
- **Revocation**: `revoked` flag for immediate invalidation

### 4.4 Organization Membership & Roles

**Role hierarchy**: OWNER > ADMIN > MEMBER

| Action              | Owner | Admin            | Member |
| ------------------- | ----- | ---------------- | ------ |
| Invite members      | Yes   | Yes              | No     |
| Update member roles | Yes   | Yes (not owners) | No     |
| Remove members      | Yes   | Yes (not owners) | No     |
| Create pipelines    | Yes   | Yes              | Yes    |
| View analytics      | Yes   | Yes              | Yes    |
| Manage integrations | Yes   | Yes              | No     |

**Invitation flow:**

1. Admin/Owner creates invitation → pending `org_member` record
2. Email sent via SendGrid with login link
3. On signup, `accept_pending_invites_for_user()` auto-activates matching invitations
4. Invitations can be resent or cancelled

### 4.5 Rate Limiting

In-memory sliding window rate limiter on sensitive endpoints:

- **Protected paths**: `/auth/google`, `/auth/github`, `/organizations`
- **Limit**: 10 requests per 60-second window per client IP
- **Cleanup**: Memory cleanup every 5 minutes
- **Response**: HTTP 429 Too Many Requests on violation

______________________________________________________________________

## 5. Pipeline System

The pipeline system is the core orchestration layer that manages conversation ingestion and analysis
across all supported platforms.

### 5.1 Pipeline Lifecycle

```
DRAFT → ACTIVE → PAUSED → ARCHIVED
  │        │        │
  │        ├────────┘ (can resume)
  └────────┘ (can activate)
```

**Pipeline configuration** is stored as both structured JSON (`config`) and generated YAML
(`yaml_content`). Version snapshots are taken on every update.

### 5.2 Pipeline CRUD

All operations in `src/services/pipeline/service.py`:

- `create_pipeline(org_id, user_id, name, config)` — Creates pipeline with auto-generated YAML
- `update_pipeline(pipeline_id, user_id, updates)` — Updates config with automatic version snapshot
- `archive_pipeline(pipeline_id)` — Soft delete via status change to ARCHIVED
- `get_pipeline_versions(pipeline_id)` — Full version history for audit/rollback

### 5.3 Pipeline Execution

**Entry point**: `POST /pipelines/{pipeline_id}/run`

```
start_pipeline_run()
    │
    ├─→ Validate pipeline exists and status is ACTIVE/DRAFT
    ├─→ Create PipelineRun record (status: PENDING)
    ├─→ Spawn background thread
    │
    └─→ execute_pipeline_run()
            │
            ├─→ Route by source type:
            │    ├─ Slack → _execute_pipeline_run_async()
            │    ├─ Discourse → execute_discourse_pipeline_run()
            │    ├─ HackerNews → execute_hackernews_pipeline_run()
            │    ├─ Discord → execute_discord_pipeline_run()
            │    └─ File Upload → zip_executor
            │
            ├─→ Platform-specific analysis pipeline
            ├─→ Store results in platform tables + backward-compat tables
            ├─→ Send completion notifications
            │
            └─→ trigger_aggregation() (background thread)
                    └─→ 7-stage aggregation pipeline
```

### 5.4 Pipeline Scheduling

Pipelines can be scheduled for automatic recurring execution:

- **Frequencies**: Hourly, Daily, Weekly, Twice-a-week
- **Timezone support**: Per-schedule timezone configuration
- **Start time**: Optional HH:MM start time for cron-based triggers
- **Implementation**: APScheduler with SQLAlchemy job store + ThreadPoolExecutor

```python
# Frequency → APScheduler trigger mapping
HOURLY      → IntervalTrigger(hours=1)
DAILY       → IntervalTrigger(days=1)
WEEKLY      → IntervalTrigger(weeks=1)
TWICE_A_WEEK → CronTrigger(day_of_week="mon,thu")
```

______________________________________________________________________

## 6. Conversation Analysis Engine

The analysis engine extracts signals from conversations using a combination of ML models and LLM
fallbacks. Each signal type has its own background job.

### 6.1 Sentiment Analysis

**File**: `src/jobs/sentiment_analysis.py`

**Primary model**: `cardiffnlp/twitter-roberta-base-sentiment` (HuggingFace) **Fallback**: Claude
Haiku (when ML model fails or user messages can't be extracted)

**Flow:**

1. Extract user messages from conversation (priority-based rules: `me:`, `user:`, `customer:`)
2. Run ML model → returns (sentiment label, confidence score)
3. If extraction fails → fall back to Haiku for conversation parsing
4. Upsert result into `conv_sentiment` table

**Output labels**: `POSITIVE`, `NEUTRAL`, `NEGATIVE`

### 6.2 Emotion Detection

**File**: `src/jobs/emotion_detection.py`

**Model**: `SamLowe/roberta-base-go_emotions` (HuggingFace, 27 labels)

**Flow:** Same as sentiment — ML primary, Haiku fallback.

**Output labels** (27): ADMIRATION, AMUSEMENT, ANGER, ANNOYANCE, APPROVAL, CARING, CONFUSION,
CURIOSITY, DESIRE, DISAPPOINTMENT, DISAPPROVAL, DISGUST, EMBARRASSMENT, EXCITEMENT, FEAR, GRATITUDE,
GRIEF, JOY, LOVE, NERVOUSNESS, NEUTRAL, OPTIMISM, PRIDE, REALIZATION, RELIEF, REMORSE, SADNESS,
SURPRISE

### 6.3 Topic Detection

**File**: `src/jobs/topic_detection.py`

**Model**: Azure OpenAI GPT-4o-mini (batched LLM calls)

**Flow:**

1. Fetch unprocessed conversations
2. Smart batching with token estimation (`len(text)//4 + buffer`)
3. Encourage topic reuse by including existing org topics in prompt
4. LLM returns: `primary_topic`, `secondary_topic`
5. Zero-shot classification for "bot unable to help" detection → `is_unsupported` flag
6. Deduct credits for token usage

**Smart batching**: Calculates `max_input_tokens` per conversation based on batch size and
`max_total_tokens` (default 12,400) to avoid context window overflow.

### 6.4 Intent Detection

**File**: `src/jobs/intent_detection.py`

**Model**: Azure OpenAI GPT-4o-mini (batched LLM calls)

**Flow:**

1. Concatenate user messages from conversation
2. Smart batching with token-aware grouping
3. LLM returns: `primary_intent`, with optional `secondary_intent`, `intent_category`
4. Encourage intent reuse (top 50 existing intents per org included in prompt)
5. Deduct credits

### 6.5 Risk Detection

**File**: `src/jobs/risk_detection.py`

**Model**: `facebook/bart-large-mnli` (zero-shot classification, no LLM cost)

**Risk labels**: SAFE, CREDENTIALS, PERSONAL_INFO, ADMIN_ACCESS, DISCRIMINATORY, HARASSING,
PROMPT_INJECTION, MALICIOUS, RACIST, RISKY

**Logic**: If confidence < 50%, defaults to SAFE.

### 6.6 Signal Classification (Real-Time)

**File**: `src/services/classification/signal_classifier.py`

For real-time API calls (not batch jobs), a unified classifier:

1. Uses Azure OpenAI with tool calling
2. System prompt from prompt registry (`classification_system`)
3. Returns: `signal_type`, `category_type`, `priority`, `suggested_action`, `suggested_team`
4. Checks user preferences for team_chat platform and issue_tracker before executing actions
5. Deducts credits per call

### 6.7 Analysis Flow Diagram

```
ConversationHistory
    │
    ├─→ Sentiment Job ──→ conv_sentiment
    ├─→ Emotion Job ────→ conv_emotion
    ├─→ Topic Job ──────→ conv_topics
    ├─→ Intent Job ─────→ conv_intent
    ├─→ Risk Job ───────→ conv_risk
    │
    └─→ Policy Engine (materialize_base_result)
            │
            └─→ conv_results_base (consolidated)
                    │
                    └─→ produce_final_result() (apply policies/rules/overrides)
                            │
                            └─→ conv_results_final (with field_sources provenance)
```

______________________________________________________________________

## 7. Policy Layer (Signal Tuning)

The policy layer sits between AI analysis and the aggregation pipeline. It allows users to correct
AI outputs and define rules that adjust future classifications.

### 7.1 Architecture

```
conversation → deterministic extraction → LLM extraction → base_result
base_result + facts → apply global policies → apply reusable rules
  → apply conversation overrides → sanity checks → final_result
```

**Override priority** (highest to lowest):

1. Conversation overrides (user edits per conversation per pipeline)
2. Reusable rules (user-defined preferences)
3. Global policies (organizational guardrails)
4. Base AI result

### 7.2 Materialization Pipeline

**Step 1: `materialize_base_result()`**

Consolidates per-signal analysis tables into a unified `conv_results_base` record:

- `ConversationSentiment` → `base.sentiment`, `base.sentiment_confidence`
- `ConversationEmotion` → `base.emotion`, `base.emotion_confidence`
- `ConversationTopics` → `base.primary_topic`, `base.secondary_topic`
- `ConversationIntent` → `base.intent`, `base.intent_confidence`
- `ConversationRisk` → `base.risk_category`, `base.risk_confidence`

Idempotent: upserts if record exists. Batch variant processes all conversations for an org.

**Step 2: `produce_final_result()`**

Applies the full policy layer in order:

1. Initialize all fields from base values (`field_sources` = `"base"`)
2. Load and apply global policies (`rule_type="policy"`) — applied first as guardrails
3. Load and apply reusable rules (`rule_type="rule"`) — applied second as preferences
4. Apply conversation overrides — highest priority, `field_sources` = `"override"`
5. Run sanity checks — normalize values, default invalid to safe values
6. Persist `conv_results_final` with `field_sources` JSON

**Sanity checks:**

- Sentiment: must be {positive, neutral, negative}
- Priority: must be {low, medium, high}
- Flags (is_resolved, needs_action): must be {yes, no}
- Invalid values default to safe defaults

### 7.3 Rule Suggestions

The suggestion pipeline (`src/jobs/suggestion_pipeline.py`) auto-generates rule suggestions from
three sources:

1. **Owner confirmations** — When users confirm topic owners, suggest routing rules
2. **Discussion patterns** — Track responder + topic + count from resolved discussions; suggest
   rules when count ≥ threshold
3. **Override patterns** — Detect repeated field overrides to the same value in the last 30 days;
   suggest organizational rules

Suggestions are created with `status="pending"` and can be approved (creates a PolicyRule) or
rejected by the user.

### 7.4 Audit Trail

Every rule/override application is logged to `rule_application_logs` with:

- `source_type` (policy, rule, override)
- `rule_id` (for policy/rule applications)
- `field_name`, `value_before`, `value_after`
- `user_id`, `pipeline_id`, `applied_at`

### 7.5 Re-Materialization

Any layer can be re-run independently:

- `POST /policy/{org_id}/materialize?from_layer=base` — Re-consolidate from signal tables
- `POST /policy/{org_id}/materialize?from_layer=final` — Re-apply policies/rules/overrides

______________________________________________________________________

## 8. Aggregation Pipeline

The aggregation pipeline converts individual conversation analysis results into organizational
insights, metrics, alerts, and action items.

### 8.1 Seven-Stage Pipeline

```
Stage A: Fact Building
    │   Build ConversationFact from SlackThreadAnalysis / ConversationSegment
    ▼
Stage B: Quality Validation
    │   Quarantine records with missing critical fields
    ▼
Stage C: Enrichment
    │   Normalize, deduplicate, add derived fields
    ▼
Stage D: Metrics & Clustering
    │   Build AggregateMetric at multiple grains
    │   Build IssueCluster, CustomerHealthSnapshot, ProductHealthSnapshot
    ▼
Stage E: Detectors
    │   Run 8 threshold-based alert detectors
    ▼
Stage F: Action Generation
    │   Convert detector results → AggregateActionItem
    ▼
Stage G: Playbook Learning
        Update IssuePlaybook from aggregate evidence
```

**Idempotency**: Re-running the same `source_run_id` produces identical results. Facts are marked
`is_aggregated=True` after processing.

### 8.2 Fact Builder

Converts platform-specific analysis into a unified fact schema:

- **Slack** → Reads `SlackThreadAnalysis` → normalizes issue keys, maps severity to priority,
  extracts Slack-specific flags (unanswered, docs_gap, bug_report, feature_request)
- **WhatsApp** → Reads `ConversationSegment` → extracts WhatsApp-specific flags (payment_issue,
  outage, compliance, training_request)

Key normalizations:

- `_normalize_issue_key()` — Lowercase, strip, collapse spaces (for clustering)
- Priority mapping: `critical/blocker → high`, `major → medium`, `minor/trivial → low`

### 8.3 Metrics

Metrics are computed at **multiple grains**:

| Grain       | Description              | Key                  |
| ----------- | ------------------------ | -------------------- |
| Global      | Org-wide aggregates      | —                    |
| By Customer | Per-customer health      | customer_id          |
| By Channel  | Per-channel performance  | channel_id           |
| By Issue    | Per-issue-type tracking  | normalized_issue_key |
| By Product  | Per-product-area metrics | product_area         |

**Core metric categories** (50+ fields):

- **Volume**: conversation_count, unique_customers, unique_channels, total_messages
- **Sentiment**: negative_count, positive_count, smoothed_negative_rate
- **Resolution**: resolved_count, unresolved_count, resolution_rate
- **Action**: needs_action_count
- **Severity**: high_priority_count, blocker_count
- **Trends**: 1-day, 7-day, 28-day rolling counts for momentum detection
- **Issue tracking**: breadth (# customers per issue)
- **Platform-specific**: unanswered_count, response_time_median, docs_gap_count (Slack);
  payment_issue_count, outage_count (WhatsApp)

**Advanced calculations:**

- **Beta-binomial smoothing**: Priors (2 successes, 10 total) for confidence-aware rate calculation
- **Impact score**:
  `log(1 + count) × (0.4×neg_rate + 0.3×unresolved_rate + 0.2×breadth + 0.1×momentum)`

### 8.4 Detectors

Eight threshold-based alert detectors run against computed metrics:

| #   | Detector                         | Trigger Conditions                                | Severity |
| --- | -------------------------------- | ------------------------------------------------- | -------- |
| 1   | Negative Sentiment Spike         | n≥30, smoothed_neg_rate≥20%, delta>baseline+8%    | HIGH     |
| 2   | Unresolved Issue Cluster         | issue_count≥10, unresolved_rate≥40%, rising trend | HIGH     |
| 3   | Repeating Issue Across Customers | ≥3 customers, ≥12 total, ≥5 unresolved            | MEDIUM   |
| 4   | Docs Gap Candidate               | High feature_requests, low docs_gaps              | MEDIUM   |
| 5   | Bug Cluster Candidate            | High bug_reports, low fixed_count                 | HIGH     |
| 6   | Customer Health Deterioration    | Declining sentiment/resolution per customer       | HIGH     |
| 7   | Release Regression               | Post-release sentiment drop                       | CRITICAL |
| 8   | Support Load Imbalance           | Uneven distribution across channels               | MEDIUM   |

### 8.5 Action Generation

Each detector maps to a specific action type:

| Detector             | Action                        | Owner Role       | Due       |
| -------------------- | ----------------------------- | ---------------- | --------- |
| Sentiment Spike      | investigate_systemic_issue    | support_lead     | this week |
| Unresolved Cluster   | run_root_cause_analysis       | engineering      | this week |
| Repeating Issue      | prioritize_bug_cluster        | engineering      | this week |
| Docs Gap             | create_docs_update            | docs             | next week |
| Bug Cluster          | prioritize_bug_cluster        | engineering      | this week |
| Health Deterioration | schedule_customer_outreach    | customer_success | today     |
| Release Regression   | improve_release_communication | product          | today     |
| Load Imbalance       | rebalance_support_staffing    | support_lead     | this week |

### 8.6 Health Snapshots

**Customer Health** (0–100 score):

- Inputs: `negative_rate`, `unresolved_rate`, `needs_action_rate`, `repeat_issue_count`,
  `high_priority_count`
- Trend: `improving`, `stable`, `declining`
- Top issues tracked per customer

**Product Health** (0–100 score):

- Same base metrics plus: `bug_count`, `feature_request_count`, `docs_gap_count`

### 8.7 Playbook Learning

The aggregation pipeline builds and updates `IssuePlaybook` records:

- `normalized_issue_key` + `issue_family`
- `common_root_cause`, `best_owner_role`
- `best_first_response_template`, `best_resolution_steps`
- `historical_resolution_rate`, `time_to_resolution_hours`, `conversation_volume_30d`

______________________________________________________________________

## 9. Smart Actions & Workflow Automation

The smart actions system detects actionable items from conversations and executes multi-step
workflows across integrated platforms.

### 9.1 Architecture

```
Conversation Text
    │
    ▼
Action Detection (LLM)
    │   Extracts: verb, category, team, priority, confidence
    ▼
Workflow Creation
    │   Maps detected actions → ActionTemplate + ActionExecution
    │   Determines team assignment and priority
    │   Collects required inputs
    ▼
Workflow Execution
    │   Runs actions in order with dependency checking
    │   Routes to platform-specific executors
    ▼
Integration Executors
    ├─ Jira, GitHub, Linear (issue creation)
    ├─ Slack, Teams (messages)
    ├─ SendGrid (email)
    ├─ PagerDuty, Opsgenie (escalation)
    ├─ Salesforce, HubSpot (CRM)
    ├─ Confluence, Notion, Zendesk (documentation)
    └─ Custom webhooks
```

### 9.2 Action Detection

`detect_actions_from_conversation()` uses LLM to extract actions:

```json
[
  {
    "action_text": "Create a bug report for the login timeout issue",
    "verb": "create_issue",
    "category": "TICKETS",
    "team": "ENGINEERING",
    "priority": "high",
    "confidence": 0.92,
    "reason": "User reports repeated login failures",
    "context_snippet": "I've been getting timeouts on login for 3 days..."
  }
]
```

### 9.3 Action Library

Predefined action templates organized by category:

| Category   | Actions                                                        |
| ---------- | -------------------------------------------------------------- |
| Messages   | send_channel_message, send_direct_message                      |
| Tasks      | create_issue, create_task, create_ticket, schedule_meeting     |
| Escalation | escalate_ticket, escalate_team, page_pagerduty, alert_opsgenie |
| Follow-up  | schedule_reminder, create_followup, set_sla, queue_review      |
| Alerts     | alert_engineering, alert_sales_team, alert_support_team        |
| Sales      | schedule_demo, schedule_support_call                           |
| Data       | log_event, export_report, send_webhook                         |

Each template specifies: `required_inputs`, `optional_inputs`, `message_template`, integration type.

### 9.4 Workflow Execution

Workflows are multi-action sequences with:

- **Dependency ordering** — Actions can depend on previous steps
- **Input collection** — Required inputs are collected before execution
- **Status tracking** — PENDING → IN_PROGRESS → AWAITING_INPUT → COMPLETED | FAILED
- **Per-action status** — Each action tracks its own status and retry count
- **Team routing** — Determined from action mix (engineering, product, support, etc.)
- **Priority** — Derived from highest-priority action in the workflow

### 9.5 Integration Executors

Each action verb maps to a platform-specific executor:

| Executor                          | Platform                | Authentication        |
| --------------------------------- | ----------------------- | --------------------- |
| `_execute_create_jira`            | Jira REST API           | Email + API token     |
| `_execute_create_github_issue`    | GitHub API v3           | Personal access token |
| `_execute_create_linear`          | Linear API              | API key               |
| `_execute_send_slack`             | Slack Web API           | Bot token             |
| `_execute_send_email`             | SendGrid                | API key               |
| `_execute_page_pagerduty`         | PagerDuty Events API v2 | Integration key       |
| `_execute_alert_opsgenie`         | Opsgenie Alert API      | API key               |
| `_execute_create_salesforce_lead` | Salesforce REST         | OAuth token           |
| `_execute_create_hubspot_contact` | HubSpot API             | API key               |
| `_execute_create_confluence`      | Confluence REST         | Email + API token     |
| `_execute_send_webhook`           | Custom HTTP             | Bearer token          |

______________________________________________________________________

## 10. Platform-Specific Pipelines

All platform pipelines follow a consistent architecture:

```
Platform API → Raw Data → Rule Engine (deterministic) → LLM (ambiguous cases)
    → Post-Processing → Persistence → Clustering → Summarization
```

### 10.1 Slack Pipeline

**Source**: `src/services/slack/`

**Execution flow:**

1. Initialize Slack client (cookie-based or bot token authentication)
2. Read channel threads with pagination and time filtering
3. Classify channel type (13 types via rules → LLM fallback)
4. Convert raw threads to `SlackThread` objects, filter noise
5. Route to **Customer Mode** or **Product Mode** based on pipeline config

**Customer Mode** (`customer_pipeline.py`):

- Extracts: customer problems, severity, risk signals, expansion/adoption signals
- Detects: churn risk, upsell opportunities, customer frustration

**Product Mode** (`product_pipeline.py`):

- Extracts: product area, issue type, technical entities, workarounds
- Detects: feature requests, bugs, documentation gaps

**Rule Engine** (`rule_engine.py`):

- Keyword matching for support issues, feature requests, bugs, questions
- Negative language detection for frustration signals
- Resolution detection (shared with WhatsApp)
- Fast classification with zero LLM cost

**Post-pipeline:**

- Embedding-based clustering of similar threads
- Executive summary generation (per mode)
- Results stored in `conv_slack_thread_analysis`, `conv_slack_thread_cluster`, `conv_slack_summary`

### 10.2 Discourse Pipeline

**Source**: `src/services/discourse/`

**Execution flow:**

1. Fetch topics from Discourse API (supports auth or public forums)
2. Persist site metadata in `conv_discourse_site`
3. Category routing determines LLM prompt template
4. Rule engine classification (support, bugs, features, docs gaps, billing, incidents)
5. LLM extraction for ambiguous topics
6. Executive summary generation

**Category routes** (8): support_help, bug_reports, feature_requests, docs, announcements, billing,
integrations, general

**Discourse-specific fields:**

- `staff_replied`, `accepted_answer_present`, `community_answered`
- `needs_staff_response`, `answer_quality`, `knowledge_base_candidate`
- `first_response_time_seconds`, `time_to_accepted_solution_seconds`

### 10.3 HackerNews Pipeline

**Source**: `src/services/hackernews/`

**Execution flow:**

1. Search Algolia API (`hn.algolia.io`) for company mentions
2. Persist company metadata in `conv_hackernews_site`
3. **Direct LLM extraction** (no rule engine — HN content is too varied for keyword matching)
4. Extract: community sentiment, mention type, competitor mentions
5. Executive summary generation

**HN-specific fields:**

- `community_sentiment` (distinct from individual sentiment)
- `mention_type` (direct, indirect, comparison)
- `competitor_mentions`

### 10.4 Discord Pipeline

**Source**: `src/services/discord/`

**Key design decision**: Discord pipeline converts Discord messages to Slack format and reuses the
entire Slack analysis pipeline. This avoids duplicating analysis logic.

**Execution flow:**

1. Authenticate with Discord bot/user token
2. Read channels in three passes:
   - Pass 1: Messages with inline threads
   - Pass 2: Active + archived threads (forum posts, standalone threads)
   - Pass 3: Standalone messages without threads
3. Convert Discord messages → Slack-compatible format (preserving reactions, attachments, embeds)
4. Run full Slack pipeline (customer or product mode)
5. Results stored in `conv_slack_*` tables (shared with Slack)

### 10.5 WhatsApp / File Upload Pipeline

**Source**: `src/services/pipeline/zip_executor.py`

Handles uploaded conversation files (WhatsApp exports, plain text, etc.):

1. Extract ZIP contents
2. Detect source format (whatsapp, slack, plain_text)
3. Split into conversation segments
4. Extract: actors, entities, financial events, system events, product features
5. Results stored in `conv_segment`

### 10.6 Google Chat

**Source**: `src/services/google_chat/`

Processes Google Chat conversations:

- Fetches from Google Chat API (space_id, thread_id)
- Runs sentiment, emotion, topic, intent, risk analysis
- Signal classification with action execution
- Results stored in `conv_google_chat`

______________________________________________________________________

## 11. AI Agents & Question Routing

### 11.1 Question Router

**Source**: `src/services/ai_agents/`

The Question Router agent finds the best person or team to answer a question:

**Routing flow:**

1. Receive question with context (topic, intent, conversation)
2. Search for topic owner via 5-tier priority:
   - **Tier 1**: Manual routing rules (PolicyRule with `target_field="routing"`)
   - **Tier 2**: Auto rules (TopicOwner from historical data, 10+ resolutions)
   - **Tier 3**: Historical high confidence (resolved this topic 10+ times → auto-assign)
   - **Tier 4**: Historical medium confidence (3–9 times → send for confirmation)
   - **Tier 5**: Not found (no owner data)
3. Return routing recommendation with confidence and evidence

**API endpoints:**

- `POST /ai-agents/question-router/route` — Route question → `QuestionRoutingResponse`
- `POST /ai-agents/question-router/route/stream` — Route with SSE streaming
- `POST /ai-agents/question-router/suggest-answers` — Suggest answers
- `POST /ai-agents/question-router/notify` — Notify routed person/team
- `POST /ai-agents/question-router/confirm-owner` — Manually confirm owner assignment

### 11.2 Topic Ownership

**Tables**: `conv_topic_owner`, `conv_owner_confirmation`

Topic owners are tracked with:

- `owner_type` (individual, team)
- `source` (manual, suggestion, auto_confirm)
- `confidence` (high, medium, low)
- `assignment_count` — How many times this owner has been assigned

**Learning loop:**

1. Detect responders in resolved conversations
2. Track resolution patterns per topic per responder
3. Auto-confirm high-confidence assignments (10+ resolutions)
4. Create suggestions for medium-confidence (3–9 resolutions)
5. Users can manually confirm or override

### 11.3 Responder Detection

**File**: `src/jobs/responder_detection.py`

Detects who resolved conversations by scanning for resolution patterns:

- "thanks", "thank you", "fixed", "worked", "helped", "solved"
- "problem solved", "issue resolved", "perfect", "awesome", "great work"
- "appreciate it/your"

Updates discussion metadata with responder information and resolution status.

______________________________________________________________________

## 12. Integrations Hub

### 12.1 Integration Registry

**Source**: `src/integrations/registry.py`

Central registry mapping 30+ integration types to implementation classes:

| Category           | Integrations                               |
| ------------------ | ------------------------------------------ |
| Team Chat          | Slack, Microsoft Teams, Google Chat        |
| Dev Issues         | Jira, GitHub, GitLab, Linear, Azure DevOps |
| Project Management | Asana, Monday, Notion                      |
| Support            | Zendesk, Freshdesk, Intercom, ServiceNow   |
| CRM                | Salesforce, HubSpot                        |
| Escalation         | PagerDuty, Opsgenie                        |
| Email              | Gmail, Microsoft Outlook, SendGrid         |
| Calendar           | Google Calendar, Microsoft Outlook         |
| Product Feedback   | Productboard                               |
| Knowledge Base     | Confluence, Notion                         |
| Webhooks           | Custom Webhook                             |

### 12.2 Integration Architecture

Each integration extends `BaseIntegration` with:

- `name`, `description`, `category`, `icon`
- `auth_type` (OAuth, API Key, Custom)
- `config_fields` — JSON schema for configuration
- `supported_actions` — List of actions this integration supports

### 12.3 Issue Connectors

**Source**: `src/services/destinations/issue_connectors.py`

Abstract `IssueConnector` interface with implementations:

| Connector | Auth                  | Actions                          |
| --------- | --------------------- | -------------------------------- |
| GitHub    | Personal access token | Create issue in owner/repo       |
| Jira      | Email + API token     | Create issue in project_key      |
| Linear    | API key               | Create issue (framework present) |

**Factory**: `get_issue_connector()` instantiates the appropriate connector based on org integration
configuration.

### 12.4 Webhook Subscribers

Real-time event forwarding to external systems:

- **Types**: Slack, Teams, Jira, Generic webhook
- **Status**: Active, Stopped
- **Configuration**: URL, client_secret, webhook_type
- **Lifecycle**: Create → Enable → Stop → Delete

______________________________________________________________________

## 13. Background Jobs & Scheduling

### 13.1 Job Executor

**Entry point**: `src/job_executor.py`

```bash
python src/job_executor.py        # Start scheduler with all registered jobs
python src/job_executor.py -f     # Run all jobs once manually
```

### 13.2 Job Registry

**Source**: `src/services/scheduler/runner.py`

| Job ID                  | Function                              | Frequency | Description                      |
| ----------------------- | ------------------------------------- | --------- | -------------------------------- |
| sync_end_users          | `sync_end_users()`                    | 2 min     | Aggregate UserRawEvent → EndUser |
| sync_agents             | `sync_agents()`                       | 2 min     | Auto-discover agents from events |
| process_conversations   | `process_conversations()`             | 5 min     | Raw events → ConversationSummary |
| conversation_sentiments | `calculate_conversation_sentiments()` | 5 min     | ML sentiment analysis            |
| conversation_emotions   | `calculate_conversation_emotion()`    | 5 min     | ML emotion detection             |
| conversation_topics     | `detect_conversation_topics()`        | 5 min     | LLM topic classification         |
| conversation_intents    | `detect_conversation_intents()`       | 5 min     | LLM intent extraction            |
| conversation_risks      | `calculate_conversation_risks()`      | 5 min     | Zero-shot risk detection         |
| extract_preferences     | `extract_preferences()`               | 5 min     | LLM memory extraction            |
| suggestion_pipeline     | `run_suggestion_pipeline()`           | 30 min    | Auto-generate rule suggestions   |

### 13.3 Conversation Processing Pipeline

Jobs run in dependency order within `run_conversation_pipeline()`:

```
process_conversations (raw events → history + summary)
    │
    ├─→ calculate_conversation_sentiments
    ├─→ detect_conversation_topics
    ├─→ detect_conversation_intents
    ├─→ detect_conversation_emotions
    ├─→ calculate_conversation_risks
    │
    └─→ extract_preferences
```

### 13.4 Job Status Tracking

All jobs are wrapped with `JobStatus` tracking:

- Records `started_at`, `finished_at`, `status`, `error_message`
- `get_job_last_processed_time()` returns last completion time (or 90 days ago if never run)
- Used for incremental processing — only process conversations since last run

### 13.5 Pipeline Schedules

Alongside fixed-interval jobs, pipeline schedules are hot-loaded from the database:

- `register_pipeline_schedules()` queries enabled `PipelineSchedule` records
- Each schedule is registered as an APScheduler job with the appropriate trigger
- Supports timezone-aware scheduling

______________________________________________________________________

## 14. Frontend Architecture

### 14.1 Technology Stack

| Technology     | Purpose                                                |
| -------------- | ------------------------------------------------------ |
| React 19       | UI framework with functional components and hooks      |
| TypeScript     | Type safety with strict mode                           |
| Vite           | Build tool and dev server                              |
| Tailwind CSS 4 | Utility-first styling with dark mode                   |
| Shadcn/ui      | Component library (Radix UI + Tailwind)                |
| Express        | SSR server for proxy, token management, static reports |
| Recharts       | Chart and visualization library                        |

### 14.2 Application Structure

```
frontend/src/
├── pages/           30 page components
├── components/      Domain-organized reusable components
│   ├── ui/          25+ pure UI building blocks
│   ├── dashboard/   Sidebar, global filters
│   ├── settings/    Account, org, members, projects, integrations, API keys, usage
│   ├── smart_actions/  Overview, playground, analytics, workflows, integrations
│   ├── policy/      Signal tuning (rules, overrides, suggestions)
│   ├── agents/      Agent overview, analytics, configuration
│   ├── apps/        Application management
│   ├── playground/  Full conversation analysis demo
│   └── playground2/ Lightweight demo variant
├── hooks/           4 custom hooks
├── context/         2 React context providers
├── lib/             API client (4,100+ lines), utilities
├── routes/          Route configuration
├── types/           15 TypeScript type definition files
└── server/          Express server (proxy, tokens, reports)
```

### 14.3 Key Pages

| Page                   | Route                    | Description                                                 |
| ---------------------- | ------------------------ | ----------------------------------------------------------- |
| Dashboard              | `/dashboard`             | Main analytics with trends, breakdowns, metrics             |
| Conversations          | `/conversation`          | Conversation analytics with filtering                       |
| Users                  | `/users`                 | User registry with conversation history                     |
| Agents                 | `/agents`                | Agent registry with performance analytics                   |
| Applications           | `/apps`                  | Application management with analytics                       |
| Actions                | `/actions`               | Smart actions + Signal Tuning tab                           |
| Pipelines              | `/pipelines`             | Pipeline management with runs and versions                  |
| Settings               | `/settings`              | Tabbed settings (account, org, members, integrations, etc.) |
| Real-time Intelligence | `/realtime-intelligence` | Webhook subscriber management                               |
| Playground             | `/playground`            | Conversation analysis demo                                  |
| Google Chat            | `/google-chat`           | Google Chat integration                                     |

### 14.4 Route Guards

Four route guard components control access:

| Guard                  | Purpose                                              |
| ---------------------- | ---------------------------------------------------- |
| `PrivateRoute`         | Requires authenticated user                          |
| `PublicRoute`          | Prevents logged-in users from accessing login/signup |
| `PlaygroundRoute`      | Allows both authenticated and anonymous access       |
| `RequiresOrganization` | Allows access with any org (placeholder or complete) |

### 14.5 State Management

**Auth Context** (`auth_context.tsx`):

- User authentication state (user object with token)
- Organizations list and pending invitations
- Playground mode with time tracking
- Methods: login, signup, logout, refreshOrganizations, enterPlaygroundMode

**Organization Context** (`organization_context.tsx`):

- Current organization (org_id, name, slug, settings)
- Auto-loads from localStorage
- Methods: setOrganization, refreshOrganization

**URL State** (`use_url_filters` hook):

- Date range, organization, pipeline, granularity persisted in URL query params
- Enables browser back/forward navigation

### 14.6 API Client

**Source**: `frontend/src/lib/api.ts` (4,100+ lines, 100+ functions)

Organized by domain:

| Domain                  | Function Count | Examples                                                            |
| ----------------------- | -------------- | ------------------------------------------------------------------- |
| Authentication          | 15+            | `authenticateWithGoogle`, `loginWithEmail`, `changePassword`        |
| Organizations & Members | 15+            | `createOrganization`, `inviteMember`, `updateMemberRole`            |
| Projects & Applications | 15+            | `createProject`, `createApplication`, `getAppsAnalytics`            |
| Agents                  | 10+            | `getAgents`, `getAgentsOverview`, `getAgentConversations`           |
| Analysis                | 15+            | `analyzeSentiment`, `detectTopics`, `detectIntent`, `detectRisk`    |
| Smart Actions           | 15+            | `detectSmartActions`, `createSmartWorkflow`, `executeSmartWorkflow` |
| Pipelines               | 10+            | `createPipeline`, `startPipelineRun`, `listPipelineRuns`            |
| Policy                  | 10+            | `listPolicyRules`, `upsertConvOverride`, `materializePolicyLayer`   |
| Integrations            | 20+            | `createIntegration`, `testIntegration`, `executeAction`             |
| Analytics               | 15+            | `getUnifiedAnalytics`, `getSentimentBreakdown`, `getTopicBreakdown` |
| Memory                  | 10+            | `listMemories`, `createMemory`, `searchMemories`                    |
| Payments                | 5+             | `createCheckoutSession`, `getPaymentStatus`                         |

All functions use a shared `apiCall()` base with error handling and custom `ApiError` class.

### 14.7 Express Server

**Source**: `frontend/src/server/`

The Express server provides:

- **Proxy middleware**: Routes `/backend/*` to the Python FastAPI backend
- **Token refresh**: `POST /frontend/get_access_token` for JWT token refresh
- **Static reports**: Serves pre-built HTML report pages
- **Page tracking**: Silent fire-and-forget page view tracking
- **Compression**: gzip compression for all responses

______________________________________________________________________

## 15. API Reference

The backend exposes 200+ endpoints across 33 router files. Below is a summary organized by domain.

### 15.1 Authentication

| Method | Path                             | Description                            |
| ------ | -------------------------------- | -------------------------------------- |
| POST   | `/auth/google`                   | Google OAuth login                     |
| POST   | `/auth/google/signup`            | Google OAuth signup                    |
| POST   | `/auth/github`                   | GitHub OAuth login                     |
| POST   | `/auth/github/signup`            | GitHub OAuth signup                    |
| POST   | `/auth/github/callback`          | Exchange GitHub code for token         |
| POST   | `/auth/email/signup`             | Start email signup (send verification) |
| POST   | `/auth/email/verify`             | Verify email code                      |
| POST   | `/auth/email/complete-signup`    | Complete signup with password          |
| POST   | `/auth/email/login`              | Email/password login                   |
| POST   | `/auth/email/forgot-password`    | Start password reset                   |
| POST   | `/auth/email/reset-password`     | Reset password                         |
| POST   | `/auth/email/change-password`    | Change password                        |
| GET    | `/users/{user_id}/organizations` | Get user's organizations               |
| PATCH  | `/users/{user_id}`               | Update user profile                    |

### 15.2 Organizations & Members

| Method | Path                                             | Description         |
| ------ | ------------------------------------------------ | ------------------- |
| POST   | `/organizations`                                 | Create organization |
| GET    | `/organizations/{org_id}`                        | Get organization    |
| PUT    | `/organizations/{org_id}`                        | Update organization |
| POST   | `/organizations/{org_id}/logo`                   | Upload logo         |
| POST   | `/organizations/{org_id}/members/invite`         | Invite member       |
| GET    | `/organizations/{org_id}/members`                | List members        |
| PUT    | `/organizations/{org_id}/members/{user_id}/role` | Update role         |
| DELETE | `/organizations/{org_id}/members/{user_id}`      | Remove member       |

### 15.3 Analysis

| Method | Path                 | Description                  |
| ------ | -------------------- | ---------------------------- |
| POST   | `/analyze-sentiment` | Analyze sentiment            |
| POST   | `/detect-topics`     | Detect topics                |
| POST   | `/detect-emotion`    | Detect emotion               |
| POST   | `/detect-intent`     | Detect intent                |
| POST   | `/detect-risk`       | Detect risk                  |
| POST   | `/summarize`         | Summarize conversation       |
| POST   | `/extract-text`      | Extract text from images     |
| POST   | `/analyze-audio`     | Transcribe and analyze audio |

### 15.4 Policy Layer

| Method | Path                                                             | Description            |
| ------ | ---------------------------------------------------------------- | ---------------------- |
| POST   | `/policy/{org_id}/pipelines/{pid}/conversations/{cid}/overrides` | Create/update override |
| GET    | `/policy/{org_id}/pipelines/{pid}/conversations/{cid}/overrides` | Get override           |
| DELETE | `/policy/{org_id}/pipelines/{pid}/conversations/{cid}/overrides` | Delete override        |
| POST   | `/policy/{org_id}/rules`                                         | Create rule            |
| GET    | `/policy/{org_id}/rules`                                         | List rules             |
| PATCH  | `/policy/{org_id}/rules/{rule_id}`                               | Update rule            |
| DELETE | `/policy/{org_id}/rules/{rule_id}`                               | Delete rule            |
| GET    | `/policy/{org_id}/pipelines/{pid}/conversations/{cid}/results`   | Get results            |
| GET    | `/policy/{org_id}/pipelines/{pid}/conversations/{cid}/logs`      | Get audit logs         |
| POST   | `/policy/{org_id}/materialize`                                   | Re-run policy layer    |
| GET    | `/policy/{org_id}/suggestions`                                   | List suggestions       |
| POST   | `/policy/{org_id}/suggestions/{id}/approve`                      | Approve suggestion     |
| POST   | `/policy/{org_id}/suggestions/{id}/reject`                       | Reject suggestion      |

### 15.5 Pipelines

| Method | Path                                | Description      |
| ------ | ----------------------------------- | ---------------- |
| POST   | `/pipelines`                        | Create pipeline  |
| GET    | `/pipelines`                        | List pipelines   |
| GET    | `/pipelines/{pipeline_id}`          | Get pipeline     |
| PATCH  | `/pipelines/{pipeline_id}`          | Update pipeline  |
| DELETE | `/pipelines/{pipeline_id}`          | Archive pipeline |
| POST   | `/pipelines/{pipeline_id}/run`      | Start run        |
| GET    | `/pipelines/{pipeline_id}/runs`     | List runs        |
| POST   | `/pipelines/{pipeline_id}/schedule` | Create schedule  |
| GET    | `/pipelines/{pipeline_id}/schedule` | Get schedule     |

### 15.6 Smart Actions & Agents

| Method | Path                                         | Description          |
| ------ | -------------------------------------------- | -------------------- |
| POST   | `/ai-agents/question-router/route`           | Route question       |
| POST   | `/ai-agents/question-router/route/stream`    | Route with streaming |
| POST   | `/ai-agents/question-router/suggest-answers` | Suggest answers      |
| POST   | `/ai-agents/question-router/confirm-owner`   | Confirm owner        |

### 15.7 Integrations & Webhooks

| Method | Path                                  | Description         |
| ------ | ------------------------------------- | ------------------- |
| GET    | `/integrations/hub`                   | Integration catalog |
| POST   | `/integrations/{org_id}`              | Create integration  |
| GET    | `/integrations/{org_id}`              | List integrations   |
| POST   | `/integrations/{org_id}/{id}/connect` | Connect integration |
| POST   | `/webhooks/subscribers`               | Create subscriber   |
| GET    | `/webhooks/subscribers`               | List subscribers    |

### 15.8 Analytics

| Method | Path                                     | Description       |
| ------ | ---------------------------------------- | ----------------- |
| POST   | `/conversations/analytics/unified`       | Unified analytics |
| GET    | `/organizations/{org_id}/usage/overview` | Usage overview    |
| GET    | `/users/registry`                        | List end users    |

### 15.9 Payments

| Method | Path                        | Description            |
| ------ | --------------------------- | ---------------------- |
| POST   | `/payments/create-checkout` | Create Stripe checkout |
| GET    | `/payments/status/{org_id}` | Payment status         |
| POST   | `/payments/verify-session`  | Verify session         |
| POST   | `/payments/webhook`         | Stripe webhook         |

______________________________________________________________________

## 16. LLM Prompt System

### 16.1 Architecture

All LLM prompts are managed through a centralized, versioned registry. **Prompts are never defined
inline in service files or jobs.**

```
src/prompts/
├── __init__.py         # Imports registry, calls register_all()
├── registry.py         # PromptDefinition model + PromptRegistry class
└── v1/                 # Version 1 prompt definitions
    ├── conversation_analysis.py
    ├── summarization.py
    ├── content_detection.py
    ├── image_extraction.py
    ├── dashboard_insights.py
    ├── memory_extraction.py
    ├── topic_detection.py
    ├── intent_detection.py
    ├── slack_analysis.py
    ├── discourse_analysis.py
    ├── hackernews_analysis.py
    ├── source_profiling.py
    ├── segment_extraction.py
    ├── conversation_splitting.py
    └── chunk_conversation_splitting.py
```

### 16.2 PromptDefinition Model

```python
class PromptDefinition(BaseModel):
    name: str                          # Unique identifier
    version: str = "1.0.0"            # Semver versioning
    category: str                      # Domain grouping
    description: str                   # Human-readable description
    system_prompt: str | None          # System message for LLM
    user_prompt_template: str | None   # User message template with {placeholders}
    template_vars: list[str] = []      # Expected template variables
    changelog: list[str] = []          # Version history
```

### 16.3 Usage Pattern

```python
from prompts import registry

# Get a prompt definition
prompt = registry.get("classification_system")

# Render with variables
rendered = registry.get_user_prompt("sentiment_analysis", raw_text=text)

# Get just the system prompt
system = registry.get_system_prompt("classification_system")
```

### 16.4 Key Prompt Categories

| Category              | Prompts                                                                        | Purpose                                     |
| --------------------- | ------------------------------------------------------------------------------ | ------------------------------------------- |
| Conversation Analysis | classification_system, action_system, detect_actions, extract_signals          | Signal classification and action detection  |
| Summarization         | summarize_conversation, summarize_email, summarize_document                    | Multi-format summarization                  |
| Content Detection     | detect_conversation_image, detect_conversation_email, detect_conversation_text | Determine if content is a conversation      |
| Image/Vision          | extract_text_from_image                                                        | OCR text extraction                         |
| Dashboard             | dashboard_insights                                                             | Analytics data analysis                     |
| Memory                | memory_extraction                                                              | Extract user preferences from conversations |
| Platform-Specific     | slack_analysis, discourse_analysis, hackernews_analysis                        | Platform-tailored analysis                  |

### 16.5 Templates (Non-LLM)

Non-LLM string templates live in `src/templates/`, separate from the prompt registry:

- **automation_templates.py** — Response, summary, assignment, report, follow-up templates
- **issue_bodies.py** — Bug report, dev task, feature request, support ticket, escalation templates

______________________________________________________________________

## 17. Billing & Usage Tracking

### 17.1 Credit System

**Source**: `src/services/billing/usage_tracking.py`

**Cost calculation:**

| Model       | Input (per 1M tokens) | Output (per 1M tokens) |
| ----------- | --------------------- | ---------------------- |
| GPT-4o      | $2.50                 | $10.00                 |
| GPT-4o-mini | $0.15                 | $0.60                  |

Minimum: 1 cent per call (rounds up).

### 17.2 Usage Tracking

**Table**: `org_usage_tracking`

- `credit_balance_cents` — Pre-purchased credits
- `total_usage_cents` — Cumulative usage
- `is_unlimited` — Bypass credit checks

**Operations:**

- `check_credit_available(org_id)` → `(has_credit, used_cents, is_paid)`
- `deduct_credit(org_id, model, input_tokens, output_tokens)` — Deduct after each LLM call

### 17.3 Stripe Integration

**Source**: `src/routers/payments.py`

- `POST /payments/create-checkout` — Creates Stripe checkout session
- `POST /payments/webhook` — Handles `checkout.session.completed` events
- `GET /payments/status/{org_id}` — Returns payment status
- `POST /payments/verify-session` — Verifies completed sessions

### 17.4 Free Tier Limits

Pipeline report generation is gated by `PipelineReportUsage`:

- Free tier has limited report count
- Paid orgs or `is_unlimited` flag bypass limits
- `increment_report_usage()` raises error if limit reached

### 17.5 Anonymous Usage

Anonymous users have their own credit tracking:

- `anonymous_user.credit_balance_cents` / `total_usage_cents`
- `anonymous_usage_log` tracks per-call usage (endpoint, model, tokens, cost)
- Credits can be converted when anonymous user creates an account

______________________________________________________________________

## 18. Infrastructure & DevOps

### 18.1 Database

- **Engine**: PostgreSQL with `psycopg2` driver
- **ORM**: SQLModel (SQLAlchemy + Pydantic)
- **Migrations**: Alembic (auto-run on app startup via `alembic upgrade head`)
- **Session**: `get_session()` generator with `expire_on_commit=False`

### 18.2 Environment Variables

| Variable                                                  | Purpose                    |
| --------------------------------------------------------- | -------------------------- |
| `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`, `DB_NAME` | Database connection        |
| `GOOGLE_CLIENT_ID`                                        | Google OAuth               |
| `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`                | GitHub OAuth               |
| `SENDGRID_API`                                            | Email delivery             |
| `FERNET_KEY`                                              | API key encryption         |
| `JIRA_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`                | Jira integration           |
| `GITHUB_TOKEN`                                            | GitHub issue creation      |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`              | Payments                   |
| `FRONTEND_URL`, `FRONTEND_PORT`                           | Frontend configuration     |
| `VITE_BE_API_URL`, `VITE_FRONTEND_URL`                    | Build-time frontend config |

### 18.3 CI/CD Workflows

GitHub Actions workflows in `.github/workflows/`:

| Workflow                  | Trigger              | Description                       |
| ------------------------- | -------------------- | --------------------------------- |
| `build.yaml`              | Frontend changes     | Node 22, npm ci + build           |
| `pytest.yaml`             | Backend changes      | Python 3.13, pytest with coverage |
| `ruff.yaml`               | Backend changes      | Ruff lint check                   |
| `pyright.yaml`            | Backend changes      | Type checking                     |
| `eslint.yaml`             | Frontend changes     | ESLint                            |
| `prettier.yaml`           | Frontend changes     | Prettier formatting               |
| `spellcheck.yaml`         | All changes          | cspell                            |
| `mdformat.yaml`           | Markdown changes     | Markdown formatting               |
| `shellcheck.yaml`         | Shell script changes | Shell script linting              |
| `pr-title-min-length.yml` | PRs                  | Commit message length enforcement |

### 18.4 Local Development Commands

```bash
# Full validation (mirrors CI)
npm run validate

# Quick checks (lint + format only)
npm run validate:quick

# Backend only
npm run validate:backend

# Frontend only
npm run validate:frontend

# Auto-fix
./scripts/validate.sh --fix

# Individual checks
npm run lint                    # All linting
npm run format:check            # Format check without changes
npm run typecheck               # Type checking
npm test                        # All tests
```

### 18.5 Pre-commit Hooks

**Backend**: pre-commit framework with Ruff (lint + format) and Pyright (types) **Frontend**: Husky
\+ lint-staged with ESLint --fix, Prettier --write, and Jest on staged files

Setup: `./scripts/setup-hooks.sh`

### 18.6 Code Quality Standards

| Area                  | Tool       | Config                         |
| --------------------- | ---------- | ------------------------------ |
| Python linting        | Ruff       | Line length 100, double quotes |
| Python formatting     | Ruff       | Consistent with linting        |
| Python types          | Pyright    | Strict mode                    |
| TypeScript linting    | ESLint     | React/TypeScript plugins       |
| TypeScript formatting | Prettier   | Semicolons, single quotes      |
| TypeScript types      | TypeScript | Strict mode                    |
| Spelling              | cspell     | Custom dictionary              |
| Shell scripts         | shellcheck | Standard rules                 |
| Markdown              | mdformat   | Consistent formatting          |

______________________________________________________________________

## Appendix A: Cross-Service Dependency Map

```
Pipeline Service
    ├─→ Pipeline Executor (source routing)
    │    ├─→ Slack Pipeline → Orchestrator → Customer/Product Pipeline
    │    ├─→ Discourse Pipeline → Orchestrator → Topic Pipeline
    │    ├─→ HackerNews Pipeline → Orchestrator → Thread Pipeline
    │    ├─→ Discord Pipeline → (converts to Slack format) → Slack Pipeline
    │    └─→ WhatsApp/File Pipeline → Zip Executor
    │
    ├─→ Aggregation Orchestrator (triggered after pipeline run)
    │    ├─→ Fact Builder
    │    ├─→ Quality Validator
    │    ├─→ Enrichment
    │    ├─→ Metrics Calculator
    │    ├─→ Detectors (8 threshold-based)
    │    ├─→ Action Generator
    │    └─→ Playbook Learner
    │
    └─→ Pipeline Scheduler

Policy Engine
    └─→ Policy Service (reads rules, overrides)

Signal Classifier
    ├─→ Prompt Registry
    ├─→ Usage Tracking (credit deduction)
    ├─→ Issue Connectors (Jira, GitHub)
    └─→ Smart Action Service

Action Executor
    ├─→ Smart Action Service (per-app routing)
    ├─→ Issue Connectors (fallback)
    └─→ Slack Notifier

Organization Members
    ├─→ User Auth Service
    └─→ SendGrid Service (invitation emails)

Question Router
    └─→ Policy Service (routing rules)
```

## Appendix B: Data Flow — End to End

```
External Source (Slack/Discord/Discourse/HackerNews/WhatsApp/Email/Image/Audio)
    │
    ▼
Pipeline Executor → Platform-Specific Pipeline
    │
    ├─→ Rule Engine (deterministic, zero LLM cost)
    ├─→ LLM Extraction (ambiguous cases, Azure OpenAI)
    ├─→ Post-Processing (normalization, sanity checks)
    │
    ▼
Platform-Specific Tables (conv_slack_*, conv_discourse_*, etc.)
    │
    ├─→ Backward-Compatible Tables (conv_history, conv_sentiment, etc.)
    │
    ▼
Policy Engine
    │
    ├─→ Materialize Base (consolidate signal tables → conv_results_base)
    ├─→ Apply Policies (guardrails)
    ├─→ Apply Rules (preferences)
    ├─→ Apply Overrides (user corrections)
    ├─→ Sanity Checks
    │
    ▼
conv_results_final (with field_sources provenance)
    │
    ▼
Aggregation Pipeline (7 stages)
    │
    ├─→ agg_metric (50+ metrics at 5 grains)
    ├─→ agg_issue_cluster (semantic grouping)
    ├─→ agg_customer_health / agg_product_health (0-100 scores)
    ├─→ agg_alert_event (8 threshold detectors)
    ├─→ agg_action_item (recommended actions)
    ├─→ agg_issue_playbook (learned patterns)
    │
    ▼
Frontend Dashboard / API / Webhooks / Smart Actions
```
