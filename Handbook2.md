# Chapter 11: Entity Extraction

## 1. Definition

**Entity extraction** is the process of identifying and pulling out the specific named things from a user's query — the *who*, *what*, *where*, and *when* that the agent needs to perform its investigation.

From: *"Why has MarketID ABC stopped processing trades since 10:30?"*

Extracted entities:
```json
{
  "market_id": "ABC",
  "event_type": "trade processing stoppage",
  "time_reference": "10:30",
  "time_type": "relative_today"
}
```

Entities are the **parameters** of the investigation. Without them, the agent knows *what kind* of investigation to run (intent) but not *what to run it on*.

---

## 2. Why It Exists

Intent tells the agent *what to do*. Entities tell the agent *what to do it on*.

Consider these queries — same intent, different entities:
- *"Why has MarketID ABC stopped processing trades since 10:30?"*
- *"Why has MarketID XYZ stopped processing trades since 09:00?"*
- *"Why has the bond market stopped processing trades since yesterday?"*

All three are `OUTAGE_INVESTIGATION`. But without extracting the market ID and time, the agent cannot query the right data.

Entity extraction converts the ambiguity of natural language into precise parameters usable by APIs and databases.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│              POLICE DISPATCHER                          │
│                                                         │
│  Caller: "There's been an accident near the big mall    │
│           on the main road around noon!"               │
│                                                         │
│  Dispatcher extracts:                                   │
│  Location: Main road, near Westfield Mall               │
│  Event:    Vehicle accident                             │
│  Time:     ~12:00                                       │
│  Severity: (unknown — needs follow-up)                  │
│                                                         │
│  Without extraction: "Send someone... somewhere"        │
│  With extraction: Dispatches unit to precise location   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

### Entities Your RCA Platform Needs to Extract

```
┌──────────────────────────────────────────────────────────────┐
│                  ENTITY TYPES: RCA PLATFORM                  │
│                                                              │
│  MARKET ENTITIES                                             │
│  ├── market_id:       "ABC", "XYZ", "bond-market"            │
│  ├── market_type:     "equities", "bonds", "FX"             │
│  └── trading_session: "pre-market", "regular", "after-hours" │
│                                                              │
│  TIME ENTITIES                                               │
│  ├── absolute_time:   "10:30", "09:00 AM"                    │
│  ├── relative_time:   "since this morning", "last hour"      │
│  ├── duration:        "for 30 minutes"                       │
│  └── date:            "today", "yesterday", "Jan 15"         │
│                                                              │
│  SERVICE ENTITIES                                            │
│  ├── service_name:    "trade-processor", "order-router"      │
│  ├── environment:     "production", "UAT"                    │
│  └── region:          "India", "APAC", "US"                  │
│                                                              │
│  SYMPTOM ENTITIES                                            │
│  ├── symptom_type:    "not processing", "slow", "errors"     │
│  └── impact:          "stopped", "degraded", "intermittent"  │
└──────────────────────────────────────────────────────────────┘
```

From *"Why has MarketID ABC stopped processing trades since 10:30?"*:

```
Extracted:
  market_id     = "ABC"         (MARKET ENTITY)
  symptom_type  = "stopped"     (SYMPTOM ENTITY — implies outage, not latency)
  time_reference = "10:30"      (TIME ENTITY — absolute time, today)
  impact        = "stopped"     (SYMPTOM ENTITY)
  
Implied (not stated but inferable):
  environment   = "production"  (trading systems default assumption)
  date          = "today"       (no date mentioned → today)
```

---

## 5. Visual Workflow

### Entity Extraction Pipeline

```
Raw Query: "Why has MarketID ABC stopped processing trades since 10:30?"
     │
     ▼
┌─────────────────────────────────────────────────────┐
│              ENTITY EXTRACTOR (LLM)                 │
│                                                     │
│  System prompt:                                     │
│  "Extract entities from this trading platform query.│
│   Return JSON with: market_id, time_reference,      │
│   symptom_type, service_name (if mentioned),        │
│   environment (default: production)"                │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│                 EXTRACTED ENTITIES                  │
│  {                                                  │
│    "market_id": "ABC",                              │
│    "time_reference": "10:30",                       │
│    "time_type": "absolute_today",                   │
│    "symptom_type": "processing_stoppage",           │
│    "environment": "production",                     │
│    "confidence": {                                  │
│      "market_id": 0.99,                             │
│      "time_reference": 0.95,                        │
│      "environment": 0.80                            │
│    }                                                │
│  }                                                  │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
                    Entity Resolution
                    (Chapter 12)
```

### Ambiguous Entity Handling

```
Query: "Why did ABC stop at half ten?"

  "ABC"        → market_id? service_name? customer?
  "half ten"   → 10:30? 22:30? dialect for 9:30?

                    DISAMBIGUATION NEEDED

  Resolution strategies:
  ┌────────────────────────────────────────────────┐
  │  1. Domain default: "ABC" in trading context   │
  │     → market_id (high confidence)              │
  │                                                │
  │  2. Time normalization: "half ten" in UK       │
  │     English → 10:30 AM                         │
  │                                                │
  │  3. If still ambiguous → ask user:             │
  │     "Did you mean MarketID ABC at 10:30 AM?"   │
  └────────────────────────────────────────────────┘
```

### Entity Types by Query Pattern

