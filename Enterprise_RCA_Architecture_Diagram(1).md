# Enterprise AI Observability & RCA Architecture

## High-Level Architecture

``` text
USER
  │
  ▼
Semantic Planner (Claude SDK)
  │
  ▼
Structured Planning Output
  │
  ▼
Capability / Domain Resolver
  │
  ▼
Selected MCPs
  │
  ▼
Investigation Agent (Claude SDK)
  │
  ▼
MCP Layer
  │
  ▼
Enterprise Data Sources
```

------------------------------------------------------------------------

# Semantic Planner (formerly Intent Extraction)

The planner is **not** an investigation agent.

Its purpose is to understand the request and produce a structured plan.

It should answer:

1.  What is the user trying to accomplish?
2.  What entities are involved?
3.  What contextual information is relevant?
4.  What information is required to answer the request?
5.  Which investigation methodology (or playbook) should be used?

The planner **must not**:

-   Query Prometheus
-   Search Loki
-   Execute SQL
-   Retrieve logs
-   Perform RCA
-   Correlate evidence

Its job ends before investigation begins.

## Information to Extract

### Intent

Examples:

-   root_cause_analysis
-   explain
-   topology_explanation
-   compare
-   health_check
-   summarize
-   capacity_analysis

### Entities

Examples:

-   Service
-   Kafka Topic
-   Consumer Group
-   Broker
-   MongoDB Database
-   Snowflake Warehouse
-   Kubernetes Pod
-   Namespace
-   Node

### Context

Examples:

-   Time range
-   Environment (Prod/UAT)
-   Deployment reference
-   Comparison request
-   Severity
-   Error code

### Domain vs Capability (Open Design Decision)

Option A

Planner returns Domain:

-   Application
-   Messaging
-   Database
-   Infrastructure

Resolver derives capabilities.

Option B

Planner returns Capabilities directly:

-   Metrics
-   Logs
-   Traces
-   Topology
-   Deployments
-   Kafka
-   Snowflake
-   MongoDB

Resolver maps capabilities to MCPs.

This remains an open architectural decision.

### Investigation Playbook

Another design question:

Option A: Planner selects the investigation playbook.

Option B: A downstream policy/router selects the playbook from planner
output.

## Example Structured Output

``` json
{
  "intent": "root_cause_analysis",
  "entities": [
    {
      "type": "service",
      "name": "orders"
    }
  ],
  "context": {
    "deployment": "after latest deployment",
    "environment": "production"
  },
  "capabilities": [
    "metrics",
    "logs",
    "deployments"
  ]
}
```

------------------------------------------------------------------------

# Responsibilities

  Component                    Responsibility
  ---------------------------- ------------------------------------------------
  Semantic Planner             Understand request and produce structured plan
  Capability/Domain Resolver   Map planner output to integrations
  Investigation Agent          Perform reasoning and investigation
  MCP                          Retrieve data / execute actions
  Enterprise Systems           Source of truth

------------------------------------------------------------------------

# Notes

-   MCPs expose generic capabilities only.
-   Dynamic tool exposure is preferred over exposing every MCP.
-   Claude SDK may be sufficient initially.
-   LangGraph should be introduced only if future needs include
    checkpoints, long-running workflows, multiple agents, approvals, or
    persistent execution state.
