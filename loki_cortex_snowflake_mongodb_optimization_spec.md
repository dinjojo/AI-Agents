# Observability & Data Correlation Optimization Specification
**Target Stack:** Python 3.11+ (`asyncio.TaskGroup`, `httpx`, `pymongo`/`motor`, `snowflake-connector-python`)  
**Capabilities:** Grafana Loki (Logs), Cortex (Prometheus Metrics), Snowflake (Analytical/Audit Data), MongoDB (Operational State)

---

## 1. Objective & Two-Phase Architecture

**Goal:** Cut observability and database token consumption by **70%–90%** before LLM prompt injection while preserving 100% of root-cause diagnostic signals and preventing warehouse, cluster, or event-loop bottlenecks.

### Cascading Two-Phase Execution Model
1. **Phase 1 — Anchor Discovery (Loki):**
   - Push down noise filtering and plain-text formatting into LogQL.
   - Sort per stream, stitch multi-line stack traces, redact secrets, extract error-anchored windows, deduplicate retry storms, and emit a structured `CorrelationContext` (`namespaces`, `services`, `pods`, `trace_ids`, `order_ids`, `user_ids`, and exact `incident_window`).
2. **Phase 2 — Fault-Isolated Parallel Enrichment (`asyncio.TaskGroup`):**
   - Fan out concurrently across **Cortex**, **Snowflake**, and **MongoDB** using `CorrelationContext`.
   - Each backend summarizes its response into a bounded, plain-text `str` block inside an isolated error boundary so a timeout in one store never cancels sibling tasks.

---

## 2. Universal Rules Across All 4 Capabilities

1. **Plain-Text Wire Format Only:** Never pass JSON log envelopes, raw Cortex time-series matrices, or `DictCursor` JSON arrays to the LLM. Repeating keys and braces adds **30%–50% token overhead**.
2. **Every LLM Tool MUST Return `str`:** Agent frameworks automatically run `json.dumps()` on `dict` or `list` return values, silently re-wrapping clean output into escaped JSON strings.
3. **Batch Lookups (Zero $N+1$ Queries):** Collapse extracted IDs and pods into a single batched query per store (`pod=~"a|b"` in Cortex, `$in` in MongoDB, parameterized `IN (...)` in Snowflake) capped at $K \le 10$ entities.
4. **Isolate `asyncio.TaskGroup` Branches:** Wrap every backend coroutine in `asyncio.wait_for` + `try/except` returning a `<store_error>` string on failure so one slow store cannot abort healthy siblings.
5. **Maintain Persistent Client Pools:** Initialize `httpx.AsyncClient`, MongoDB async clients, and Snowflake connection pools once at app startup—never per chat turn.
6. **Fence Untrusted Data:** Wrap all external payloads in XML delimiter tags (`<loki_logs>`, `<cortex_metrics>`, `<snowflake_rows>`, `<mongodb_docs>`) and instruct the LLM in its system prompt to treat enclosed text strictly as untrusted diagnostic data.

---

## 3. Production Blind Spots & Mitigations

| Blind Spot | System | Failure Mode | Mandatory Mitigation |
| :--- | :--- | :--- | :--- |
| **Reverse Ordering & Stream Interleaving** | Loki | Loki defaults to `direction=backward` and groups by label stream; windowing unsorted results mixes context across pods. | Pass `direction=forward` and sort ascending **per `(namespace, service, pod)` stream** before windowing. |
| **Severed Multi-Line Tracebacks** | Loki | Unmerged stack traces arrive as 30+ entries; `window_after=2` drops `Caused by:` lines before folding. | Stitch continuation lines per stream **before** applying context windows or stack folding. |
| **Silent `limit` Truncation** | Loki, Snowflake | Hitting `limit` silently drops rows, misleading the LLM into assuming clean baselines. | Fetch `limit + 1` in Snowflake and check `len >= limit` in Loki; prepend explicit `[TRUNCATED]` warnings. |
| **High-Cardinality Join Mismatch** | Loki $\leftrightarrow$ Cortex | Cortex metrics do not store `trace_id` or `order_id`; querying them returns empty results. | Bridge Loki to Cortex strictly via low-cardinality labels: `(namespace, service, pod)`. |
| **Hardcoded Range `step` & Bucket Scans** | Cortex | Fixed `step=60s` over 24h returns 1,441 points; raw `histogram_quantile` scans all buckets. | Use `/api/v1/query` with `*_over_time()` when possible; compute adaptive `step`; use Ruler recording rules. |
| **Single-Connection Thread Contention** | Snowflake | Running concurrent `asyncio.to_thread()` queries on one Snowflake connection corrupts buffers. | Check out one connection per query from an `asyncio.Queue` pool ($3\text{–}5$ warm connections). |
| **Function-Wrapped Timestamps & Cache Busting** | Snowflake | `DATE_TRUNC('hour', ts)` disables partition pruning; `CURRENT_TIMESTAMP()` disables Result Cache. | Filter with raw bounds `ts >= %(start)s AND ts < %(end)s` and round timestamps to 30s buckets. |
| **Unindexed `COLLSCAN` & Secret Leaks** | MongoDB, Loki | Querying unindexed fields stalls MongoDB; unredacted tokens break log deduplication. | Enforce an indexed-field allowlist + `maxTimeMS(3000)` in MongoDB; redact secrets **before** templating. |

