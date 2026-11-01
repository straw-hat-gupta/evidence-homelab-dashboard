---
title: Home Lab (Claude)
page_width: full
auto_refresh: 1000
---

<!--
Built by Claude via the Evidence Studio MCP on 2026-10-31 (independent of the existing "Homelab Overview" page).
Data: Neon Postgres, schema `telemetry`, direct connector.
auto_refresh is in milliseconds (page-settings docs). Only node CPU/memory change ~every 1 s;
temperature/storage/network ~15 s, services ~30 s, collector cycles and history ~60 s.
-->

```sql issues
select 'critical' as severity, 'Node' as area, display_name || ': ' || case when not scrape_success then 'scrape failed' else 'data stale (' || round(collection_age_seconds)::text || ' s)' end as detail
from telemetry.latest_nodes where expected and (not scrape_success or collection_age_seconds > 10)
union all
select 'critical', 'Service', node_id || ' / ' || display_name || ': ' || coalesce(failure_reason, case when not running then 'not running' else 'check failed' end)
from telemetry.latest_services where expected and (not running or not check_success)
union all
select 'critical', 'Pipeline', 'collector diagnostic ' || round(diagnostic_age_seconds)::text || ' s old'
from telemetry.pipeline_status where diagnostic_age_seconds > 180
union all
select 'critical', 'Pipeline', 'collector state ' || source_state || coalesce(': ' || error_summary, '')
from telemetry.pipeline_status where source_state <> 'ok' or queue_overflow
union all
select 'warning', 'Pipeline', pending_cycles::text || ' cycles queued for upload'
from telemetry.pipeline_status where pending_cycles > 0
union all
select 'warning', 'Node', display_name || ': CPU ' || round(cpu_pct::numeric)::text || '%'
from telemetry.latest_nodes where cpu_pct >= 85
union all
select 'warning', 'Node', display_name || ': memory ' || round(100 * memory_used_gib / nullif(memory_total_gib, 0))::text || '%'
from telemetry.latest_nodes where memory_used_gib / nullif(memory_total_gib, 0) >= 0.9
union all
select case when used_pct >= 95 or device_error then 'critical' else 'warning' end, 'Storage', node_id || ' / ' || display_name || ': ' || round(used_pct::numeric)::text || '% used'
from telemetry.latest_storage where used_pct >= 85 or device_error or readonly
union all
select case when celsius >= 90 then 'critical' else 'warning' end, 'Temperature', node_id || ' / ' || display_name || ': ' || round(celsius::numeric)::text || ' °C'
from telemetry.latest_temperatures where celsius >= 80
```

```sql nodes
select
  display_name as node,
  role,
  logical_cpus,
  cpu_pct,
  100 * memory_used_gib / nullif(memory_total_gib, 0) as mem_pct,
  memory_used_gib,
  memory_total_gib,
  uptime_seconds / 86400.0 as uptime_days,
  collection_age_seconds as data_age_s,
  case when scrape_success then 'up' else 'DOWN' end as scrape,
  freshness_status
from telemetry.latest_nodes
where expected
```

```sql node_trend
select h.evaluated_at, n.display_name as node, h.cpu_pct,
       100.0 * h.memory_used_bytes / nullif(h.memory_total_bytes, 0) as mem_pct
from telemetry.node_history h
join telemetry.nodes n using (node_id)
where h.evaluated_at > now() - interval '3 hours'
```

```sql pipeline
select
  source_state,
  extract(epoch from now() - latest_current_ingested_at) as current_age_s,
  diagnostic_age_seconds,
  duration_seconds,
  valid_count,
  expected_count,
  pending_cycles,
  pending_bytes,
  missed_deadlines,
  queue_overflow
from telemetry.pipeline_status
```

```sql cycles
select evaluated_at,
       extract(epoch from ingested_at - collected_at) as upload_lag_s,
       duration_seconds,
       valid_count
from telemetry.collector_cycles
where evaluated_at > now() - interval '6 hours'
```

```sql temps
select upper(replace(node_id, 'node-', '')) || ' ' || case kind when 'cpu_package' then 'CPU' when 'ambient' then 'Ambient' when 'gpu' then 'GPU' else kind end as sensor,
       celsius, source_age_seconds
from telemetry.latest_temperatures
where celsius is not null
```

```sql temps_unavailable
select node_id, display_name, kind, scope, quality_status
from telemetry.latest_temperatures
where celsius is null
```

```sql temp_hist
select bucket_start,
       upper(replace(node_id, 'node-', '')) || ' ' || case kind when 'cpu_package' then 'CPU' when 'ambient' then 'Ambient' else kind end as sensor,
       celsius_mean
from telemetry.temperature_history
where bucket_start > now() - interval '6 hours' and celsius_mean is not null
```