```
┌──────────────────────────────────────────────────────────┐
│  Query Pattern              │ Key Entities               │
├─────────────────────────────┼────────────────────────────┤
│  "Why is X down since T?"   │ X=market, T=time           │
│  "X is slow for Y users"    │ X=service, Y=user_segment  │
│  "Compare X vs Y"           │ X, Y=comparison subjects   │
│  "Show me X between T1-T2"  │ X=metric, T1,T2=time range │
│  "Any issues in REGION?"    │ region=location entity     │
└──────────────────────────────────────────────────────────┘
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — Intent classified as `OUTAGE_INVESTIGATION`.

**Step 2** — Entity extractor receives the query. Configured with the entity schema for your platform.

**Step 3** — LLM extracts: `market_id=ABC`, `time=10:30`, `symptom=stopped processing`.

**Step 4** — Time normalization: "10:30" + no date → `2024-01-15T10:30:00` (today's date + local timezone).

**Step 5** — Confidence check: all entities above 0.85 threshold → proceed.

**Step 6** — Entities stored in agent state: `{ market_id: "ABC", start_time: "2024-01-15T10:30:00", ... }`

**Step 7** — Entity resolution takes over (Chapter 12): "ABC" → what is the internal service ID for MarketID ABC?

---

## 7. Enterprise Design

| Approach | Pros | Cons |
|----------|------|------|
| LLM extraction | Handles all phrasings naturally | Slower, costs tokens |
| Fine-tuned NER model | Fast, cheap | Needs training data per entity type |
| Regex + rules | Instant, predictable | Brittle, doesn't handle variations |
| Hybrid (rules for structured, LLM for free-form) | Best coverage | Complexity |

For a financial platform with well-defined entity types (MarketID format, time patterns), a hybrid approach is practical.

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| No time normalization | "10:30" without date/timezone → wrong Loki query time range |
| Accepting low-confidence entities | Agent queries wrong MarketID silently |
| No missing entity handling | Query without a MarketID → agent queries all markets |
| Not extracting implicit entities | "Is the system healthy?" → environment not extracted → defaults to wrong env |

---

## 9. Summary

**Key Ideas:**
- Entities are the parameters of the investigation — market, time, service, symptom
- Intent = what to do; Entities = what to do it on
- Time entities need normalization (relative → absolute, timezone-aware)
- Low-confidence extractions should trigger clarification, not guessing

**Mental Model:**
> Entity extraction is form-filling from natural language. The form has fields: MarketID, Time, Service, Symptom. The user fills it in conversationally. Your extractor reads their words and populates the form. Missing required fields → ask the user.

---

# Chapter 12: Entity Resolution

## 1. Definition

**Entity resolution** is the process of mapping extracted entity strings (what the user said) to actual system identifiers (what your systems understand).

Entity extraction gives you: `market_id = "ABC"`

Entity resolution answers: *What exactly IS "ABC" in our systems?*
- Is it an internal market code? `MARKET_CODE: ABC_EQ_NSE`
- Which Kafka topic does it use? `kafka.topic: market-abc-trades`
- Which service processes it? `service: trade-processor-abc-v2`
- Which Grafana dashboard shows it? `dashboard_id: market-abc-ops`

Without resolution, your tools receive *user-facing names*, not *system identifiers*. APIs fail silently or return wrong data.

---

## 2. Why It Exists

Real systems have a naming gap between what users say and what systems understand:

| What user says | What the system needs |
|---------------|----------------------|
| "MarketID ABC" | Internal code: `MARKET_ABC_NSE_EQ` |
| "trade processor" | Service name: `svc-trade-proc-abc-prod-v2` |
| "since 10:30" | UTC timestamp: `2024-01-15T05:00:00Z` |
| "the bond market" | Market group: `[MARKET_001, MARKET_007, MARKET_042]` |

Entity resolution bridges this gap using reference data — your system's master data about markets, services, and their relationships.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│                  HOSPITAL PATIENT LOOKUP                │
│                                                         │
│  Receptionist says: "John Smith is here"               │
│                                                         │
│  System needs: Patient ID P-2024-04782                 │
│                Ward: 3B, Bed 12                        │
│                Attending physician: Dr. Patel (ID 991) │
│                                                         │
│  Resolution: "John Smith" → patient record lookup       │
│              → all system IDs retrieved                 │
│                                                         │
│  Without it: Doctor goes to wrong ward.                 │
│  In your system: Loki query uses wrong service name.    │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

**Entity: `market_id = "ABC"`**

Resolution process:

```
┌──────────────────────────────────────────────────────────────┐
│                   ENTITY RESOLUTION: MarketID "ABC"          │
│                                                              │
│  STEP 1: Look up "ABC" in Market Registry (MongoDB)          │
│  Result:                                                     │
│  {                                                           │
│    "market_id": "ABC",                                       │
│    "display_name": "ABC Equities",                           │
│    "internal_code": "MARKET_ABC_NSE_EQ",                     │
│    "kafka_topic": "market-abc-trade-events",                 │
│    "consumer_group": "abc-trade-processor-grp",              │
│    "loki_label": { "market": "abc", "service": "trade-proc"} │
│    "grafana_dashboard": "d/abc-market-ops",                  │
│    "prometheus_labels": { "market_id": "ABC" },              │
│    "team_owner": "equities-ops",                             │
│    "pagerduty_escalation": "PD-EQ-OPS"                      │
│  }                                                           │
│                                                              │
│  STEP 2: Resolve time "10:30" (today, UTC+5:30 IST)         │
│  Result: start_time = 2024-01-15T05:00:00Z                  │
│          window_start = 2024-01-15T04:45:00Z  (15min buffer) │
│          window_end   = 2024-01-15T05:30:00Z                 │
└──────────────────────────────────────────────────────────────┘
```

Now every tool call uses resolved identifiers:
- Loki query: `{market="abc", service="trade-proc"}` ← from resolution
- Kafka check: consumer group `abc-trade-processor-grp` ← from resolution
- Prometheus: `{market_id="ABC"}` ← from resolution

---

## 5. Visual Workflow

### Entity Resolution Architecture

```
Extracted Entities                  Resolved Entities
{ market_id: "ABC" }                { internal_code: "MARKET_ABC_NSE_EQ"
{ time: "10:30" }          ──▶       kafka_topic: "market-abc-trade-events"
{ symptom: "stopped" }               loki_labels: {market:"abc"}
                                     time_utc: "2024-01-15T05:00:00Z"
                                     ... }
         │                                    │
         ▼                                    ▼
┌────────────────────┐          ┌─────────────────────────────┐
│   ENTITY RESOLVER  │          │     TOOL CALLS USE          │
│                    │          │     RESOLVED VALUES ONLY    │
│  ├── Market Registry          │                             │
│  │   (MongoDB)     │          │  Loki: {market="abc"}       │
│  ├── Service Catalog          │  Kafka: abc-trade-proc-grp  │
│  │   (MongoDB)     │          │  Prom:  {market_id="ABC"}   │
│  └── Time Resolver │          └─────────────────────────────┘
│      (TZ + UTC)    │
└────────────────────┘
```

### Resolution Failure Handling

```
Resolution attempt for: "ABC"
         │
    ┌────┴────┐
    │ Found?  │
    └────┬────┘
    YES  │  NO
         │   │
         │   ▼
         │  ┌──────────────────────────────────────────┐
         │  │  FUZZY MATCH                              │
         │  │  Try: "ABCD", "AB", "ABC-OLD"            │
         │  │  Return matches with confidence scores    │
         │  └──────────────┬───────────────────────────┘
         │           Found?│
         │        YES      │  NO
         │          │      ▼
         │          │  ┌──────────────────────────────────┐
         │          │  │  ASK USER                        │
         │          │  │  "I couldn't find MarketID 'ABC' │
         │          │  │   Did you mean: ABC-EQ, ABC-FX?" │
         │          │  └──────────────────────────────────┘
         ▼          ▼
    Use resolved entity
```

### Time Resolution Examples

```
┌──────────────────────────────────────────────────────────────┐
│                    TIME ENTITY RESOLUTION                    │
│                                                              │
│  "10:30"           → 2024-01-15T05:00:00Z  (today, IST→UTC) │
│  "since this morning" → 2024-01-15T03:30:00Z (market open)  │
│  "last hour"       → [2024-01-15T07:00:00Z to 08:00:00Z]    │
│  "yesterday at 2"  → 2024-01-14T08:30:00Z  (yesterday, UTC) │
│  "past 30 minutes" → [now-30min to now]                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — Entities arrive from extraction: `{market_id: "ABC", time: "10:30"}`