---

## 4. Loki Pipeline (`loki_pipeline.py`)

### 4.1 LogQL Pushdown Rules
- **Filter before parsing:** Place substring filters (`|=`, `!=`) *before* `| json` or `| pattern` so Loki skips decompressing irrelevant lines.
- **Strip JSON in Loki:** Use `| line_format` for JSON logs and `| pattern` for plain-text logs.
- **Wide-window metric pre-check:** For windows $> 30\text{m}$, run `sum by (service) (count_over_time({namespace="prod"} |= "ERROR" [5m]))` first to isolate the spike window before pulling raw logs.

```python
from __future__ import annotations

import hashlib
import re
import time
from collections import OrderedDict
from dataclasses import dataclass, field
from datetime import datetime, timezone
import httpx

_SECRET_PATTERNS: list[tuple[re.Pattern[str], str]] = [
    (re.compile(r"(?i)\b(authorization\s*[:=]\s*bearer\s+)[a-z0-9\-._~+/]+=*"), r"\1[REDACTED]"),
    (re.compile(r"\beyJ[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}\b"), "[REDACTED_JWT]"),
    (re.compile(r"(?i)\b(api[_-]?key|secret|password|passwd|token)\s*[:=]\s*['\"]?[^\s'\",;}{]+"), r"\1=[REDACTED]"),
]

_TEMPLATE_MASK_PATTERNS: list[tuple[re.Pattern[str], str]] = [
    (re.compile(r"\b[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}\b", re.I), "<UUID>"),
    (re.compile(r"\b\d{1,3}(?:\.\d{1,3}){3}(?::\d+)?\b"), "<IP>"),
    (re.compile(r"\b\d+(?:\.\d+)?(?:ns|us|ms|s|m|B|KB|MB|GB)\b"), "<DUR_OR_SIZE>"),
    (re.compile(r"\b0x[0-9a-f]+\b", re.I), "<HEX>"),
    (re.compile(r"\b\d{4,}\b"), "<NUM>"),
]

_TRACE_ID_RE = re.compile(r"(?:trace[_-]?id|traceId)[=:\s\"']+([a-f0-9]{16,32})", re.I)
_ORDER_ID_RE = re.compile(r"(?:order[_-]?id|orderId)[=:\s\"']+([a-zA-Z0-9_-]{6,36})", re.I)
_USER_ID_RE = re.compile(r"(?:user[_-]?id|userId)[=:\s\"']+([a-zA-Z0-9_-]{4,36})", re.I)
_NEW_ENTRY_RE = re.compile(
    r"^(?:\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}|\d{2}:\d{2}:\d{2}|\[?\b(?:TRACE|DEBUG|INFO|WARN|WARNING|ERROR|FATAL)\b\]?)"
)


@dataclass
class CorrelationContext:
    start_ns: int
    end_ns: int
    namespaces: set[str] = field(default_factory=set)
    services: set[str] = field(default_factory=set)
    pods: set[str] = field(default_factory=set)
    trace_ids: list[str] = field(default_factory=list)
    order_ids: list[str] = field(default_factory=list)
    user_ids: list[str] = field(default_factory=list)
    first_error_ns: int | None = None
    last_error_ns: int | None = None

    def add_unique(self, target: list[str], value: str, cap: int = 10) -> None:
        if value and value not in target and len(target) < cap:
            target.append(value)


@dataclass
class _DedupBucket:
    template: str
    first_ts: str
    last_ts: str
    first_raw: str
    severity: int
    count: int = 1


def redact_secrets(text: str) -> str:
    for pat, repl in _SECRET_PATTERNS:
        text = pat.sub(repl, text)
    return text


def _severity_rank(line: str) -> int:
    if any(t in line for t in ("FATAL", "ERROR", "Exception", "Traceback", "panic:")):
        return 3
    if "WARN" in line:
        return 2
    return 1


def stitch_multiline_entries(entries: list[tuple[int, str]]) -> list[tuple[int, str]]:
    if not entries:
        return []
    stitched: list[tuple[int, str]] = []
    curr_ts, curr_lines = entries[0][0], [entries[0][1]]
    for ts_ns, line in entries[1:]:
        is_cont = line.startswith((" ", "\t", "Caused by:", "Traceback (most recent call last):"))
        if is_cont or not _NEW_ENTRY_RE.match(line.lstrip()):
            curr_lines.append(line)
        else:
            stitched.append((curr_ts, "\n".join(curr_lines)))
            curr_ts, curr_lines = ts_ns, [line]
    stitched.append((curr_ts, "\n".join(curr_lines)))
    return stitched


def fold_stacktrace(text: str, keep_top: int = 4) -> str:
    lines = text.splitlines()
    if len(lines) <= keep_top + 5:
        return text
    caused_by_idx = next((i for i, l in enumerate(lines) if l.lstrip().startswith("Caused by:")), None)
    head = lines[:keep_top]
    tail = lines[caused_by_idx:] if caused_by_idx is not None else lines[-3:]
    omitted = len(lines) - len(head) - len(tail)
    if omitted <= 1:
        return text
    return "\n".join([*head, f"  ... [{omitted} intermediate frames omitted] ...", *tail])


def _truncate_huge_line(text: str, max_line_chars: int = 900) -> str:
    if len(text) <= max_line_chars:
        return text
    return f"{text[:600]} ... [truncated {len(text) - 800} chars] ... {text[-200:]}"


def _to_template(msg: str) -> str:
    for pattern, token in _TEMPLATE_MASK_PATTERNS:
        msg = pattern.sub(token, msg)
    return msg


def _ns_to_iso(ns: int) -> str:
    return datetime.fromtimestamp(ns / 1_000_000_000, tz=timezone.utc).strftime("%H:%M:%S.%f")[:-3] + "Z"


_LOKI_CACHE: dict[str, tuple[float, str, CorrelationContext]] = {}
_CACHE_TTL_SEC = 20.0


async def fetch_and_compress_loki(
    client: httpx.AsyncClient,
    loki_base_url: str,
    logql: str,
    start_ns: int,
    end_ns: int,
    allowed_namespaces: set[str],
    tenant_id: str | None = None,
    limit: int = 500,
    max_chars: int = 12_000,
) -> tuple[str, CorrelationContext]:
    if not any(f'namespace="{ns}"' in logql for ns in allowed_namespaces):
        raise ValueError(f"LogQL must filter an allowed namespace: {sorted(allowed_namespaces)}")

    cache_key = hashlib.sha256(f"{tenant_id}|{logql}|{start_ns}|{end_ns}|{limit}".encode()).hexdigest()
    now = time.monotonic()
    if cache_key in _LOKI_CACHE and (now - _LOKI_CACHE[cache_key][0]) <= _CACHE_TTL_SEC:
        return _LOKI_CACHE[cache_key][1], _LOKI_CACHE[cache_key][2]

    headers = {"X-Scope-OrgID": tenant_id} if tenant_id else {}
    resp = await client.get(
        f"{loki_base_url.rstrip('/')}/loki/api/v1/query_range",
        params={"query": logql, "start": str(start_ns), "end": str(end_ns), "limit": limit, "direction": "forward"},
        headers=headers,
        timeout=8.0,
    )
    resp.raise_for_status()
    streams = resp.json().get("data", {}).get("result", [])

    ctx = CorrelationContext(start_ns=start_ns, end_ns=end_ns)
    if not streams:
        return "<loki_logs>\nNo logs matched the query window.\n</loki_logs>", ctx

    total_raw_lines = 0
    sections: list[str] = []

    for stream in streams:
        labels: dict[str, str] = stream.get("stream", {})
        if ns := labels.get("namespace"):
            ctx.namespaces.add(ns)
        if svc := (labels.get("service") or labels.get("app")):
            ctx.services.add(svc)
        if pod := labels.get("pod"):
            ctx.pods.add(pod)

        raw_values = sorted((int(ts), redact_secrets(line)) for ts, line in stream.get("values", []))
        total_raw_lines += len(raw_values)
        stitched = stitch_multiline_entries(raw_values)

        severities = [_severity_rank(line) for _, line in stitched]
        keep_indices: set[int] = set()
        if len(stitched) <= 40:
            keep_indices = set(range(len(stitched)))
        else:
            for idx, sev in enumerate(severities):
                if sev >= 2:
                    keep_indices.update(range(max(0, idx - 5), min(len(stitched), idx + 3)))
        if not keep_indices:
            keep_indices.update(range(min(10, len(stitched))))
            keep_indices.update(range(max(0, len(stitched) - 15), len(stitched)))

        buckets: OrderedDict[str, _DedupBucket] = OrderedDict()
        for idx in sorted(keep_indices):
            ts_ns, raw_line = stitched[idx]
            sev = severities[idx]
            if sev >= 2:
                ctx.first_error_ns = min(ctx.first_error_ns, ts_ns) if ctx.first_error_ns else ts_ns
                ctx.last_error_ns = max(ctx.last_error_ns, ts_ns) if ctx.last_error_ns else ts_ns

            for m in _TRACE_ID_RE.findall(raw_line):
                ctx.add_unique(ctx.trace_ids, m)
            for m in _ORDER_ID_RE.findall(raw_line):
                ctx.add_unique(ctx.order_ids, m)
            for m in _USER_ID_RE.findall(raw_line):
                ctx.add_unique(ctx.user_ids, m)

            cleaned = _truncate_huge_line(fold_stacktrace(raw_line))
            tmpl = _to_template(cleaned)
            iso_ts = _ns_to_iso(ts_ns)

            if tmpl in buckets:
                buckets[tmpl].count += 1
                buckets[tmpl].last_ts = iso_ts
            else:
                buckets[tmpl] = _DedupBucket(tmpl, iso_ts, iso_ts, cleaned, sev)

        rendered_items = list(buckets.values())
        if len(rendered_items) > 35:
            rendered_items.sort(key=lambda b: (-b.severity, -b.count))

        header = " | ".join(f"{k}={v}" for k, v in sorted(labels.items()))
        lines_out = [f"=== Stream: {header} ==="]
        for b in rendered_items:
            prefix = f"[{b.first_ts}]" if b.count == 1 else f"[{b.first_ts} .. {b.last_ts}] (x{b.count})"
            lines_out.append(f"{prefix} {b.first_raw}")
        sections.append("\n".join(lines_out))

    preamble: list[str] = ["<loki_logs>"]
    if total_raw_lines >= limit:
        preamble.append(f"[WARNING: Hit query limit ({limit} lines). Earlier events may be truncated; narrow window.]")
    preamble.append(
        "CORRELATION_IDS: "
        f"services={sorted(ctx.services)} | pods={sorted(ctx.pods)[:10]} | "
        f"trace_ids={ctx.trace_ids} | order_ids={ctx.order_ids} | user_ids={ctx.user_ids}"
    )

    body = "\n\n".join(sections)
    budget = max(1000, max_chars - sum(len(p) for p in preamble) - 64)
    if len(body) > budget:
        body = body[:budget] + "\n... [Output truncated at character budget; filter by trace_id or pod]"

    final_output = "\n".join([*preamble, body, "</loki_logs>"])
    _LOKI_CACHE[cache_key] = (now, final_output, ctx)
    return final_output, ctx
```

