---
title: Homelab Dashboard
page_width: full
auto_refresh: 90000
---

# Homelab Dashboard

Live health of the homelab nodes, queried from Neon and refreshed every second.

```sql current_nodes
select
  node_id,
  display_name,
  role,
  scrape_success,
  cpu_pct / 100.0 as cpu,
  memory_used_gib,
  memory_total_gib,
  memory_used_bytes::float / nullif(memory_total_bytes, 0) as mem,
  uptime_seconds / 86400.0 as uptime_days,
  collection_age_seconds,
  extract(epoch from now() - collected_at) as age_s,
  freshness_status,
  quality_status,
  expected,
  case
    when not scrape_success then 'Down'
    when extract(epoch from now() - collected_at) > 3600 then 'Stale'
    when quality_status = 'warming_up'
      and freshness_status = 'warming_up' then 'Warming up'
    when cpu_pct >= 90
      or memory_used_bytes::float / nullif(memory_total_bytes, 0) >= 0.9 then 'Critical'
    when cpu_pct >= 80
      or memory_used_bytes::float / nullif(memory_total_bytes, 0) >= 0.8 then 'Warning'
    else 'Healthy'
  end as status
from telemetry.latest_nodes
```

```sql pipeline_now
select
  source_state,
  valid_count,
  expected_count,
  extract(epoch from ingested_at - collected_at) as ingest_lag_s,
  error_summary
from telemetry.pipeline_status
```

```sql overall
select
  case
    when bool_or(n.status in ('Down', 'Stale', 'Critical')) then 'error'
    when bool_or(n.status in ('Warning', 'Warming up')) or max(p.source_state) <> 'ok' then 'warning'
    else 'ok'
  end as severity,
  count(*) filter (where n.status = 'Healthy') as healthy_nodes,
  count(*) as total_nodes,
  coalesce(
    string_agg(n.display_name || ' ' || lower(n.status), ', ') filter (where n.status <> 'Healthy'),
    'none'
  ) as node_issues,
  max(p.source_state) as pipeline_state
from {{current_nodes}} n
cross join {{pipeline_now}} p
```

```sql node_trend
select
  date_bin('5 minutes', s.collected_at, timestamptz '2026-01-01') as ts,
  n.display_name as node,
  avg(s.cpu_pct) / 100.0 as cpu,
  avg(s.memory_used_bytes::float / nullif(s.memory_total_bytes, 0)) as mem
from telemetry.node_samples s
join telemetry.nodes n using (node_id)
where s.scrape_success
  and s.collected_at >= now() - interval '24 hours'
group by 1, 2
```

```sql temps_now
select
  n.display_name as node,
  t.display_name as sensor,
  t.kind,
  t.celsius,
  t.quality_status,
  extract(epoch from now() - t.collected_at) as age_s,
  h.min_c,
  h.avg_c,
  h.max_c
from telemetry.latest_temperatures t
join telemetry.nodes n on n.node_id = t.node_id
left join (
  select entity_id, min(celsius) as min_c, avg(celsius) as avg_c, max(celsius) as max_c
  from telemetry.temperature_samples
  where celsius is not null
    and collected_at >= now() - interval '6 hours'
  group by 1
) h on h.entity_id = t.entity_id
```

```sql temp_trend
select
  date_bin('1 minute', s.collected_at, timestamptz '2026-01-01') as ts,
  n.display_name as node,
  l.kind,
  avg(s.celsius) as celsius
from telemetry.temperature_samples s
join telemetry.latest_temperatures l using (entity_id)
join telemetry.nodes n on n.node_id = l.node_id
where s.celsius is not null
  and s.collected_at >= now() - interval '6 hours'
group by 1, 2, 3
```

```sql pipeline_trend
select
  collected_at as ts,
  source_state,
  duration_seconds,
  valid_count,
  expected_count,
  extract(epoch from ingested_at - collected_at) / 60.0 as ingest_lag_min
from telemetry.collector_cycles
where collected_at >= now() - interval '6 hours'
```

{% if data="overall" where="severity = 'ok'" %}
{% callout type="success" title="All systems healthy" %}
All nodes are reporting and the collection pipeline is running normally.
{% /callout %}
{% /if %}

{% if data="overall" where="severity = 'warning'" %}
{% callout type="warning" title="Degraded" %}
Needs a look: {% value data="overall" value="string_agg(node_issues, '')" /%}. Pipeline: {% value data="overall" value="string_agg(pipeline_state, '')" /%}.
{% /callout %}
{% /if %}

{% if data="overall" where="severity = 'error'" %}
{% callout type="error" title="Attention needed" %}
Problem: {% value data="overall" value="string_agg(node_issues, '')" /%}. Pipeline: {% value data="overall" value="string_agg(pipeline_state, '')" /%}.
{% /callout %}
{% /if %}

{% big_value
    data="current_nodes"
    value="count(*) filter (where scrape_success)"
    title="Nodes Online"
    fmt="num0"
/%}

