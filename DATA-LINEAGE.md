# Pipeline Observability dashboard — data lineage

This document answers two questions:

1. What **Outliers** means on this dashboard.
2. For **each chart and control**, where the numbers come from (system, view/table, columns, and SQL).

---

## 1. What “Outliers” means

**Outliers** are pipeline runs whose **duration is unusually high versus that pipeline’s own baseline**. They are not the same as **long-running** runs.

| Flag | Meaning | Rule used in this dashboard |
|---|---|---|
| **Long-running** | The run took longer than a **user threshold** (default 10 minutes; also 5 / 15 / 20 / custom). | `run_duration_minutes >= threshold` |
| **Outlier** | The run is **slow relative to that pipeline’s recent average**, even if it is under the long-running threshold — or much slower than usual even when it is already long. | See formula below |
| **SLA / STATUS breach** | ADM recorded at least one alert on the run (typically a failed run). | `run_alerts_count > 0` |

### Outlier formula (client-side)

From [ACR-2935](ACR-2935.txt): outliers should be detected from a **configurable threshold or statistical deviation from historical baselines**.

This dashboard uses a **baseline + guardrail** rule:

```
expected  = 7-day average duration for that pipeline (minutes)
outlier   = duration > expected + max(2 minutes, 20% of expected)
```

Worked example:

- `eli_lilly_glue_pipeline_python` 7-day average = **19.37 min**
- Guardrail = `max(2, 0.2 × 19.37)` = **3.87 min**
- Outlier if duration **> 23.24 min**
- The 21 Aug run at **23.77 min** is an outlier (and also long-running, because 23.77 ≥ 10).

A 12-minute `dag2_load_transform` run (baseline 11.4 min) is **long-running** but **not** an outlier, because 12 is not more than `11.4 + max(2, 2.28) = 13.4`.

The **Outliers** checkbox in the filter bar keeps only runs that pass this rule. Those runs also raise a **Duration outlier** alert (TIME / HIGH) in the run sheet.

`EXPECTED` in `pipeline-observability.html` is the baked-in 7-day average per pipeline. It was computed from ADM executions (query in §4). It is **not** a live ADM column.

---

## 2. Where the dashboard gets data

### Runtime (important)

`pipeline-observability.html` **does not query ADM while you use it**. Charts are computed in the browser from a **snapshot** embedded as `RAW` + `EXPECTED` + `PIPE_META` + `FAIL_ALERT` + `POLICIES` + `POLICY_EVALS` + `INCIDENTS`.

| Item | Value |
|---|---|
| Source product | Acceldata ADM (pipeline observability catalog) |
| Access path | MCP tool `user-ADM` → `pipelines_agent` (SQL against named views) |
| Snapshot time | **26 Aug 2026, 04:30 UTC** |
| Database / schema | ADM does not expose a physical `database.schema.table` name through this agent. Queries run against **catalog views** (`vw_*`). Treat those view names as the table layer. |
| Query engine | PostgreSQL-compatible SQL (filters such as `COUNT(*) FILTER (...)`, `INTERVAL`, `::numeric`) |

To refresh the HTML later, re-run the queries in §4 and replace `RAW` / `EXPECTED` / `PIPE_META` / `FAIL_ALERT` / `POLICIES` / `POLICY_EVALS` / `INCIDENTS`.

### Views used

| View | Grain | Role on this dashboard |
|---|---|---|
| **`vw_pipeline_executions_v2`** | One row per pipeline **run** | Primary fact for every chart: runs, duration, status, owner, source, alerts, spans |
| **`vw_pipeline_summary_v2`** | One row per **pipeline** | Estate size (166 pipelines), source types, owner/team/tags, failure-alert settings. Most pipelines had **no run in the 7-day window**, so glance cards show 11 pipelines in scope, not 166. |
| **`vw_pipeline_incidents_v2`** | Incident occurrence | Banner plus **Policies & alerts** tab: STATUS failures on `simple_failure_pipeline` / `eli_lilly_glue_pipeline`; TIME / CRITICAL missed start on `snowflake_copy_and_trigger_transform` |
| **`vw_pipeline_monitoring_policies_v2`** | One row per policy | **Policies & alerts** tab: start / end / duration monitors. 18 enabled policies on 15 estate pipelines; **none** of the 11 pipelines with runs in this window have a TIME policy. |
| **`vw_pipeline_monitoring_policy_executions_v2`** | One row per evaluation | **Policies & alerts** tab: last eval for the snowflake start-time policy (24 Aug, in progress). `evaluation_status` is eval health; `alert_raised` is the breach. |