---

## 5. Cortex Metrics Optimization & Correlation (`cortex_correlator.py`)

### 5.1 PromQL & Cluster Rules
- **Instant vs. Range Queries:** Default to `/api/v1/query` with `max_over_time()`, `avg_over_time()`, or `rate()` for threshold checks. Use `/api/v1/query_range` only when detecting anomaly onset timing.
- **Adaptive `step` Sizing:** Compute `step = max(scrape_interval_sec, window_seconds // 25)` to cap evaluated series at ~25 points regardless of window duration.
- **Cardinality & Index Pruning:** Use exact equality matchers (`=`) where possible, wrap pod-level metrics in `topk(5, ...)`, consolidate dimensions via `sum by (...)`, and keep `rate()` windows $\ge 4\times$ the scrape interval (e.g., `[2m]` for 15s scrapes).
- **Cortex Infrastructure:** Route queries through `query-frontend` (with `align_queries_with_step: true` and result caching), pre-compute expensive `histogram_quantile` buckets via **Cortex Ruler recording rules**, and drop high-cardinality labels (`request_id`, `pod_uid`) before ingestion.

### 5.2 Python Statistical Digest with Spike-Onset Detection

```python
from __future__ import annotations

import asyncio
import re
import statistics
from datetime import datetime, timezone
import httpx
from loki_pipeline import CorrelationContext


def summarize_cortex_matrix(series_list: list[dict], metric_label: str) -> list[str]:
    """Convert Cortex range matrix JSON into compact plain text with baseline and spike onset."""
    lines: list[str] = []
    for item in series_list:
        labels = ",".join(
            f"{k}={v}" for k, v in sorted(item.get("metric", {}).items()) if k != "__name__"
        ) or "aggregated"
        points = [(float(ts), float(val)) for ts, val in item.get("values", []) if val not in ("NaN", "+Inf", "-Inf")]
        if not points:
            continue

        values = [v for _, v in points]
        baseline = statistics.median(values[: max(1, len(values) // 3)])
        peak_ts, peak_val = max(points, key=lambda p: p[1])
        latest_val = points[-1][1]

        onset_str = "none"
        if baseline > 0:
            for ts, val in points:
                if val >= baseline * 1.5:
                    onset_str = datetime.fromtimestamp(ts, tz=timezone.utc).strftime("%H:%M:%SZ")
                    break
        peak_time = datetime.fromtimestamp(peak_ts, tz=timezone.utc).strftime("%H:%M:%SZ")
        delta_pct = ((peak_val - baseline) / baseline * 100) if baseline > 0 else 0.0

        lines.append(
            f"{metric_label}[{labels}] -> baseline={baseline:.3g} | peak={peak_val:.3g} at {peak_time} "
            f"({delta_pct:+.0f}%) | latest={latest_val:.3g} | spike_onset={onset_str}"
        )
    return lines


async def correlate_cortex_metrics(
    client: httpx.AsyncClient,
    cortex_base_url: str,
    ctx: CorrelationContext,
    tenant_id: str | None = None,
    scrape_interval_sec: int = 15,
) -> str:
    if not ctx.namespaces or (not ctx.services and not ctx.pods):
        return "<cortex_metrics>\nInsufficient namespace/service/pod labels for metric correlation.\n</cortex_metrics>"

    ns_list = sorted(ctx.namespaces)
    ns_matcher = f'namespace="{ns_list[0]}"' if len(ns_list) == 1 else f'namespace=~"{"|".join(re.escape(n) for n in ns_list)}"'
    svc_regex = "|".join(re.escape(s) for s in sorted(ctx.services)[:10])
    pod_regex = "|".join(re.escape(p) for p in sorted(ctx.pods)[:10])

    center_start = (ctx.first_error_ns or ctx.start_ns) / 1_000_000_000
    center_end = (ctx.last_error_ns or ctx.end_ns) / 1_000_000_000
    start_sec = center_start - 300
    end_sec = center_end + 120
    step = max(scrape_interval_sec, int((end_sec - start_sec) // 25))

    queries: dict[str, str] = {}
    if svc_regex:
        queries["http_5xx_rate"] = (
            f'sum by (service) (rate(http_requests_total{{{ns_matcher},service=~"{svc_regex}",status=~"5.."}}[2m]))'
        )
        # Prefer Ruler recording rule `service:http_request_duration_seconds:p99_5m` if configured
        queries["latency_p99_sec"] = (
            f'histogram_quantile(0.99, sum by (le, service) (rate(http_request_duration_seconds_bucket{{{ns_matcher},service=~"{svc_regex}"}}[2m])))'
        )
    if pod_regex:
        queries["cpu_cores"] = (
            f'topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total{{{ns_matcher},pod=~"{pod_regex}",container!=""}}[2m])))'
        )
        queries["mem_working_mb"] = (
            f'topk(5, sum by (pod) (container_memory_working_set_bytes{{{ns_matcher},pod=~"{pod_regex}",container!=""}}) / 1048576)'
        )
        queries["oom_restarts_delta"] = (
            f'sum by (pod) (increase(kube_pod_container_status_restarts_total{{{ns_matcher},pod=~"{pod_regex}"}}[10m]))'
        )

    headers = {"X-Scope-OrgID": tenant_id} if tenant_id else {}

    async def _run_promql(name: str, expr: str) -> list[str]:
        resp = await client.get(
            f"{cortex_base_url.rstrip('/')}/api/v1/query_range",
            params={"query": expr, "start": start_sec, "end": end_sec, "step": f"{step}s"},
            headers=headers,
            timeout=6.0,
        )
        resp.raise_for_status()
        return summarize_cortex_matrix(resp.json().get("data", {}).get("result", []), name)

    results = await asyncio.gather(*(_run_promql(k, v) for k, v in queries.items()), return_exceptions=True)

    out = ["<cortex_metrics>"]
    for (name, _), res in zip(queries.items(), results):
        if isinstance(res, Exception):
            out.append(f"{name}: [QUERY_ERROR: {type(res).__name__}]")
        else:
            out.extend(res)
    if len(out) == 1:
        out.append("No non-zero Cortex metric series found for correlated targets.")
    out.append("</cortex_metrics>")
    return "\n".join(out)
```

