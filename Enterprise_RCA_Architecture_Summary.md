# Enterprise AI Observability & RCA Architecture Notes

## Purpose

This document summarizes the major architectural discussions, design
decisions, alternatives, and open questions. It is intended as a concise
knowledge base for review by another LLM or architect.

------------------------------------------------------------------------

# Goals

-   Enterprise-grade RCA agent
-   Modular and maintainable
-   Efficient tool usage
-   Reusable integrations
-   Strong reasoning with minimal orchestration complexity

Technology stack includes Prometheus, Loki, Tempo, Kafka, Snowflake,
MongoDB, JVM diagnostics, Kubernetes, Grafana, and internal APIs.

------------------------------------------------------------------------

# Core Design Principles

## 1. Separate responsibilities

**LLM** - Planning - Reasoning - Correlation - Explanation - RCA
generation

**MCP** - Retrieve data - Execute actions - Never perform reasoning

**Skills / Playbooks** - Investigation methodology - Evidence
requirements - Stopping criteria - Output expectations

------------------------------------------------------------------------

## 2. MCP Philosophy

MCPs should expose generic operations.

Good examples: - query_metrics() - search_logs() - execute_sql() -
get_service_topology() - consumer_lag()

Avoid: - explain_topology() - diagnose_latency() - find_root_cause()

------------------------------------------------------------------------

# Planner

Purpose: - Understand the request - Extract intent - Extract entities -
Determine required information

Planner should NOT investigate incidents.

Possible outputs:

``` json
{
  "intent": "...",
  "entities": [],
  "capabilities": []
}
```

Structured outputs or tool calling are preferred over free-form JSON.

------------------------------------------------------------------------

# Intent vs Investigation

Intent extraction answers:

"What does the user want?"

Example: - RCA - Explain topology - Health check - Compare

Investigation answers:

"How should this problem be solved?"

These are separate responsibilities.

------------------------------------------------------------------------

# Domain vs Capability

This remains an open architectural decision.

## Option A

Planner outputs domains.

Application → Metrics → Logs → Deployments

Messaging → Kafka → Broker Health

## Option B

Planner outputs capabilities directly.

-   Metrics
-   Logs
-   Topology
-   Deployments
-   Kafka
-   Snowflake

Capability-first reduces coupling to specific technologies but
introduces another abstraction.

This decision should be validated.

------------------------------------------------------------------------

# Capability Resolver

Maps planner output to implementations.

Example:

Metrics → Prometheus MCP

Logs → Loki MCP

This allows backend implementations to change without modifying planning
logic.

------------------------------------------------------------------------

# Dynamic Tool Exposure

Do not expose every MCP.

Instead:

Planner → Required capabilities → Resolver → Only relevant MCPs

Example:

Orders latency

Expose: - Prometheus - Loki - Deployment

Hide: - Kafka - MongoDB - Snowflake

Benefits: - Fewer unnecessary tool calls - Smaller context - Better
reasoning

------------------------------------------------------------------------

# Skills

Two viewpoints were discussed.

## Option A

Do not build a Skills framework initially.

Keep methodology in: - System prompt - Tool descriptions - Few-shot
examples

Extract skills only after repeated failures.

## Option B

Treat skills as reusable operational playbooks from day one.

Examples: - Kafka RCA - JVM Memory Investigation - Latency
Investigation - Topology Explanation

Question: When does a skill become valuable enough to justify a
dedicated abstraction?

------------------------------------------------------------------------

# Claude SDK

Claude SDK already provides: - Planning - Tool selection - Multi-step
reasoning - Iterative investigations - Stopping decisions

Question: Is additional orchestration required?

------------------------------------------------------------------------

# LangGraph

Current view:

Likely unnecessary initially.

Potential justification: - Long-running workflows - Pause/resume - Human
approvals - Multiple agents - Parallel investigations - Persistent state

Question: Should LangGraph be introduced only when these needs appear?

------------------------------------------------------------------------

# Investigation Loop

Think

↓

Call MCP

↓

Analyze evidence

↓

Need more evidence?

↓

Repeat

↓

Generate RCA

------------------------------------------------------------------------

# Architecture (High Level)

User

↓

Planner

↓

Capability / Domain Resolution

↓

Selected MCPs

↓

Investigation Agent

↓

Enterprise Data Sources

↓

RCA

------------------------------------------------------------------------

# Open Questions

1.  Is a planner layer necessary?
2.  Should planning produce domains or capabilities?
3.  Is a capability resolver useful or unnecessary complexity?
4.  Should skills exist as first-class artifacts?
5.  When does LangGraph become worthwhile?
6.  Is Claude SDK alone sufficient?
7.  What enterprise patterns would simplify this architecture?

------------------------------------------------------------------------

# Guiding Principle

Prefer introducing abstractions only after there is repeated evidence
they solve a real problem. Avoid premature frameworks, but also
recognize when existing operational runbooks justify a reusable
investigation methodology.