**Step 2** — Market Resolver queries the Market Registry: `db.markets.findOne({market_id: "ABC"})`

**Step 3** — Full market record returned: Kafka topic, Loki labels, Prometheus labels, team owner.

**Step 4** — Time Resolver converts "10:30" (user's timezone: IST) → UTC: `2024-01-15T05:00:00Z`

**Step 5** — Time window created: 15 minutes before 10:30 → `10:15 IST` to `10:30+45min = 11:15 IST`

**Step 6** — Resolved entity package assembled and stored in agent state.

**Step 7** — All subsequent tool calls use resolved identifiers, not user-facing names.

---

## 7. Enterprise Design

```
┌──────────────────────────────────────────────────────────────┐
│              ENTITY RESOLUTION INFRASTRUCTURE                │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 RESOLUTION CACHE                     │   │
│  │  Redis or in-memory cache for frequent lookups       │   │
│  │  Market ABC → resolved record (TTL: 5 minutes)       │   │
│  │  Purpose: avoid DB lookup on every query             │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  MASTER DATA STORE                   │   │
│  │  MongoDB / Snowflake for reference data              │   │
│  │  Market registry, service catalog, team ownership    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │               FUZZY MATCHING ENGINE                  │   │
│  │  Elasticsearch / vector similarity for near-matches  │   │
│  │  Handles typos, abbreviations, aliases               │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Using user-facing names in API calls | Loki returns no results; agent concludes "no issues found" (wrong) |
| No timezone handling for time entities | Time window is 5.5 hours off for IST users |
| No cache for resolution results | Every tool call triggers a DB lookup; latency explodes |
| No fuzzy matching | "trade processor" vs "trade-processor" causes resolution failure |

---

## 9. Summary

**Key Ideas:**
- Entity resolution maps human-facing names to system identifiers
- Without it, tools receive wrong parameters and return wrong (or no) data
- Time resolution is particularly critical — timezones and relative expressions must be normalized
- Cache resolution results; master data changes infrequently

**Mental Model:**
> Entity resolution is the translation layer between human vocabulary and system vocabulary. The user says "MarketID ABC". The system needs `{kafka_topic: "market-abc-trade-events", loki_label: "market=abc", ...}`. The resolver is the dictionary that translates between them.

---

# Chapter 13: Semantic Layer

## 1. Definition

The **semantic layer** is a shared vocabulary and mapping that translates between what users say, what the agent understands, and what your systems need.

It is the authoritative source of:
- What entities exist in your domain (markets, services, teams)
- What they're called (display names, aliases, codes)
- How they relate to each other (Market ABC is processed by Service X which runs on Cluster Y)
- What metrics, logs, and data are associated with them

The semantic layer is what makes your agent *domain-aware* rather than *generically capable*.

---

## 2. Why It Exists

Without a semantic layer:
- Each tool needs its own mapping logic
- Each prompt needs to explain domain concepts
- User queries must use exact system names
- Adding a new market means updating every integration

With a semantic layer:
- One place defines what "MarketID ABC" means
- All tools look up this definition
- Users use natural names; the layer translates
- Adding a new market means one registry update

The semantic layer is the **single source of truth** for your domain vocabulary.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│                  HOSPITAL MASTER PATIENT INDEX          │
│                                                         │
│  Every department uses different terms:                 │
│  - Billing: "Patient #84782"                           │
│  - Ward: "Room 3B, Bed 12"                             │
│  - Pharmacy: "John Smith, DOB 1965-03-15"              │
│                                                         │
│  Master Patient Index (MPI) is the semantic layer:     │
│  All three identifiers map to the same patient.        │
│  Add a new identifier once → all systems see it.       │
│                                                         │
│  Your semantic layer = MPI for your trading platform   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

### The RCA Platform Semantic Layer

```
┌──────────────────────────────────────────────────────────────┐
│                   SEMANTIC LAYER SCHEMA                      │
│                                                              │
│  MARKET_ENTITY: "ABC"                                        │
│  ├── display_name: "ABC Equities Market"                     │
│  ├── aliases: ["ABC", "abc", "ABC-EQ", "Market Alpha"]       │
│  ├── internal_code: "MARKET_ABC_NSE_EQ"                      │
│  ├── type: "equities"                                        │
│  ├── exchange: "NSE"                                         │
│  │                                                           │
│  ├── SYSTEM_MAPPINGS:                                        │
│  │   ├── kafka_topic: "market-abc-trade-events"              │
│  │   ├── consumer_group: "abc-trade-processor-grp"           │
│  │   ├── loki_labels: {market: "abc", svc: "trade-proc"}     │
│  │   ├── prometheus_labels: {market_id: "ABC"}               │
│  │   ├── grafana_dashboard: "d/abc-market-ops"               │
│  │   └── mongodb_collection: "trades_abc"                    │
│  │                                                           │
│  ├── RELATIONSHIPS:                                          │
│  │   ├── processed_by: "trade-processor-service"             │
│  │   ├── depends_on: ["kafka-cluster-1", "mongodb-primary"]  │
│  │   └── team_owner: "equities-operations"                   │
│  │                                                           │
│  └── METADATA:                                               │
│      ├── criticality: "HIGH"                                 │
│      ├── sla_uptime: "99.99%"                                │
│      └── escalation_path: "PD-EQ-OPS → VP-Trading-Tech"     │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Visual Workflow

### Semantic Layer as the Central Hub

```
                        USER QUERY
                            │
                            ▼
                     ENTITY EXTRACTION
                     "MarketID ABC"
                            │
                            ▼
              ┌─────────────────────────┐
              │      SEMANTIC LAYER     │
              │   (Single Source of     │
              │       Truth)            │
              └────────────┬────────────┘
                           │
       ┌───────────────────┼──────────────────┐
       │                   │                  │
       ▼                   ▼                  ▼
┌───────────┐       ┌────────────┐    ┌──────────────┐
│   LOKI    │       │ PROMETHEUS │    │    KAFKA     │
│  labels:  │       │  labels:   │    │  consumer    │
│  market=  │       │  market_id │    │  group:      │
│  abc      │       │  ="ABC"    │    │  abc-trade-  │
│  svc=     │       │            │    │  proc-grp    │
│  trade-   │       │            │    │              │
│  proc     │       │            │    │              │
└───────────┘       └────────────┘    └──────────────┘
```

### Semantic Layer vs. No Semantic Layer

```
WITHOUT SEMANTIC LAYER:

Agent Prompt:  "Query Loki with label market='abc'"
               "Query Prometheus with label market_id='ABC'"
               "Check Kafka group 'abc-trade-processor-grp'"

Problem: This knowledge is buried in every prompt.
         Update the Kafka group name → update every prompt.

─────────────────────────────────────────────────────────────

WITH SEMANTIC LAYER:

Semantic layer defines all of the above once.
Agent prompt:  "Look up MarketID ABC in the semantic layer.
                Use the returned system mappings for all queries."

Update the Kafka group name → update the semantic layer once.
All agents automatically use the new value.
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — User query: *"Why has MarketID ABC stopped processing trades since 10:30?"*

**Step 2** — Entity extracted: `market_id = "ABC"`

**Step 3** — Semantic layer lookup: `GET /semantic-layer/market/ABC`

**Step 4** — Full entity record returned (all system mappings, relationships, metadata).

**Step 5** — Agent now knows:
- Which Kafka topic to check
- Which Loki labels to filter on
- Which Prometheus labels to use
- Which team owns this market
- What the escalation path is

**Step 6** — All tool calls use values from the semantic layer record — no hardcoded identifiers anywhere.

---

## 7. Enterprise Design

```
┌──────────────────────────────────────────────────────────────┐
│              SEMANTIC LAYER IMPLEMENTATION OPTIONS           │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  SIMPLE: MongoDB collection                         │    │
│  │  markets, services, relationships stored as docs    │    │
│  │  Good for: < 1000 entities, simple relationships    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ADVANCED: Knowledge Graph (Neo4j / Neptune)        │    │
│  │  Entities as nodes, relationships as edges          │    │
│  │  Good for: Complex dependencies, graph traversal    │    │
│  │  "What depends on kafka-broker-1?" → graph query    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  HYBRID: MongoDB + Redis cache + Search index        │    │
│  │  Fast lookup + fuzzy search + caching               │    │
│  │  Best for: Production enterprise use                │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| Embedding system identifiers in prompts | Prompt needs updating every time a service name changes |
| No versioning of semantic layer | Historical investigations break when mappings change |
| Semantic layer not used by tools directly | Tools each maintain their own mapping logic → drift |
| No ownership/team data in semantic layer | Agent can't route escalations correctly |

---

## 9. Summary

**Key Ideas:**
- The semantic layer is the domain vocabulary — all entities, their identifiers, and relationships
- It decouples human-facing names from system-facing identifiers
- All tools and agents should look up system identifiers from the semantic layer, not hardcode them
- It is the most critical piece of infrastructure for a domain-specific agent

**Mental Model:**
> The semantic layer is the Rosetta Stone of your platform. It translates between the language your users speak (market names, service names) and the language your systems speak (Kafka topics, Loki labels, Prometheus selectors). Without it, every agent integration is a custom translation exercise.

---

# Chapter 14: Planner

## 1. Definition

The **planner** is the component of an agent that decides *what to do next* given the current state of the investigation. It takes the intent, resolved entities, and any evidence collected so far, and generates a sequence of actions (tool calls, sub-queries, or synthesis steps).

The planner answers: *"Given what I know and what I need to find out, what is the best next step?"*

---

## 2. Why It Exists

A single LLM call cannot complete a multi-step investigation. Finding the root cause of a trade outage requires:
1. Checking error logs
2. Checking infrastructure health
3. Checking data pipeline state
4. Correlating across all three
5. Forming a hypothesis
6. Verifying the hypothesis
7. Generating remediation steps

Without a planner, the agent either:
- Tries to do all of this in one context-overloading prompt
- Stops after the first tool call and generates a premature answer
- Repeats the same investigation steps in every query

The planner manages the *multi-step reasoning loop*.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│              INCIDENT COMMANDER                         │
│                                                         │
│  When an outage is reported, the Incident Commander:   │
│                                                         │
│  1. Assesses: What do we know? What do we need?        │
│  2. Plans: "First check the service, then logs,        │
│             then infrastructure"                        │
│  3. Delegates: Assigns specific tasks to team members  │
│  4. Reviews: As results come in, adjusts the plan      │
│  5. Concludes: When enough evidence → root cause       │
│                                                         │
│  The Planner = the Incident Commander                  │
│  The tools = the team members                          │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

**Investigation: "Why has MarketID ABC stopped processing trades since 10:30?"**

```
PLANNER REASONING CHAIN:

Turn 1: "I need to understand the outage. Starting with error logs."
  → ACTION: Call loki_log_query(market=abc, severity=ERROR, time=10:15-10:45)
  → RESULT: 147 error lines. FATAL "Connection refused: kafka-broker-1:9092" at 10:29:58
  
Turn 2: "Loki shows Kafka connection failure. Need to confirm with Kafka state."
  → ACTION: Call kafka_consumer_group_state(group=abc-trade-processor-grp)
  → RESULT: lag=45000, status=LAGGING, last_committed=10:29:45
  
Turn 3: "Kafka lag confirms the failure. Is this a broker issue or the processor?"
  → ACTION: Call prometheus_metric(metric=kafka_broker_up, broker=kafka-broker-1, time=10:15-10:45)
  → RESULT: kafka_broker_up=0 from 10:29:51
  
Turn 4: "Evidence is sufficient. kafka-broker-1 went down at 10:29:51, causing cascade."
  → ACTION: Generate structured RCA output
  → DONE
```

Four reasoning steps. The planner decided each next action based on what the previous step revealed.

---

## 5. Visual Workflow

### Planner State Machine

```
                    ┌─────────────┐
          ┌────────▶│  PLANNING   │◀──────────────┐
          │         │  (What next?)│               │
          │         └──────┬──────┘               │
          │                │                      │
          │         ┌──────▼──────┐               │
          │         │   ACTING    │               │
          │         │  (Tool call)│               │
          │         └──────┬──────┘               │
          │                │                      │
          │         ┌──────▼──────┐               │
          │         │ OBSERVING   │               │
          │         │ (Review     │  Insufficient │
          │         │  results)   │  evidence     │
          │         └──────┬──────┘               │
          │                │                      │
          │      Sufficient│evidence              │
          │         ┌──────▼──────┐               │
          └─────────│ SUFFICIENT? │───────────────┘
                    └──────┬──────┘
                     YES   │
                           ▼
                    ┌─────────────┐
                    │  CONCLUDING │
                    │  (Generate  │
                    │   output)   │
                    └─────────────┘
```

### Plan Types

```
┌──────────────────────────────────────────────────────────────┐
│                   PLANNING STRATEGIES                        │
│                                                              │
│  SEQUENTIAL (default for RCA)                                │
│  Step 1 → Step 2 → Step 3 → Conclude                        │
│  Each step informs the next                                  │
│  Good for: Investigations where findings guide next steps    │
│                                                              │
│  PARALLEL (for independent data collection)                  │
│  Step 1 ─┬─▶ Loki query                                      │
│           ├─▶ Prometheus query   All at once                 │
│           └─▶ Kafka query                                    │
│  Good for: When data sources are independent                 │
│                                                              │
│  HIERARCHICAL (for complex investigations)                   │
│  Plan ──▶ Sub-plan A ──▶ Sub-plan B                         │
│  Each sub-plan has its own steps                             │
│  Good for: Multi-system, multi-team investigations           │
│                                                              │
│  REACTIVE (adapts to findings)                               │
│  Starts with minimal plan, extends based on results          │
│  Good for: Unknown problem types, exploratory investigation  │
└──────────────────────────────────────────────────────────────┘
```

### Planner in the Agent Architecture

```
Intent + Entities
      │
      ▼
┌─────────────────────────────────────────────────────┐
│                    PLANNER                          │
│                                                     │
│  Input:  Intent=OUTAGE_INV, Market=ABC, Time=10:30  │
│  Output: Ordered list of investigation steps        │
│                                                     │
│  Step 1: Query Loki (error logs)                    │
│  Step 2: Query Kafka (consumer state)               │
│  Step 3: Query Prometheus (service health)          │
│  Step 4: Correlate and conclude                     │
└──────────────────────┬──────────────────────────────┘
                       │
           ┌───────────┼──────────────┐
           ▼           ▼              ▼
          Loki       Kafka        Prometheus
           │           │              │
           └───────────┼──────────────┘
                       │
                       ▼
                  Correlator
                       │
                       ▼
                  RCA Output
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — Intent: `OUTAGE_INVESTIGATION`. Entities: `{market_id: ABC, time: 10:30}`.

**Step 2** — Planner generates initial plan based on intent template:
```
OUTAGE_INVESTIGATION template:
  1. Query error logs (Loki)
  2. Query infrastructure state (Kafka + Prometheus)
  3. Check service health (Grafana)
  4. Correlate findings → root cause
```

**Step 3** — Execute Step 1: Loki returns error logs. Planner sees "Connection refused to Kafka broker".

**Step 4** — Planner updates plan: "The Kafka connection issue is key. Escalate Kafka investigation. Skip Grafana for now."

**Step 5** — Execute updated Step 2: Kafka state + Prometheus broker health confirm kafka-broker-1 is down.

**Step 6** — Planner evaluates: *Do I have sufficient evidence to conclude?* Yes — broker down at 10:29:51, consumer lag confirms, error logs confirm.

**Step 7** — Planner triggers conclusion: Generate structured RCA output.

---

## 7. Enterprise Design

| Planning Architecture | Description | When to Use |
|----------------------|-------------|-------------|
| **ReAct (Reason + Act)** | LLM plans one step, acts, observes, repeats | Most common; flexible |
| **Plan-and-Execute** | LLM generates full plan upfront, then executes | When steps are well-defined |
| **Chain of Thought** | LLM reasons through a plan in text before acting | Complex, novel problems |
| **Tree of Thought** | LLM explores multiple plan branches | When path is uncertain |

For your RCA platform: **ReAct** for standard outage investigations (adaptive), **Plan-and-Execute** for scheduled analysis reports (predictable).

---

## 8. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| No max steps limit | Planner loops indefinitely |
| Planner ignores previous findings | Re-queries the same Loki endpoint repeatedly |
| Static plan for all intents | Latency investigation follows outage steps |
| No backtracking | One wrong hypothesis leads investigation in wrong direction with no recovery |

---

## 9. Summary

**Key Ideas:**
- The planner manages the multi-step reasoning loop
- Each planning cycle: decide next action → execute → observe → decide again
- The plan adapts based on what each step reveals
- Always cap the number of planning steps to prevent runaway agents

**Mental Model:**
> The planner is the investigation strategy. A good investigator doesn't just dump all evidence at once — they form a hypothesis from the first clue, then test it with the next step, updating their hypothesis as evidence accumulates. That's the planning loop.

---

# Chapter 15: Routing

## 1. Definition

**Routing** is the decision of *which agent, tool, or workflow* should handle a given query or sub-task.

It is different from planning (what steps to take) — routing is about *who or what* performs those steps.

Routing decisions:
- Should this query go to the RCA agent or the status agent?
- Should this tool call go to Loki or to Tempo?
- Should this investigation be escalated to a human or handled automatically?
- Is this a known problem type → specialized handler, or novel → general agent?

---

## 2. Why It Exists

Not all queries are the same, and not all agents/tools are equally suited to all queries.

A generalist agent trying to handle every query type is:
- Expensive (uses maximum context and reasoning every time)
- Slow (no optimization for common cases)
- Less accurate (generalist prompts are vaguer than specialist prompts)

Routing sends queries to the right handler — optimizing for cost, speed, and accuracy.

---

## 3. Real-World Analogy

```
┌─────────────────────────────────────────────────────────┐
│                  BANK CALL CENTER                       │
│                                                         │
│  Customer calls: "I have a question"                   │
│                                                         │
│  IVR Router listens and routes:                         │
│  "Press 1 for account balance" → automated system      │
│  "Press 2 for loan queries"    → loans specialist      │
│  "Press 3 for fraud"           → fraud team (priority) │
│  "Something else"              → general agent         │
│                                                         │
│  Your agent router does the same — matches query type  │
│  to the right handler before any investigation begins. │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Observability Example

### Query Routing in the RCA Platform

```
User Query
     │
     ▼
┌────────────────────────────────────────────────────────┐
│                   QUERY ROUTER                         │
│                                                        │
│  Rule 1: Is this a "why is X down" pattern?           │
│           → Route to OUTAGE_INVESTIGATION_AGENT        │
│                                                        │
│  Rule 2: Is this a "how is X performing" pattern?     │
│           → Route to PERFORMANCE_ANALYSIS_AGENT        │
│                                                        │
│  Rule 3: Is this a "show me X data" pattern?          │
│           → Route to DATA_QUERY_AGENT                  │
│                                                        │
│  Rule 4: Is this a "what should I do about X" pattern?│
│           → Route to ADVISORY_AGENT                    │
│                                                        │
│  Rule 5: None of the above?                           │
│           → Route to GENERAL_AGENT                     │
└────────────────────────────────────────────────────────┘
```

### Tool-Level Routing

```
Within OUTAGE_INVESTIGATION_AGENT:

Symptom: "connection refused"  → Route to: Infrastructure tools (Kafka, Prometheus)
Symptom: "data missing"        → Route to: Data tools (MongoDB, Snowflake)
Symptom: "service crashed"     → Route to: Log tools (Loki) + Process health
Symptom: "slow performance"    → Route to: Trace tools (Tempo) + Metrics
```

---

## 5. Visual Workflow

### Multi-Level Routing Architecture

```
User Query
     │
     ▼
┌────────────────────────────────────────────────────────────┐
│                 LEVEL 1: INTENT ROUTER                     │
│                                                            │
│  OUTAGE → Outage Agent                                     │
│  LATENCY → Performance Agent                               │
│  DATA_QUALITY → Data Quality Agent                         │
│  STATUS → Status Agent                                     │
└────────────────────────────────────────────────────────────┘
     │
     ▼ (e.g., OUTAGE → Outage Agent)
┌────────────────────────────────────────────────────────────┐
│                LEVEL 2: TOOL ROUTER                        │
│                                                            │
│  Based on extracted symptom:                               │
│  CONNECTION → [Kafka, Network tools]                       │
│  CRASH      → [Loki, Process health]                       │
│  TIMEOUT    → [Prometheus, Tempo]                          │
└────────────────────────────────────────────────────────────┘
     │
     ▼
┌────────────────────────────────────────────────────────────┐
│              LEVEL 3: ESCALATION ROUTER                    │
│                                                            │
│  If confidence < 0.6 → Human review                        │
│  If criticality = HIGH → Auto-escalate to PagerDuty        │
│  If investigation > 5 steps → Escalate to senior SRE       │
└────────────────────────────────────────────────────────────┘
```

### Routing Decision Matrix

```
┌───────────────────┬────────────────┬────────────────┬──────────────┐
│  Intent           │  Confidence    │  Criticality   │  Route To    │
├───────────────────┼────────────────┼────────────────┼──────────────┤
│ OUTAGE_INV        │ HIGH (>0.85)   │ HIGH           │ Auto + Alert │
│ OUTAGE_INV        │ HIGH           │ LOW            │ Auto only    │
│ OUTAGE_INV        │ LOW (<0.6)     │ ANY            │ Human first  │
│ LATENCY_INV       │ ANY            │ ANY            │ Perf. Agent  │
│ DATA_QUALITY      │ ANY            │ ANY            │ DQ Agent     │
│ GENERAL_QUERY     │ ANY            │ ANY            │ General Agent│
└───────────────────┴────────────────┴────────────────┴──────────────┘
```

---

## 6. Step-by-Step Walkthrough

**Step 1** — Query arrives. Router reads the intent classification result: `OUTAGE_INVESTIGATION`.

**Step 2** — Level 1 routing: dispatches to `Outage Investigation Agent`.

**Step 3** — Agent begins investigation. Loki returns "Connection refused: kafka-broker".

**Step 4** — Level 2 routing: symptom "connection refused" → route to infrastructure-focused tool set (Kafka state, Prometheus broker health).

**Step 5** — Evidence collected. Confidence: HIGH. Market ABC criticality: HIGH.

**Step 6** — Level 3 routing: HIGH confidence + HIGH criticality → auto-generate RCA + trigger PagerDuty for the equities-ops team.

---

## 7. Enterprise Design

| Routing Pattern | Implementation | Best For |
|-----------------|----------------|----------|
| **Rules-based** | If-then logic on intent + confidence | Predictable, auditable |
| **LLM-based** | LLM decides which agent/tool | Flexible, handles novel cases |
| **Embeddings-based** | Semantic similarity to known patterns | Large query type catalog |
| **Hybrid** | Rules for known cases, LLM for unknowns | Production best practice |

---

## 8. Summary

**Key Ideas:**
- Routing directs queries to the right handler before investigation begins
- Multi-level routing: intent level, tool level, escalation level
- Routing decisions should be deterministic and auditable in production
- Routing failures (wrong handler selected) are silent and dangerous

**Mental Model:**
> Routing is the dispatcher who reads an incident report and decides: SWAT team? Traffic unit? Fire department? Paramedics? Getting the right team dispatched immediately is more important than having all teams on standby for every incident.

---

# Chapter 16: Capabilities

## 1. Definition

**Capabilities** are the specific actions or knowledge-retrieval operations that an agent can perform. They are higher-level than tools — a capability might use one or more tools to accomplish a goal.

Tool: `query_loki_logs(service, severity, time_range)`
Capability: `investigate_trade_outage(market_id, time)` — which *internally* calls Loki + Prometheus + Kafka tools.

Capabilities are the agent's *superpowers*, expressed in domain language rather than technical API language.

---

## 2. Why It Exists

Tools are low-level (one API call). Capabilities are the meaningful units of work that combine tools into domain-relevant operations.

This abstraction matters because:
- Users think in capabilities ("investigate the outage"), not tools ("call the Loki API")
- Capabilities can be described to other agents and orchestrators in domain terms
- Changing the underlying tools doesn't change the capability interface
- Capabilities can be composed to build more complex operations

---

## 3. Observability Example

### RCA Platform Capability Catalog

```
┌──────────────────────────────────────────────────────────────┐
│                    CAPABILITY CATALOG                        │
│                                                              │
│  INVESTIGATE_OUTAGE                                          │
│  Description: Investigate why a market/service stopped       │
│  Input: market_id, start_time                                │
│  Uses tools: [loki_query, kafka_state, prometheus_metrics]   │
│  Output: RCA JSON with root cause + confidence               │
│                                                              │
│  ANALYZE_PERFORMANCE                                         │
│  Description: Analyze latency/performance degradation        │
│  Input: service_name, time_range                             │
│  Uses tools: [prometheus_metrics, tempo_traces]              │
│  Output: Performance analysis with bottleneck identification │
│                                                              │
│  CHECK_DATA_QUALITY                                          │
│  Description: Validate trade data completeness               │
│  Input: market_id, date                                      │
│  Uses tools: [mongodb_query, snowflake_query]                │
│  Output: Data quality report with gap identification         │
│                                                              │
│  ESCALATE_INCIDENT                                           │
│  Description: Create incident and notify responsible team    │
│  Input: rca_result, severity                                 │
│  Uses tools: [pagerduty_create, slack_notify, jira_create]   │
│  Output: Incident ID + notification confirmation             │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Visual Workflow

### Capability vs. Tool Hierarchy

```
USER REQUEST: "Investigate why MarketID ABC stopped at 10:30"
                              │
                              ▼
              CAPABILITY: INVESTIGATE_OUTAGE
              (domain-level abstraction)
                              │
              ┌───────────────┼──────────────────┐
              │               │                  │
              ▼               ▼                  ▼
        TOOL: loki      TOOL: kafka_        TOOL: prometheus_
        _log_query      consumer_state      metrics
              │               │                  │
              ▼               ▼                  ▼
          Loki API        Kafka API          Prom API
```

### Capability Composition

```
Complex Task: "Investigate outage AND escalate if confirmed"
                              │
              ┌───────────────▼──────────────────┐
              │   CAPABILITY ORCHESTRATOR         │
              └───────────────┬──────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                                       │
          ▼                                       ▼
  INVESTIGATE_OUTAGE                    ESCALATE_INCIDENT
  (if root cause confirmed,             (triggered after
   confidence HIGH)                      investigation)
```

---

## 5. Summary

**Key Ideas:**
- Capabilities are the domain-language abstraction above tools
- One capability can use multiple tools
- Capabilities can be composed to build complex workflows
- Capabilities are the unit of description for agent-to-agent communication

**Mental Model:**
> Tools are verbs (query, fetch, write). Capabilities are sentences ("Investigate the outage", "Analyze performance"). Users think in sentences. Agents should be designed around sentences.

---

# Chapter 17 & 18: Tool Registry & Capability Registry

## 1. Definition

The **tool registry** is the catalog of all available tools — their names, descriptions, schemas, and access controls.

The **capability registry** is the catalog of all available capabilities — what they do, what tools they use, who can invoke them, and what permissions are required.

Together, they are the *menu* from which agents and orchestrators choose when deciding how to accomplish a task.

---

## 2. Why They Exist

Without a registry:
- Each agent hardcodes which tools it uses → can't discover new tools
- Adding a tool requires updating every agent that might use it
- No centralized view of what the platform can do
- No access control on tool/capability usage

With registries:
- Agents query the registry to discover available tools at runtime
- New tools are registered once → all agents can discover them
- Access control is enforced at registry level
- Operations team can see the full capability surface of the platform

---

## 3. Observability Example

```
TOOL REGISTRY ENTRY:

{
  "tool_id": "loki_log_query",
  "name": "Query Loki Logs",
  "description": "Query application logs from Loki using LogQL",
  "schema": {
    "inputs": {
      "service": "string (required)",
      "market_id": "string (optional)",
      "severity": "enum[DEBUG|INFO|WARN|ERROR|FATAL]",
      "start_time": "ISO8601 datetime",
      "end_time": "ISO8601 datetime",
      "limit": "integer (default: 100, max: 1000)"
    },
    "output": "array of log line objects"
  },
  "endpoint": "http://loki-mcp-server:8080/query",
  "auth_required": true,
  "rate_limit": "100 calls/minute",
  "timeout_seconds": 30,
  "permissions_required": ["observability:read"],
  "owner_team": "observability-platform",
  "tags": ["logs", "observability", "read-only"]
}
```

---

## 4. Visual Workflow

### Registry-Driven Tool Discovery

```
Agent starts investigation
          │
          ▼
┌─────────────────────────────────────────────────────┐
│              TOOL REGISTRY QUERY                    │
│  "List all tools tagged 'observability' AND 'read'" │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│              RETURNED TOOLS                         │
│  - loki_log_query                                   │
│  - prometheus_metric_query                          │
│  - grafana_dashboard_get                            │
│  - tempo_trace_search                               │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
Agent selects relevant tools → Investigation begins
```

### Registry as Governance Layer

```
┌──────────────────────────────────────────────────────────┐
│              TOOL REGISTRY GOVERNANCE                    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  ACCESS CONTROL                                 │    │
│  │  - observability:read  → Loki, Prometheus, Tempo│    │
│  │  - data:read           → MongoDB, Snowflake      │    │
│  │  - action:write        → PagerDuty, Slack, JIRA  │    │
│  │  - kafka:read          → Kafka consumer state    │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  RATE LIMITING per tool                         │    │
│  │  Prevents runaway agents from DDoSing systems   │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  AUDIT LOG                                      │    │
│  │  Every tool call logged: who, what, when, result│    │
│  └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## 5. Summary

**Key Ideas:**
- Registries decouple tool/capability definition from agent implementation
- Dynamic discovery enables agents to use newly registered tools without code changes
- Access control and audit logging belong at the registry layer, not the agent layer
- The registry is a governance layer, not just a lookup table

**Mental Model:**
> The tool registry is the app store. Developers (your team) publish tools. Agents (users) discover and install them. The store enforces permissions and tracks usage. No one hardcodes which apps are installed.

---

# Chapter 19: State

## 1. Definition

**State** is everything the agent needs to remember about the *current* conversation or investigation — what has been tried, what has been found, what decisions have been made, and where in the investigation process the agent currently is.

State is temporary — it exists for the duration of one investigation session.

---

## 2. Why It Exists

An LLM has no memory between API calls. Every time you call the LLM, it starts fresh — unless you explicitly pass the previous context.

State management is the mechanism that maintains continuity across multiple LLM calls within a single investigation.

Without state:
- Step 2 doesn't know what Step 1 found
- The planner can't avoid re-querying the same data
- Tool results are lost between turns

With state:
- The full investigation history is preserved and passed to each LLM call
- The planner knows exactly what has been done and what hasn't
- Results from earlier steps inform later steps

---

## 3. Observability Example

### RCA Investigation State Object

```json
{
  "session_id": "RCA-2024-0115-ABC-001",
  "created_at": "2024-01-15T10:45:00Z",
  "status": "IN_PROGRESS",
  
  "query": {
    "original": "Why has MarketID ABC stopped processing trades since 10:30?",
    "intent": "OUTAGE_INVESTIGATION",
    "entities": {
      "market_id": "ABC",
      "time_reference": "10:30",
      "time_utc": "2024-01-15T05:00:00Z"
    }
  },
  
  "plan": {
    "steps_total": 4,
    "steps_completed": 2,
    "current_step": 3
  },
  
  "tool_calls": [
    {
      "step": 1,
      "tool": "loki_log_query",
      "params": {"service": "trade-proc", "market_id": "ABC", "severity": "ERROR"},
      "result_summary": "147 errors. FATAL connection refused to kafka-broker-1 at 10:29:58",
      "result_token_count": 4200
    },
    {
      "step": 2,
      "tool": "kafka_consumer_state",
      "params": {"group": "abc-trade-processor-grp"},
      "result_summary": "Lag=45000, status=LAGGING since 10:29:45",
      "result_token_count": 800
    }
  ],
  
  "current_hypothesis": "kafka-broker-1 failure caused trade processing stoppage",
  "confidence": 0.82,
  "evidence_count": 2
}
```

---

## 4. Visual Workflow

### State in the Agent Loop

```
┌─────────────────────────────────────────────────────────────┐
│                    AGENT EXECUTION LOOP                     │
│                                                             │
│  Turn 1:                                                    │
│  State: { step: 1, tool_calls: [] }                         │
│  LLM input: [system_prompt + user_query + state]            │
│  LLM output: call loki_log_query(...)                       │
│  State update: { step: 1, tool_calls: [loki_result] }       │
│                                                             │
│  Turn 2:                                                    │
│  State: { step: 1, tool_calls: [loki_result] }              │
│  LLM input: [system_prompt + user_query + state + turn1]    │
│  LLM output: call kafka_consumer_state(...)                 │
│  State update: { step: 2, tool_calls: [loki, kafka] }       │
│                                                             │
│  Turn 3:                                                    │
│  State: { step: 2, tool_calls: [loki, kafka] }              │
│  LLM input: [system_prompt + user_query + state + turns1-2] │
│  LLM output: Final RCA JSON                                 │
│  State update: { status: COMPLETE, result: {...} }          │
└─────────────────────────────────────────────────────────────┘
```

### State Components

```
                    AGENT STATE
                         │
     ┌───────────────────┼─────────────────────┐
     │                   │                     │
     ▼                   ▼                     ▼
 SESSION INFO      INVESTIGATION         WORKING MEMORY
 ─────────────     ─────────────         ──────────────
 session_id        intent                current_hypothesis
 user_id           entities              confidence_level
 created_at        plan                  evidence_gathered
 status            steps_done            key_findings
```

---

## 5. Enterprise Design

| State Storage | When to Use | Trade-off |
|--------------|-------------|-----------|
| **In-memory (dict)** | Single-turn, stateless services | Fast; lost on crash |
| **Redis** | Multi-turn, high-throughput | Fast; TTL management needed |
| **MongoDB** | Full persistence, audit trail | Slower; full history |
| **LangGraph StateGraph** | Complex state machines | Built-in; framework dependency |

For a production RCA platform: Redis for active session state + MongoDB for completed investigation records.

---

## 6. Summary

**Key Ideas:**
- State is the agent's working memory for the current investigation
- Without explicit state management, each LLM call starts blind
- State grows with each turn — design for context budget
- Persist completed investigation states for audit and ML training data

**Mental Model:**
> State is the detective's case notes. They write down every clue, every lead, every hypothesis. When they come back to the case tomorrow, they read the notes — not start over. Your agent's state is those case notes.

---

# Chapter 20: Memory

## 1. Definition

**Memory** is the agent's ability to retain information *across sessions* — not just within one investigation, but across many investigations over time.

State (Chapter 19) = what happened in *this* conversation.
Memory = what the agent has learned from *all previous* conversations.

Types of memory:
- **Episodic memory**: "The last time MarketID ABC had this error, it was a Kafka broker failover."
- **Semantic memory**: "MarketID ABC is a high-criticality equities market owned by the equities-ops team."
- **Procedural memory**: "For Kafka connection errors, always check broker health before consumer group state."

---

## 2. Why It Exists

Without memory, every investigation starts from zero. The agent doesn't know:
- That MarketID ABC had the same issue last Tuesday
- That the equities-ops team prefers Slack alerts over PagerDuty
- That kafka-broker-1 is aging hardware that fails intermittently

Memory makes the agent progressively smarter about *your specific environment*.

---

## 3. Observability Example

### Memory Types for the RCA Platform

```
┌──────────────────────────────────────────────────────────────┐
│                  MEMORY TYPES IN USE                         │
│                                                              │
│  EPISODIC MEMORY (past incidents)                            │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  "MarketID ABC had 3 outages in the last 30 days.     │  │
│  │   All three were caused by kafka-broker-1 instability. │  │
│  │   Root cause was always confirmed within 3 tool calls."│  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  SEMANTIC MEMORY (domain knowledge)                          │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  "kafka-broker-1 is due for replacement (aging HW).   │  │
│  │   During NSE trading hours (9:15–15:30 IST), any      │  │
│  │   ABC outage automatically triggers P1 incident."      │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  PROCEDURAL MEMORY (investigation playbooks)                 │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  "For ABC trade processing outages: check Kafka        │  │
│  │   broker health BEFORE consumer group lag —            │  │
│  │   this reduces investigation time by 40%."             │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Visual Workflow

### Memory Architecture

```
                        INVESTIGATION COMPLETE
                                │
                                ▼
                    ┌───────────────────────┐
                    │   MEMORY WRITER       │
                    │   Extracts learnings  │
                    │   from this session   │
                    └───────────┬───────────┘
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
           ▼                    ▼                    ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │  EPISODIC    │    │  SEMANTIC    │    │  PROCEDURAL  │
   │  STORE       │    │  STORE       │    │  STORE       │
   │  (MongoDB:   │    │  (MongoDB:   │    │  (Vector DB: │
   │  past RCAs)  │    │  domain KB)  │    │  playbooks)  │
   └──────────────┘    └──────────────┘    └──────────────┘
           │                    │                    │
           └────────────────────┼────────────────────┘
                                │
                                ▼
                    NEXT INVESTIGATION SESSION
                    ┌───────────────────────┐
                    │  MEMORY RETRIEVER     │
                    │  Fetches relevant     │
                    │  memories for context │
                    └───────────┬───────────┘
                                │
                                ▼
                    Injected into LLM context
                    alongside current query
```

### Memory vs. State vs. RAG

```
┌──────────────────────────────────────────────────────────────┐
│                    CONCEPT COMPARISON                        │
├─────────────────┬────────────────────────────────────────────┤
│ STATE           │ Current session only. Gone when done.      │
│                 │ "What have I done so far TODAY?"           │
├─────────────────┼────────────────────────────────────────────┤
│ MEMORY          │ Across sessions. Persists over time.        │
│                 │ "What have I learned from past cases?"     │
├─────────────────┼────────────────────────────────────────────┤
│ RAG             │ External knowledge base. Static documents. │
│                 │ "What do runbooks and docs say?"           │
│                 │ (See Chapter 21)                           │
└─────────────────┴────────────────────────────────────────────┘
```

---

## 5. Enterprise Design

| Memory Pattern | Implementation | Trade-off |
|----------------|----------------|-----------|
| **Verbatim episode storage** | Store full RCA JSON | High storage; full fidelity |
| **Summarized episodes** | Compress past RCAs | Lower storage; some detail lost |
| **Vector-indexed episodes** | Embed + store for semantic search | Fast retrieval; needs vector DB |
| **Structured knowledge** | Extract facts into knowledge graph | Precise; curation effort |

For your platform: Store RCA JSON in MongoDB + embed summaries in a vector DB for semantic retrieval.

---

## 6. Common Mistakes

| Mistake | Consequence |
|---------|-------------|
| No memory → same mistakes repeated | Agent re-investigates known issues from scratch every time |
| Injecting all memory → context overflow | Memory consumes the entire context window |
| No memory quality control | Wrong memories (from failed investigations) pollute future reasoning |
| Memory without privacy controls | Sensitive investigation details leaked across user sessions |

---

## 7. Summary

**Key Ideas:**
- Memory is the agent's learning from past investigations
- Three types: episodic (past events), semantic (domain knowledge), procedural (best practices)
- Memory must be retrieved selectively — injecting all memory defeats the purpose
- Memory quality degrades over time; curation and expiry are necessary

**Mental Model:**
> Memory is the experience that separates a junior engineer from a senior one. The junior investigates every Kafka issue from first principles. The senior remembers: "kafka-broker-1 failed twice last month during peak load — check there first." Your agent's memory is that accumulated experience.

**Practical Exercise:**
Define what your agent should remember after each completed RCA:
1. What was the root cause?
2. How long did investigation take?
3. Which tool call was decisive?
4. Was the root cause a known pattern or novel?
5. What was the remediation?

This is your memory schema.

---

*[End of Part 2 — Chapters 11–20]*