---

## 6. Snowflake Optimization & Correlation (`snowflake_correlator.py`)

### 6.1 Warehouse, Caching & Concurrency Rules
- **Thread-Safe Connection Pool:** Keep $3\text{–}5$ warm connections in an `asyncio.Queue` with `client_session_keep_alive=True`. Never execute two `asyncio.to_thread()` queries concurrently on the same connection object.
- **Micro-Partition Pruning:** Never wrap timestamp columns in `DATE()` or `DATE_TRUNC()`. Always filter with explicit UTC range bounds (`ts >= %(start)s AND ts < %(end)s`).
- **24-Hour Result Cache Hit Optimization:** Never embed `CURRENT_TIMESTAMP()` inside correlation SQL, and round query window boundaries to 30-second buckets so multi-turn follow-ups hit Snowflake's zero-compute Result Cache.
- **Warehouse & Search Config:** Isolate agent queries on a dedicated interactive warehouse with `AUTO_SUSPEND = 60` (preserves Local SSD Cache across chat turns without ETL queuing), set `STATEMENT_TIMEOUT_IN_SECONDS = 10`, and enable **Search Optimization Service (SOS)** on `TRACE_ID` / `ORDER_ID` columns for multi-terabyte tables.
- **SQL-Side Aggregation & `limit + 1`:** Push `GROUP BY`, `COUNT(*)`, and `QUALIFY ROW_NUMBER()` into SQL, fetch raw tuples (not `DictCursor`) with `fetchmany(max_rows + 1)` to detect truncation, and format as compact pipe-delimited text.