Not used for charts (available in ADM, not in this HTML): `vw_pipeline_automations_v2`, `vw_pipeline_run_graph`.

### How `RAW` maps to ADM columns

Each `RAW` tuple is one execution:

```
[run_id, pipeline_name, pipeline_source_type, pipeline_owner, run_result,
 run_started_at, run_duration_minutes, run_alerts_count, run_alert_types,
 total_spans, span_success_percentage]
```

| Tuple index | ADM column (`vw_pipeline_executions_v2`) |
|---|---|
| 0 | `run_id` |
| 1 | `pipeline_name` |
| 2 | `pipeline_source_type` |
| 3 | `pipeline_owner` |
| 4 | `run_result` (`SUCCESS` / `FAILURE`) |
| 5 | `run_started_at` (stored UTC; displayed in the selected timezone) |
| 6 | `run_duration_minutes` |
| 7 | `run_alerts_count` |
| 8 | `run_alert_types` (e.g. `STATUS`) |
| 9 | `total_spans` |
| 10 | `span_success_percentage` |

Identity fields that are **not** in the `RAW` tuple are joined in the browser from `PIPE_META` (baked from `vw_pipeline_summary_v2`):

| Field | Source |
|---|---|
| `pipeline_uid` | `vw_pipeline_summary_v2.pipeline_uid` |
| Namespace | Prefix of `pipeline_uid` (everything before the last `.`) |
| Tags | `pipeline_tags` (empty on the 11 pipelines that ran in this window) |

Baked blobs used only on the **Policies & alerts** tab:

| Constant | Source |
|---|---|
| `FAIL_ALERT` | `failure_alert_enabled`, `failure_alert_severity`, `failure_alert_name` on `vw_pipeline_summary_v2` (all 12 pipelines in that tab have alerting on) |
| `POLICIES` | `vw_pipeline_monitoring_policies_v2` — the 4 TIME policies on `snowflake_copy_and_trigger_transform` (1 start, 2 end, 1 duration) |
| `POLICY_EVALS` | `vw_pipeline_monitoring_policy_executions_v2` — 1 eval in the 7-day window (start-time policy 5512, 24 Aug, `IN_PROGRESS`) |
| `INCIDENTS` | `vw_pipeline_incidents_v2` — 9 occurrences in the window (7 STATUS on `simple_failure_pipeline`, 1 STATUS on `eli_lilly_glue_pipeline`, 1 TIME CRITICAL on snowflake) |

---

## 3. Chart-by-chart lineage

All charts below start from the same in-memory set: `RAW` filtered by the selected time window, then by **multi-select** source / status / severity / owner / pipeline / namespace / tags, then search / flags. Checked values are **OR** within a filter (any selected source, any selected pipeline). Empty selection means All. **Severity** is a **derived** run attribute (highest alert on that run, or “No alerts”), not an ADM column.

Each mix card can be viewed as **pie**, **bar**, or **heatmap**. Trend charts can be **stacked bar**, **line**, or **heatmap**.

Tabs: **Overview** | **Long-running** | **Policies & alerts**. The third tab is the pipeline → policy → alert drill-down in §3.11. Sidebar filters apply to all three.

### Shared time-window filter (all charts)

| Control | Logic |
|---|---|
| Last 10 runs | First 10 rows of `RAW` (already newest-first) |
| Last 24 hours / 3 days / 7 days | `run_started_at >= snapshot_time - interval` (applied immediately) |
| Custom time range | From date+time → To date+time in the **selected timezone**, inclusive to the To minute, applied on **Apply**. Dates are not capped; this snapshot only has runs **19 Aug 2026 06:00 UTC – 26 Aug 2026 00:09 UTC**, so earlier From/To return 0 runs. |
| Custom last N runs | First N rows of `RAW`, applied on **Apply** |
| Timezone | Header search over IANA zones (`Intl.supportedValuesOf('timeZone')`). Converts table stamps, trend buckets, and custom From/To. Does not change the stored UTC `run_started_at`. |

Equivalent ADM window for the default Last 7 days:

```sql
WHERE run_started_at >= NOW() - INTERVAL '7 days'
```

At snapshot time that window is **19 Aug 04:30 UTC → 26 Aug 04:30 UTC** and contains **67 runs**.