{% big_value
    data="current_nodes"
    value="avg(cpu)"
    title="Avg CPU"
    fmt="pct1"
/%}

{% big_value
    data="current_nodes"
    value="sum(memory_used_gib) / sum(memory_total_gib)"
    title="Cluster Memory Used"
    fmt="pct1"
/%}

{% big_value
    data="current_nodes"
    value="max(age_s)"
    title="Data Age (sec)"
    fmt="num1"
/%}

{% big_value
    data="pipeline_now"
    value="string_agg(source_state, '')"
    title="Pipeline State"
/%}

{% big_value
    data="pipeline_now"
    value="sum(valid_count) || ' / ' || sum(expected_count)"
    title="Nodes Valid (last cycle)"
/%}

{% heat_grid
    data="current_nodes"
    dimension="display_name"
    value="avg(cpu)"
    thresholds=[0.8, 0.9]
    lower_is_better=true
    fmt="pct1"
    title="CPU by Node"
/%}

{% heat_grid
    data="current_nodes"
    dimension="display_name"
    value="avg(mem)"
    thresholds=[0.8, 0.9]
    lower_is_better=true
    fmt="pct1"
    title="Memory by Node"
/%}

{% line_chart
    data="node_trend"
    x="ts"
    y="avg(cpu)"
    series="node"
    y_fmt="pct0"
    title="CPU Utilization by Node"
    subtitle="Last 24 hours, 5-minute average"
/%}

{% line_chart
    data="node_trend"
    x="ts"
    y="avg(mem)"
    series="node"
    y_fmt="pct0"
    title="Memory Used by Node"
    subtitle="Last 24 hours, 5-minute average"
/%}

## Pipeline Health

{% line_chart
    data="pipeline_trend"
    x="ts"
    y="avg(valid_count)"
    y_fmt="num0"
    title="Valid Nodes per Collection Cycle"
    subtitle="Last 6 hours, 5 expected. Dips mean the collector got fewer good readings."
/%}

{% line_chart
    data="pipeline_trend"
    x="ts"
    y="avg(ingest_lag_min)"
    y_fmt="num1"
    title="Ingest Lag (minutes)"
    subtitle="Time between collection and arrival in Neon. Should sit near zero."
/%}

## Temperatures

{% big_value
    data="temps_now"
    value="max(celsius) filter (where kind = 'cpu_package')"
    title="Hottest CPU"
    fmt="0'°C'"
/%}

{% big_value
    data="temps_now"
    value="avg(celsius) filter (where kind = 'cpu_package')"
    title="Avg CPU Temp"
    fmt="0.0'°C'"
/%}

{% big_value
    data="temps_now"
    value="avg(celsius) filter (where kind = 'ambient')"
    title="Avg Ambient Temp"
    fmt="0.0'°C'"
/%}

{% heat_grid
    data="temps_now"
    dimension="node"
    value="max(celsius)"
    where="kind = 'cpu_package'"
    thresholds=[70, 85]
    lower_is_better=true
    fmt="0'°C'"
    title="CPU Temperature by Node"
/%}

{% line_chart
    data="temp_trend"
    x="ts"
    y="avg(celsius)"
    series="node"
    where="kind = 'cpu_package'"
    y_fmt="0'°C'"
    title="CPU Package Temperature"
    subtitle="1-minute average, last 6 hours"
/%}

{% line_chart
    data="temp_trend"
    x="ts"
    y="avg(celsius)"
    series="node"
    where="kind = 'ambient'"
    y_fmt="0'°C'"
    title="System Ambient Temperature"
    subtitle="1-minute average, last 6 hours. Only Node C and Node D have an ambient sensor."
/%}

{% table data="temps_now" show_total_row=false %}
    {% dimension value="node" title="Node" /%}
    {% dimension value="sensor" title="Sensor" /%}
    {% dimension value="quality_status" title="Status" /%}
    {% measure value="max(celsius)" title="Now (°C)" fmt="num0" /%}
    {% measure value="max(min_c)" title="6h Min (°C)" fmt="num0" /%}
    {% measure value="max(avg_c)" title="6h Avg (°C)" fmt="num1" /%}
    {% measure value="max(max_c)" title="6h Max (°C)" fmt="num0" /%}
{% /table %}

## Node Status

{% table data="current_nodes" show_total_row=false %}
    {% dimension value="display_name" title="Node" /%}
    {% dimension value="status" title="Status" /%}
    {% dimension value="role" title="Role" /%}
    {% dimension value="freshness_status" title="Freshness" /%}
    {% measure value="avg(cpu)" title="CPU" fmt="pct1" viz="bar" /%}
    {% measure value="avg(mem)" title="Memory" fmt="pct1" viz="bar" /%}
    {% measure value="avg(memory_used_gib)" title="Mem Used (GiB)" fmt="num1" /%}
    {% measure value="avg(memory_total_gib)" title="Mem Total (GiB)" fmt="num1" /%}
    {% measure value="avg(uptime_days)" title="Uptime (days)" fmt="num1" /%}
{% /table %}