```python
from __future__ import annotations

import asyncio
from contextlib import asynccontextmanager
from datetime import datetime, timedelta, timezone
from typing import Any
import snowflake.connector
from loki_pipeline import CorrelationContext, redact_secrets


def _bucket_utc_dt(ns: int, bucket_sec: int = 30) -> datetime:
    """Round epoch nanoseconds to fixed UTC buckets to maximize Snowflake Result Cache hits."""
    sec = int(ns // 1_000_000_000)
    rounded = sec - (sec % bucket_sec)
    return datetime.fromtimestamp(rounded, tz=timezone.utc)


class SnowflakePool:
    """Thread-safe connection pool for non-blocking asyncio execution in Python 3.11."""

    def __init__(self, conn_kwargs: dict[str, Any], size: int = 4):
        self._kwargs = {
            **conn_kwargs,
            "client_session_keep_alive": True,
            "session_parameters": {
                "TIMEZONE": "UTC",
                "STATEMENT_TIMEOUT_IN_SECONDS": 10,
            },
        }
        self._size = size
        self._pool: asyncio.Queue[Any] = asyncio.Queue(maxsize=size)
        self._initialized = False
        self._init_lock = asyncio.Lock()

    def _init_sync(self) -> None:
        for _ in range(self._size):
            self._pool.put_nowait(snowflake.connector.connect(**self._kwargs))

    @asynccontextmanager
    async def acquire(self):
        if not self._initialized:
            async with self._init_lock:
                if not self._initialized:
                    await asyncio.to_thread(self._init_sync)
                    self._initialized = True
        conn = await self._pool.get()
        try:
            if conn.is_closed():
                conn = await asyncio.to_thread(snowflake.connector.connect, **self._kwargs)
            yield conn
        finally:
            self._pool.put_nowait(conn)

    async def correlate_events(
        self,
        table_fqn: str,
        ts_column: str,
        ctx: CorrelationContext,
        max_rows: int = 35,
        max_cell_chars: int = 160,
    ) -> str:
        """Execute partition-pruned, parameterized correlation query and return compact pipe-delimited text."""
        conditions: list[str] = []
        params: dict[str, Any] = {}

        for idx, tid in enumerate(ctx.trace_ids[:10]):
            key = f"tid_{idx}"
            conditions.append(f"TRACE_ID = %({key})s")
            params[key] = tid
        for idx, oid in enumerate(ctx.order_ids[:10]):
            key = f"oid_{idx}"
            conditions.append(f"ORDER_ID = %({key})s")
            params[key] = oid
        for idx, uid in enumerate(ctx.user_ids[:10]):
            key = f"uid_{idx}"
            conditions.append(f"USER_ID = %({key})s")
            params[key] = uid

        if not conditions:
            return "<snowflake_rows>\nNo correlation IDs extracted from logs for Snowflake lookup.\n</snowflake_rows>"

        params["win_start"] = _bucket_utc_dt(ctx.first_error_ns or ctx.start_ns) - timedelta(minutes=15)
        params["win_end"] = _bucket_utc_dt(ctx.last_error_ns or ctx.end_ns) + timedelta(minutes=15)

        sql = f"""
            SELECT {ts_column}, SERVICE_NAME, TRACE_ID, ORDER_ID, USER_ID, EVENT_STATUS, ERROR_CODE
            FROM {table_fqn}
            WHERE {ts_column} >= %(win_start)s
              AND {ts_column} < %(win_end)s
              AND ({' OR '.join(conditions)})
            ORDER BY {ts_column} ASC
            LIMIT {int(max_rows) + 1}
        """

        def _run(conn: Any):
            with conn.cursor() as cur:
                cur.execute(sql, params)
                cols = [desc[0] for desc in (cur.description or [])]
                rows = cur.fetchmany(max_rows + 1)
                return cols, rows

        async with self.acquire() as conn:
            cols, rows = await asyncio.to_thread(_run, conn)

        out = ["<snowflake_rows>"]
        if not cols or not rows:
            out.append("No matching Snowflake analytical rows found in pruned UTC window.")
            out.append("</snowflake_rows>")
            return "\n".join(out)

        truncated = len(rows) > max_rows
        rows = rows[:max_rows]

        header = " | ".join(cols)
        out.append(header)
        out.append("-" * min(len(header), 80))
        for row in rows:
            cells: list[str] = []
            for val in row:
                text = "NULL" if val is None else redact_secrets(str(val)).replace("\n", " ")
                if len(text) > max_cell_chars:
                    text = text[:max_cell_chars] + "..."
                cells.append(text)
            out.append(" | ".join(cells))

        if truncated:
            out.append(f"[TRUNCATED: >{max_rows} rows matched; narrow time window or filter by specific trace_id]")
        out.append("</snowflake_rows>")
        return "\n".join(out)
```