---

### 3.1 Pipelines in scope (mix card)

| | |
|---|---|
| **What it shows** | Distinct pipelines that have ≥1 run in the current filter; split by source, namespace, or owner |
| **ADM view** | `vw_pipeline_executions_v2` plus `pipeline_uid` from `vw_pipeline_summary_v2` |
| **Columns** | `pipeline_name`, `pipeline_source_type`, `pipeline_owner`, namespace from `pipeline_uid` |
| **KPI** | `COUNT(DISTINCT pipeline_name)` among filtered runs (11 in Last 7 days, unfiltered) |
| **Chart** | Count of **runs** (not pipelines) by the selected split. View as pie / bar / heatmap. |

```sql
SELECT pipeline_source_type,
       COUNT(DISTINCT pipeline_name) AS pipelines,
       COUNT(*) AS runs
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
GROUP BY pipeline_source_type
ORDER BY runs DESC;
```

**Not** from `vw_pipeline_summary_v2` (that view has 166 pipelines, including ones with no recent runs).

---

### 3.2 Pipeline runs (mix card)

| | |
|---|---|
| **What it shows** | Run count, success rate, split by status / source / namespace. View as pie / bar / heatmap. |
| **ADM view** | `vw_pipeline_executions_v2` |
| **Columns** | `run_result`, `pipeline_source_type`, `pipeline_owner` |
| **KPI** | `COUNT(*)` |
| **Success %** | `COUNT(*) FILTER (WHERE run_result = 'SUCCESS') / COUNT(*)` |

```sql
SELECT run_result, COUNT(*) AS runs
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
GROUP BY run_result;
```

Snapshot result: **59 SUCCESS + 8 FAILURE = 67**.

---

### 3.3 Long running (mix card)

| | |
|---|---|
| **What it shows** | Runs whose duration ≥ the **Threshold** control (default 10 min) |
| **ADM view** | `vw_pipeline_executions_v2` |
| **Columns** | `run_duration_minutes`, `pipeline_name`, `pipeline_source_type` |
| **KPI** | Count of runs with `run_duration_minutes >= threshold` |
| **Recurring** | Pipelines with **2+** such overruns in the window (client-side) |

```sql
SELECT pipeline_name, COUNT(*) AS long_running_runs
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
  AND run_duration_minutes >= 10
GROUP BY pipeline_name
ORDER BY long_running_runs DESC;
```

Snapshot: **21** long-running runs, **3** recurring pipelines (`dag2_load_transform`, `eli_lilly_glue_pipeline`, `eli_lilly_glue_pipeline_python`).

Threshold is **not** stored in ADM for this widget; it is the dashboard control (5 / 10 / 15 / 20 / custom). ADM **does** have duration **policies** (`vw_pipeline_monitoring_policies_v2`, `measurement_type = 'DURATION'`) — 14 enabled — which are mentioned in the long-running banner only.

---

### 3.4 SLA / alerts (mix card)

| | |
|---|---|
| **What it shows** | Filtered **runs** classified by **highest derived severity** (Critical / High / Medium / No alerts) |
| **ADM view (raw)** | `vw_pipeline_executions_v2` (`run_result`, `run_alerts_count`, `run_duration_minutes`) plus baseline from the same view |
| **ADM view (banner incidents)** | `vw_pipeline_incidents_v2` (`incident_severity`, `incident_category`, `incident_status`) |
| **Not a direct ADM column** | Run-level “severity” on the mix card is computed in the browser. View as pie / bar / heatmap. |

Severity ranking (highest wins):

1. **Critical** — none of the 67 snapshot runs (the CRITICAL missed trigger has **no row** in the 7-day executions view).
2. **High** — duration overrun well above baseline (`duration > expected + 2`) and/or **outlier**.
3. **Medium** — failed run / STATUS alert, or overrun that is only slightly above expected.
4. **No alerts** — everything else.

```sql
-- STATUS-type alerts as stored by ADM
SELECT run_result, run_alerts_count, run_alert_types, COUNT(*) AS runs
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
GROUP BY run_result, run_alerts_count, run_alert_types;
```

High / outlier slices are **not** `run_high_alerts` from ADM. They are the client formula in §1.

---

### 3.5 Run status trend