```sql services
select node_id as node, display_name as service, kind, runtime_state, health_status,
       case when running and check_success then 'ok' else 'FAIL' end as status,
       restart_count, freshness_status
from telemetry.latest_services
where expected
```

```sql storage
select upper(replace(node_id, 'node-', '')) || ' ' || display_name as volume, kind, used_pct, used_gib, total_gib, available_gib
from telemetry.latest_storage
where expected and total_gib > 1
```

```sql network
select node_id as node, display_name as interface, rx_mbps, tx_mbps,
       coalesce(rx_errors_per_second, 0) + coalesce(tx_errors_per_second, 0) as errors_per_s,
       coalesce(rx_drops_per_second, 0) + coalesce(tx_drops_per_second, 0) as drops_per_s,
       link_up
from telemetry.latest_network
```

# Home Lab — live status made changes

{% if data="issues" where="severity = 'critical'" %}
{% callout type="error" title="Critical: something needs attention" %}
A node, service, or the collector is failing or stale. Details are listed under **Open issues**.
{% /callout %}
{% /if %}
{% else_if data="issues" %}
{% callout type="warning" title="Warning: running hot" %}
Everything is up, but at least one resource crossed a warning threshold. See **Open issues**.
{% /callout %}
{% /else_if %}
{% else %}
{% callout type="success" title="All systems healthy" %}
Every node is reporting, every monitored service is running, and the collector is on schedule.
{% /callout %}
{% /else %}

{% if data="issues" %}
### Open issues
{% table data="issues" /%}
{% /if %}

{% row %}
{% big_value data="nodes" value="count(*)" title="Nodes reporting" fmt="num0" /%}
{% big_value data="nodes" value="avg(cpu_pct)" title="Avg CPU %" fmt="num1" /%}
{% big_value data="nodes" value="sum(memory_used_gib)" title="Memory used (GiB)" fmt="num1" /%}
{% big_value data="nodes" value="max(data_age_s)" title="Oldest node sample (s)" fmt="num1" /%}
{% big_value data="services" value="count(*)" title="Services monitored" fmt="num0" /%}
{% /row %}

## Node health

{% heat_grid data="nodes" dimension="node" value="max(cpu_pct)" thresholds=[60, 85] units="% CPU" lower_is_better=true title="CPU" order="node" fmt="num1" /%}
{% heat_grid data="nodes" dimension="node" value="max(mem_pct)" thresholds=[75, 90] units="% memory" lower_is_better=true title="Memory" order="node" fmt="num0" /%}

{% table data="nodes" order="node" /%}

{% row %}
{% line_chart data="node_trend" x="evaluated_at" y="avg(cpu_pct)" series="node" title="CPU %, last 3 h (1-min buckets)" /%}
{% line_chart data="node_trend" x="evaluated_at" y="avg(mem_pct)" series="node" title="Memory %, last 3 h (1-min buckets)" /%}
{% /row %}

## Pipeline health

{% row %}
{% big_value data="pipeline" value="max(current_age_s)" title="Latest ingest age (s)" fmt="num1" info="Seconds since the newest current-state row reached Neon." /%}
{% big_value data="pipeline" value="max(diagnostic_age_seconds)" title="Last cycle diagnostic (s)" fmt="num0" info="Collector cycles are summarised once a minute." /%}
{% big_value data="pipeline" value="sum(valid_count)" title="Valid nodes last cycle" fmt="num0" /%}
{% big_value data="pipeline" value="sum(pending_cycles)" title="Queued cycles" fmt="num0" /%}
{% big_value data="pipeline" value="sum(missed_deadlines)" title="Missed deadlines" fmt="num0" info="Reported by the collector; constant at 50 during this build — meaning unconfirmed." /%}
{% /row %}

{% line_chart data="cycles" x="evaluated_at" y="max(upload_lag_s)" title="Upload lag per collector cycle (s), last 6 h" /%}

## Temperatures

{% heat_grid data="temps" dimension="sensor" value="max(celsius)" thresholds=[65, 80] units="°C" lower_is_better=true title="Current temperature" order="sensor" fmt="num0" /%}

{% line_chart data="temp_hist" x="bucket_start" y="avg(celsius_mean)" series="sensor" title="Temperature, last 6 h (°C, 1-min mean)" /%}

{% if data="temps_unavailable" %}
{% callout type="info" title="Sensors without a reading" %}
These expected sensors report no temperature (for example, a GPU passed through to a guest). They are excluded from the grid above rather than shown as 0 °C.
{% /callout %}
{% table data="temps_unavailable" /%}
{% /if %}

## Services

{% table data="services" order="status, node, service" /%}

## Storage and network

{% row %}
{% horizontal_bar_chart data="storage" x="max(used_pct)" y="volume" title="Storage used %" order="volume" /%}
{% table data="network" order="node" /%}
{% /row %}
