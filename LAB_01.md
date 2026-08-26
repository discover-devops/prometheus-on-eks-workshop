

### Section 1 — Dashboard Building

**5 hands-on labs**

1. CPU — Time Series
2. Memory — Gauge
3. Running Pods — Bar Chart
4. Pod Restarts — Stat
5. HTTP/Application metric — Time Series

### Section 2 — Dynamic Dashboards

**2 theory/concept blocks**

1. Variables — how/why they work
2. Variable types + best practices/chained variables

We'll **do the labs ourselves first**, capture the actual experience, then create the student runbook.

---

# LAB 1 — CPU Usage

### Step 1 — Create Dashboard

Grafana → **Dashboards → New Dashboard**

Create:

```text
Name: Node Overview
```

For now, leave the folder/tags until the save step.

Click:

**Add visualization**

### Step 2 — Select Data Source

Choose:

```text
Prometheus
```

### Step 3 — Enter PromQL

```promql
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
```

You already verified this returns data.

### Step 4 — Visualization

Select:

```text
Time series
```

### Step 5 — Panel settings

**Title**

```text
CPU Usage %
```

**Unit**

```text
Percent (0-100)
```

**Thresholds**

```text
80 → Warning
90 → Critical
```

**Legend**

Set it to **Table** and show:

```text
Last
Min
Max
```

### Step 6 — Apply

Click:

**Apply**

You should now have your first CPU panel on the dashboard.

**Stop here.**

Once you see the CPU panel on `Node Overview`, tell me. Then we'll do **LAB 2 — Memory Usage (Gauge)**.