| | |
|---|---|
| **What it shows** | Runs per day or hour. Metrics: **Status** (Success vs Failed), **Failure rate**, or **Duration** (average minutes). Chart type: stacked bar / line / heatmap. |
| **Axes** | X = date or hour in the selected timezone. Left Y = run count (or avg minutes). Right Y = failure rate % when Status/Failure rate is selected. |
| **ADM view** | `vw_pipeline_executions_v2` |
| **Columns** | `run_started_at`, `run_result`, `run_duration_minutes` |

```sql
SELECT DATE(run_started_at) AS run_day,
       COUNT(*) FILTER (WHERE run_result = 'SUCCESS') AS successful,
       COUNT(*) FILTER (WHERE run_result = 'FAILURE') AS failed
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
GROUP BY DATE(run_started_at)
ORDER BY run_day;
```

Hour grain uses `run_started_at` converted to the selected timezone, then truncated to hour. Clicking a column sets a time-bucket filter (`state.day`).

---

### 3.5b Alert trend

| | |
|---|---|
| **What it shows** | Alert counts over the same day/hour buckets. **By type**: failure (STATUS), duration overrun (TIME), duration outlier. **By severity**: High/critical, Medium, No alerts. Chart type: stacked bar / line / heatmap. |
| **Axes** | X = date or hour in the selected timezone. Y = alert (or run) count. |
| **ADM view (raw)** | `vw_pipeline_executions_v2` |
| **Not a direct ADM column** | Type and severity follow the synthetic alerts in §3.7 (same rules as the run sheet). |

---

### 3.6 Operational run monitoring (table)

| | |
|---|---|
| **What it shows** | One row per filtered execution: pipeline, namespace, source/owner, status, derived severity, start (selected timezone), duration vs expected, SLA, flags |
| **ADM view** | `vw_pipeline_executions_v2` plus `PIPE_META` |
| **Columns** | `pipeline_name`, `pipeline_source_type`, `pipeline_owner`, `run_result`, `run_id`, `run_started_at`, `run_duration_minutes` |
| **Namespace** | Prefix of `pipeline_uid` |
| **Tags** | Shown under namespace (`Untagged` when `pipeline_tags` is empty) |
| **SLA label** | Breach if `run_alerts_count > 0`; At risk if outlier or long-running; else Healthy |
| **Group by** | None, source, namespace, owner, or pipeline (client-side) |
| **“vs expected”** | Client `EXPECTED[pipeline_name]` (7-day average), not an ADM field |

```sql
SELECT pipeline_name,
       pipeline_source_type,
       pipeline_owner,
       COUNT(*) AS runs,
       COUNT(*) FILTER (WHERE run_result = 'FAILURE') AS failed
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
GROUP BY pipeline_name, pipeline_source_type, pipeline_owner
ORDER BY failed DESC, runs DESC;
```

Click a pipeline name or **Alerts** to open the right-hand sheet.

---

### 3.7 Right-hand sheet (alerts)

| | |
|---|---|
| **What it shows** | Policies configured on the selected pipeline, plus the alert list (ADM incidents merged with synthetic duration/failure alerts) |
| **ADM overlap** | Failed runs with `run_alerts_count > 0` / `run_alert_types` containing `STATUS` match ADM STATUS incidents. TIME incidents with `monitoring_policy_id` (missed start) appear here when that pipeline is selected — including `snowflake_copy_and_trigger_transform`, which has **no run** in the window |
| **Client-only** | Duration overrun and Duration outlier rows (TIME) — generated from duration vs threshold / baseline, not from `vw_pipeline_incidents_v2` |

| Alert type | Category | Severity | Source |
|---|---|---|---|
| Run failure | STATUS | MEDIUM | `run_result = 'FAILURE'` or `run_alerts_count > 0` |
| Duration overrun | TIME | HIGH or MEDIUM | `run_duration_minutes >= threshold` |
| Duration outlier | TIME | HIGH | outlier formula in §1 |

Incidents that **do** exist in ADM but have **no matching 7-day run** (missed trigger) also appear on the **Policies & alerts** tab and in the sheet for that pipeline. The yellow banner still calls them out:

```sql
SELECT incident_id, incident_status, incident_severity, incident_category,
       help_description, incident_created_at, pipeline_id, pipeline_run_id, monitoring_policy_id
FROM vw_pipeline_incidents_v2
ORDER BY incident_created_at DESC
LIMIT 20;
```