---

## 7. MongoDB Operational Correlator (`mongo_correlator.py`)

```python
from __future__ import annotations

from typing import Any
from loki_pipeline import CorrelationContext, redact_secrets

ALLOWED_MONGO_INDEX_FIELDS = {"trace_id", "order_id", "user_id"}


def _compact_doc(doc: dict[str, Any], max_val_len: int = 160) -> str:
    parts: list[str] = []
    for k, v in doc.items():
        if k == "_id":
            parts.append(f"_id={v}")
            continue
        rendered = redact_secrets(str(v))
        if len(rendered) > max_val_len:
            rendered = rendered[:max_val_len] + "...[truncated]"
        parts.append(f"{k}={rendered}")
    return " | ".join(parts)


async def correlate_mongodb_state(
    collection: Any,
    ctx: CorrelationContext,
    projection: dict[str, int] | None = None,
    limit: int = 15,
) -> str:
    or_clauses: list[dict[str, Any]] = []
    if ctx.trace_ids and "trace_id" in ALLOWED_MONGO_INDEX_FIELDS:
        or_clauses.append({"trace_id": {"$in": ctx.trace_ids[:10]}})
    if ctx.order_ids and "order_id" in ALLOWED_MONGO_INDEX_FIELDS:
        or_clauses.append({"order_id": {"$in": ctx.order_ids[:10]}})
    if ctx.user_ids and "user_id" in ALLOWED_MONGO_INDEX_FIELDS:
        or_clauses.append({"user_id": {"$in": ctx.user_ids[:10]}})

    if not or_clauses:
        return "<mongodb_docs>\nNo indexed correlation IDs found in logs.\n</mongodb_docs>"

    query = {"$or": or_clauses} if len(or_clauses) > 1 else or_clauses[0]
    safe_projection = projection or {"payload_blob": 0, "raw_headers": 0, "embeddings": 0}

    cursor = collection.find(query, projection=safe_projection).max_time_ms(3000).limit(limit)
    docs = await cursor.to_list(length=limit)

    out = ["<mongodb_docs>"]
    if not docs:
        out.append("No matching MongoDB operational documents found.")
    else:
        for doc in docs:
            out.append(_compact_doc(doc))
    out.append("</mongodb_docs>")
    return "\n".join(out)
```

