Perfect.

Since we're staying with **real metrics from our environment**, we'll make Lab 5 a useful **Online Boutique Container CPU Usage** panel rather than inventing `http_requests_total`.

# LAB 5 — Online Boutique CPU Usage

### Step 1 — Add visualization

**Add → Visualization**

Data source:

```text
Prometheus
```

### Step 2 — PromQL

Use:

```promql
sum by (pod) (
  rate(container_cpu_usage_seconds_total{namespace="online-boutique"}[5m])
)
```

This gives CPU usage rate for each Online Boutique pod.

### Step 3 — Visualization

Select:

```text
Time series
```

### Step 4 — Panel settings

**Title:**

```text
Online Boutique Container CPU Usage
```

**Unit:**

```text
CPU Cores
```

**Fill opacity:**

```text
10
```

**Line width:**

```text
2
```

### Step 5 — Apply

Click **Apply**.

---

### Important teaching point

This is actually better for teaching the metric concept:

```text
container_cpu_usage_seconds_total
              ↓
        rate(...[5m])
              ↓
       CPU usage / second
              ↓
        Grafana panel
```

And students can see that the metric name comes from the **metric producer (cAdvisor/kubelet)**, Prometheus stores/queries it, and Grafana visualizes it.

**With this, all 5 hands-on panels are complete.**

Next we should do the **Section 1 dashboard save/organization**, then move to **Section 2: Variables**.