Snapshot: CRITICAL TIME incident on `snowflake_copy_and_trigger_transform` (no start 17–24 Aug, `monitoring_policy_id` 5512). That pipeline is **not** in the 67-run execution set, so Critical on the Overview severity mix is **0**.

---

### 3.9 Long-running view (second tab)

Same execution view as Overview, restricted to `run_duration_minutes >= threshold`. The third tab is **Policies & alerts** (§3.11).

| Card | Source |
|---|---|
| Long-running executions | Count of those runs; avg duration vs avg `EXPECTED` |
| Recurring pipelines | Pipelines with 2+ overruns |
| Failed while long | `run_result = 'FAILURE'` among long runs |
| Currently stalled | **0** at snapshot — no `run_result = 'RUNNING'` / `run_status = 'STARTED'` with no span progress |

Stalled check used at snapshot time:

```sql
SELECT pipeline_name, run_status, run_result, run_started_at,
       run_duration_minutes, running_spans, finished_spans, total_spans
FROM vw_pipeline_executions_v2
WHERE run_result = 'RUNNING' OR run_status = 'STARTED'
ORDER BY run_started_at DESC;
```

---

### 3.10 Filters (what each control reads)

List filters (source, status, severity, owner, pipeline, namespace, tags) are **checkbox multi-select**. Within a filter, checked values are OR’d. Other list options **cascade** to values still present after the current filters (except Tags, which also lists estate tags that have no runs in this window).

| Filter | ADM column / derivation | UI |
|---|---|---|
| Timezone | Display only; converts `run_started_at` | Searchable IANA list |
| Time range | `run_started_at` | Presets, custom date+time, custom last N runs |
| Pipeline source | `pipeline_source_type` | Multi-select checkboxes |
| Run status | `run_result` | Multi-select checkboxes |
| Severity | Derived (§3.4), not `incident_severity` | Multi-select checkboxes |
| Owner | `pipeline_owner` (blank → `unassigned`) | Multi-select checkboxes |
| Pipeline | `pipeline_name` | Multi-select checkboxes |
| Namespace / Airflow instance | Prefix of `pipeline_uid` | Multi-select checkboxes |
| Tags | `pipeline_tags`. **Untagged** = empty tags on the pipeline. Named estate tags (`rzr`, `acr`, `demo_inthecall`, `hsbc`, `rsa-demo`, `sc_9418`) exist on other pipelines that did not run in this window | Multi-select checkboxes |
| Search | `pipeline_name`, `run_id`, `pipeline_owner`, namespace, tags | Text |
| Threshold | Client only; applied to duration | 5 / 10 / 15 / 20 / custom |
| Long-running | `run_duration_minutes >= threshold` | Flag checkbox |
| SLA breaches | `run_alerts_count > 0` | Flag checkbox |
| Outliers | Client formula in §1 | Flag checkbox |

Region, subdomain, product, and environment from the PDF and ACR-2935 are **omitted** (not shown as disabled controls): `pipeline_tags` in this tenant did not carry those keys.

---

### 3.11 Policies & alerts tab (Monitored pipelines → Policies → Alert timeline)

Third tab (**Policies & alerts**). Same sidebar filters as Overview. This is the customer drill-down: **pipeline → policies on that pipeline → alerts for those policies**.

**Selection (cascading):**

1. Check one or more pipelines in panel 1 → panels 2 and 3 show only those pipelines.
2. Check one or more policies in panel 2 → panel 3 shows only those policies.
3. Click a pipeline **name** (not the checkbox) → right-hand sheet with every policy row and every related alert.
4. Click a heatmap cell → sheet for that pipeline, scoped to that day.

Empty checkbox selection means All in view.

| Panel | What it shows | ADM / client |
|---|---|---|
| **1 Monitored pipelines** | Pipelines in the current window, plus `snowflake_copy_and_trigger_transform` (TIME policies / missed-start incident, **0 runs** in this window). Columns: policies configured, runs, **failed runs**, alerts, alert-profile bar | `vw_pipeline_executions_v2` + `FAIL_ALERT` + `POLICIES` + `INCIDENTS` |
| **2 Policies** | Four rows per pipeline: **Duration**, **Start time**, **End time**, **Failure**. Columns: last start, last end, duration, failed, alerts, profile. Dimmed **Not configured** when no ADM TIME policy exists | TIME rows: `POLICIES` from `vw_pipeline_monitoring_policies_v2`. **Failure is not a monitoring policy** — it is `failure_alert_enabled` (`FAIL_ALERT`). Start/end/duration on a row are the **latest run** in the window (`run_started_at` + `run_duration_minutes`; end = start + duration), or last policy eval when there is no run |
| **3 Alert timeline** | Heatmap: policy × day in the selected timezone. Grey = ran clean, red = alert count, empty = did not run. Top 8 / 16 / All. Click a cell for the sheet | ADM `INCIDENTS` + client duration overrun/outlier (attached to Duration) + run failures (attached to Failure) |

