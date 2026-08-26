# LAB 2 — Memory Usage (Gauge)

### Step 1 — Add visualization

On `Node Overview`:

**Add → Visualization**

Data source:

```text
Prometheus
```

### Step 2 — PromQL

Use the actual metrics we verified:

```promql
(
  node_memory_MemTotal_bytes
  -
  node_memory_MemAvailable_bytes
)
/
node_memory_MemTotal_bytes
* 100
```

### Step 3 — Visualization

Select:

```text
Gauge
```

### Step 4 — Panel settings

**Title**

```text
Memory Usage %
```

**Unit**

```text
Percent (0-100)
```

**Min**

```text
0
```

**Max**

```text
100
```

### Step 5 — Thresholds

Configure:

```text
70 → Yellow / Warning
85 → Red / Critical
```

### Step 6 — Apply

Click **Apply**.

You should now have:

```text
┌──────────────────────┐
│ CPU Usage %          │
│      Time Series     │
└──────────────────────┘

┌──────────────────────┐
│ Memory Usage %       │
│        Gauge         │
└──────────────────────┘
```

**Stop here.**

Once the gauge is working, we'll do **LAB 3 — Running Pods per Namespace (Bar Chart)**.
