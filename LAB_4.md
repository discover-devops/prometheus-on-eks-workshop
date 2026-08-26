# LAB 4 — Pod Restart Count

### Step 1 — Add visualization

**Add → Visualization**

Data source:

```text
Prometheus
```

### Step 2 — PromQL

Use:

```promql
sum(
  increase(kube_pod_container_status_restarts_total[1h])
)
```

This calculates the **total number of container restarts during the last 1 hour**.

### Step 3 — Visualization

Select:

```text
Stat
```

### Step 4 — Panel settings

**Title:**

```text
Pod Restarts (Last 1h)
```

**Color mode:**

```text
Background
```

### Step 5 — Thresholds

Configure:

```text
0 → Green
1 → Yellow
5 → Red
```

### Step 6 — Apply

Click **Apply**.

You should see something like:

```text
┌─────────────────────────┐
│ Pod Restarts (Last 1h)  │
│                         │
│            0            │
│                         │
└─────────────────────────┘
```

### Teaching point

The important distinction:

```text
kube_pod_container_status_restarts_total
        ↓
counter metric
        ↓
increase(...[1h])
        ↓
restarts during last hour
        ↓
sum()
        ↓
total cluster restarts
```

This is a good example of why we don't simply display the raw counter.

**Lab 4 complete.**