In this snapshot:

- The 11 pipelines with runs have **no** ADM TIME policies. Only the Failure row is configured (run-failure alerting). Duration / start / end show as Not configured; Duration still collects dashboard long-running / outlier alerts.
- `snowflake_copy_and_trigger_transform` has 4 TIME policies (1 start, 2 end, 1 duration) and the 24 Aug CRITICAL missed-start incident (`monitoring_policy_id` 5512). No `pipeline_run_id` on that incident, so start/end/duration of a run are **—**.

Failed-run counts on TIME policy rows are **run failures** joined for context (the customer asked to show failure even though it is not a TIME policy). The Failure row is the actual run-failure alert.

```sql
SELECT p.monitoring_policy_id, s.pipeline_name, p.measurement_type,
       p.comparison_operator, p.comparison_threshold, p.alert_severity, p.is_enabled
FROM vw_pipeline_monitoring_policies_v2 p
JOIN vw_pipeline_summary_v2 s ON s.pipeline_id = p.pipeline_id;

SELECT i.incident_id, i.incident_status, i.incident_severity, i.incident_category,
       i.incident_created_at, s.pipeline_name, i.pipeline_run_id, i.monitoring_policy_id
FROM vw_pipeline_incidents_v2 i
JOIN vw_pipeline_summary_v2 s ON s.pipeline_id = i.pipeline_id
WHERE i.incident_created_at >= NOW() - INTERVAL '7 days';
```

---

## 4. SQL actually run against ADM (snapshot build)

All of the following were executed through **`pipelines_agent`** on **26 Aug 2026 ~04:30 UTC**. They populated `RAW`, `EXPECTED`, `PIPE_META`, banner copy, `FAIL_ALERT`, `POLICIES`, `POLICY_EVALS`, and `INCIDENTS`. The HTML file does not re-run them.

### Inventory

```sql
SELECT COUNT(DISTINCT pipeline_id) AS total_pipelines
FROM vw_pipeline_summary_v2;
```

```sql
SELECT pipeline_source_type, COUNT(*) AS pipeline_count
FROM vw_pipeline_summary_v2
GROUP BY pipeline_source_type
ORDER BY pipeline_count DESC;
```

```sql
SELECT pipeline_id, pipeline_name, pipeline_tags, pipeline_team,
       pipeline_owner, pipeline_source_type
FROM vw_pipeline_summary_v2
ORDER BY pipeline_name ASC
LIMIT 50;
```

```sql
SELECT pipeline_name, pipeline_source_type, pipeline_team, pipeline_owner,
       pipeline_tags, baseline_lookback_count, baseline_lookback_unit,
       failure_alert_enabled, failure_alert_severity
FROM vw_pipeline_summary_v2
WHERE pipeline_tags IS NOT NULL
  AND pipeline_tags <> 'NULL'
  AND pipeline_tags <> ''
ORDER BY pipeline_name
LIMIT 40;
```

### Executions (main extract baked into `RAW`)

```sql
SELECT run_id, pipeline_id, pipeline_name, pipeline_source_type,
       pipeline_owner, pipeline_team, pipeline_tags, run_status, run_result,
       run_started_at, run_finished_at, run_duration_minutes,
       run_alerts_count, run_critical_alerts, run_high_alerts, run_open_alerts,
       run_alert_types, total_spans, running_spans, errored_spans,
       finished_spans, span_success_percentage
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
ORDER BY run_started_at DESC
LIMIT 80;
```

### KPIs used to validate glance cards and time presets

```sql
SELECT COUNT(*) AS total_runs,
       COUNT(*) FILTER (WHERE run_result = 'SUCCESS') AS successful_runs,
       COUNT(*) FILTER (WHERE run_result = 'FAILURE') AS failed_runs,
       COUNT(*) FILTER (WHERE run_result = 'CANCELLED') AS cancelled_runs,
       COUNT(*) FILTER (WHERE run_result = 'RUNNING') AS running_runs,
       ROUND(AVG(run_duration_minutes)::numeric, 2) AS avg_duration_min,
       ROUND(MIN(run_duration_minutes)::numeric, 2) AS min_duration_min,
       ROUND(MAX(run_duration_minutes)::numeric, 2) AS max_duration_min
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days';
```