---

## 8. Parallel Orchestrator (`correlate_incident.py`)

```python
from __future__ import annotations

import asyncio
from collections.abc import Awaitable
from typing import Any
import httpx

from cortex_correlator import correlate_cortex_metrics
from loki_pipeline import fetch_and_compress_loki
from mongo_correlator import correlate_mongodb_state
from snowflake_correlator import SnowflakePool


async def _safe_branch(name: str, coro: Awaitable[str], timeout_sec: float = 10.0) -> str:
    try:
        return await asyncio.wait_for(coro, timeout=timeout_sec)
    except asyncio.TimeoutError:
        return f"<{name}_error>Timed out after {timeout_sec}s</{name}_error>"
    except Exception as exc:
        return f"<{name}_error>{type(exc).__name__}: {exc}</{name}_error>"


async def investigate_incident_tool(
    http_client: httpx.AsyncClient,
    snowflake_pool: SnowflakePool,
    mongo_collection: Any,
    loki_url: str,
    cortex_url: str,
    logql: str,
    start_ns: int,
    end_ns: int,
    allowed_namespaces: set[str],
    tenant_id: str | None = None,
) -> str:
    loki_text, ctx = await fetch_and_compress_loki(
        client=http_client,
        loki_base_url=loki_url,
        logql=logql,
        start_ns=start_ns,
        end_ns=end_ns,
        allowed_namespaces=allowed_namespaces,
        tenant_id=tenant_id,
    )

    async with asyncio.TaskGroup() as tg:
        cortex_task = tg.create_task(
            _safe_branch(
                "cortex",
                correlate_cortex_metrics(http_client, cortex_url, ctx, tenant_id=tenant_id),
                timeout_sec=8.0,
            )
        )
        snowflake_task = tg.create_task(
            _safe_branch(
                "snowflake",
                snowflake_pool.correlate_events(
                    table_fqn="PROD_ANALYTICS.EVENTS.ORDER_AUDIT_LOG",
                    ts_column="EVENT_TS",
                    ctx=ctx,
                ),
                timeout_sec=12.0,
            )
        )
        mongo_task = tg.create_task(
            _safe_branch(
                "mongodb",
                correlate_mongodb_state(mongo_collection, ctx),
                timeout_sec=5.0,
            )
        )

    return "\n\n".join([
        loki_text,
        cortex_task.result(),
        snowflake_task.result(),
        mongo_task.result(),
    ])
```

---

## 9. Coding Agent Verification Checklist

- [ ] **Plain-Text Boundary:** Every tool registered with the LLM returns `str`—never `dict` or `list`—with zero raw JSON envelopes.
- [ ] **Loki Integrity:** `direction=forward`, per-stream chronological sorting, multi-line trace stitching *before* windowing, secret redaction *before* templating, and `[WARNING]` headers on limit hits.
- [ ] **Cortex Optimization:** Exact label selectors where possible, `topk(5, ...)` cardinality caps, adaptive `step` sizing, Ruler recording rules for P99 histograms, and statistical digest output with `spike_onset`.
- [ ] **Snowflake Optimization:** Thread-safe `SnowflakePool` (`asyncio.Queue`), raw timestamp bounds (`ts >= %(start)s AND ts < %(end)s`), 30s bucketed timestamps for Result Cache hits, `limit + 1` truncation detection, `STATEMENT_TIMEOUT_IN_SECONDS = 10`, and pipe-delimited plain-text output.
- [ ] **MongoDB Safety & Isolation:** Indexed-field allowlist, `.max_time_ms(3000)`, large-field projection exclusion, and fault-isolated `asyncio.TaskGroup` branches.