Same SELECT with `'24 hours'` and `'3 days'` for the other date presets.

```sql
SELECT run_status, run_result, COUNT(*) AS runs
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
GROUP BY run_status, run_result;
```

### Trend

```sql
SELECT DATE(run_started_at) AS run_day,
       COUNT(*) AS total_runs,
       COUNT(*) FILTER (WHERE run_result = 'SUCCESS') AS successful,
       COUNT(*) FILTER (WHERE run_result = 'FAILURE') AS failed,
       COUNT(*) FILTER (WHERE run_result = 'RUNNING') AS running,
       COUNT(*) FILTER (WHERE run_result = 'CANCELLED') AS cancelled,
       ROUND(AVG(run_duration_minutes)::numeric, 2) AS avg_duration_min
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
GROUP BY DATE(run_started_at)
ORDER BY run_day;
```

### Expected duration (`EXPECTED` map) and long-running

```sql
SELECT pipeline_name,
       COUNT(*) AS runs,
       COUNT(*) FILTER (WHERE run_result = 'SUCCESS') AS successes,
       COUNT(*) FILTER (WHERE run_result = 'FAILURE') AS failures,
       ROUND(AVG(run_duration_minutes)::numeric, 2) AS avg_min,
       ROUND(MAX(run_duration_minutes)::numeric, 2) AS max_min,
       COUNT(*) FILTER (WHERE run_duration_minutes >= 10) AS long_running_count
FROM vw_pipeline_executions_v2
WHERE run_started_at >= NOW() - INTERVAL '7 days'
GROUP BY pipeline_name
ORDER BY long_running_count DESC, avg_min DESC
LIMIT 25;
```

```sql
SELECT pipeline_name, pipeline_source_type, pipeline_team, pipeline_owner,
       run_result, run_status, run_started_at, run_duration_minutes,
       run_alert_types, run_open_alerts, pipeline_tags
FROM vw_pipeline_executions_v2
WHERE run_duration_minutes >= 10
   OR run_result = 'RUNNING'
   OR run_status = 'STARTED'
ORDER BY run_started_at DESC
LIMIT 40;
```

### Incidents (banner and Policies & alerts tab)

```sql
SELECT incident_status, incident_severity, incident_category,
       COUNT(DISTINCT incident_id) AS incidents
FROM vw_pipeline_incidents_v2
GROUP BY incident_status, incident_severity, incident_category
ORDER BY incidents DESC;
```

```sql
SELECT incident_id, incident_status, incident_severity, incident_category,
       help_description, incident_created_at, pipeline_id,
       pipeline_run_id, monitoring_policy_id
FROM vw_pipeline_incidents_v2
ORDER BY incident_created_at DESC
LIMIT 20;
```

### Duration policies (long-running banner)

```sql
SELECT COUNT(*) AS duration_policies,
       COUNT(*) FILTER (WHERE is_enabled) AS enabled_duration_policies
FROM vw_pipeline_monitoring_policies_v2
WHERE measurement_type = 'DURATION';
```

```sql
SELECT policy_name, policy_category, measurement_type, comparison_type,
       comparison_operator, comparison_threshold, comparison_units,
       alert_severity, is_enabled, monitored_entity_type, COUNT(*) AS policy_count
FROM vw_pipeline_monitoring_policies_v2
GROUP BY policy_name, policy_category, measurement_type, comparison_type,
         comparison_operator, comparison_threshold, comparison_units,
         alert_severity, is_enabled, monitored_entity_type
ORDER BY policy_count DESC
LIMIT 40;
```

### Policies & alerts tab (`FAIL_ALERT`, `POLICIES`, `POLICY_EVALS`, `INCIDENTS`)

```sql
SELECT pipeline_id, pipeline_name, pipeline_uid, pipeline_source_type,
       pipeline_owner, failure_alert_enabled, failure_alert_severity, failure_alert_name
FROM vw_pipeline_summary_v2
WHERE pipeline_name IN (
  'databricks_shell','krish_dbt_job','dag2_load_transform','simple_failure_pipeline',
  'listPackages','dag1_ingest_s3_to_s3','adoc_bronze_to_silver_clinical_pipeline',
  'eli_lilly_glue_pipeline_python','eli_lilly_glue_pipeline','adoc_openlineage_validation',
  'manual_adoc_openlineage_validation','snowflake_copy_and_trigger_transform'
);
```

```sql
SELECT p.monitoring_policy_id, p.pipeline_id, s.pipeline_name, p.policy_name,
       p.policy_category, p.measurement_type, p.alert_severity, p.is_enabled,
       p.comparison_type, p.comparison_operator, p.comparison_threshold, p.comparison_units
FROM vw_pipeline_monitoring_policies_v2 p
LEFT JOIN vw_pipeline_summary_v2 s ON s.pipeline_id = p.pipeline_id
ORDER BY s.pipeline_name, p.policy_name;
```

```sql
SELECT e.execution_id, e.monitoring_policy_id, p.policy_name, p.measurement_type,
       s.pipeline_name, e.pipeline_run_id, e.evaluation_status, e.alert_raised, e.evaluated_at,
       r.run_started_at, r.run_finished_at, r.run_duration_minutes, r.run_result
FROM vw_pipeline_monitoring_policy_executions_v2 e
LEFT JOIN vw_pipeline_monitoring_policies_v2 p ON p.monitoring_policy_id = e.monitoring_policy_id
LEFT JOIN vw_pipeline_summary_v2 s ON s.pipeline_id = e.pipeline_id
LEFT JOIN vw_pipeline_executions_v2 r ON r.run_id = e.pipeline_run_id
WHERE e.evaluated_at >= NOW() - INTERVAL '7 days'
ORDER BY e.evaluated_at DESC;
```

---

## 5. What is computed only in the browser (not in ADM)

| Metric | Rule |
|---|---|
| Long-running | `duration >= threshold` |
| Recurring overrun | Same pipeline has ≥2 long-running runs in the window |
| Outlier | `duration > expected + max(2, 0.2 × expected)` |
| Expected duration | 7-day `AVG(run_duration_minutes)` stored in `EXPECTED` |
| Run severity | Highest of Critical / High / Medium among synthetic alerts, else No alerts |
| Duration overrun alert | Long-running run; HIGH if `duration > expected + 2`, else MEDIUM |
| Duration outlier alert | Outlier runs, TIME / HIGH |
| Success / failure rate | From `run_result` counts |
| Stalled | No live RUNNING rows at snapshot → 0 |
| Namespace | Prefix of `pipeline_uid` |
| Untagged | `pipeline_tags` empty or null |
| Timezone display | `run_started_at` (UTC) formatted with `Intl` in the selected IANA zone |
| Custom time range bounds | Calendar date + clock time interpreted in the selected timezone |
| Mix / trend chart types | Client drawing only (pie, bar, heatmap, line) |
| Failure “policy” on the Policies & alerts tab | `failure_alert_enabled` / `failure_alert_severity` from `vw_pipeline_summary_v2`, not `vw_pipeline_monitoring_policies_v2` |
| Policy start / end / duration columns | Latest run `run_started_at` + `run_duration_minutes` (end = start + duration). Empty when the pipeline has no run in the window |

---

## 6. Gaps vs Merck PDF / ACR filters

These dimensions were requested (PDF, ACR-2934, ACR-2935) but **are not populated** on the snapshot pipelines, so they are **not shown** on the HTML dashboard (not disabled placeholders):

- Region, Product, Business Domain, Environment, Subdomain

**Namespace** is derived from `pipeline_uid` (Airflow instance / project prefix) and is an active multi-select filter. Values in this window: `mwaa_aws`, `airflow-on-prem-validation`, `manual-airflow-prod-onprem`, `krish_pipeline`, `krish_dbt.krish_dbt_demo`.

**Tags:** `pipeline_tags` exists on `vw_pipeline_summary_v2` / `vw_pipeline_executions_v2`. The 11 pipelines with runs in this window have **no tags** (**Untagged**). Other estate pipelines carry `rzr`, `acr`, `demo_inthecall`, `hsbc`, `rsa-demo`, `sc_9418` — those values stay in the Tags list labeled “(no runs in window)”; selecting one shows 0 runs in this snapshot.

Live auto-refresh (ACR-2935) is **not** implemented; refresh means re-query ADM and replace `RAW`.
