# LAB 3 — Running Pods per Namespace

### Step 1 — Add visualization

**Add → Visualization**

Data source:

```text
Prometheus
```

### Step 2 — PromQL

Use:

```promql
count by (namespace) (
  kube_pod_info{phase="Running"}
)
```

### Step 3 — Visualization

Select:

```text
Bar chart
```

### Step 4 — Panel settings

**Title**

```text
Running Pods by Namespace
```

**X-axis**

Use the `namespace` label.

**Legend**

```text
Hidden
```

### Step 5 — Apply

Click **Apply**.

You should see bars for namespaces such as:

```text
monitoring
online-boutique
kube-system
```

### Teaching point

This is a good place to explain:

```text
kube_pod_info
      ↓
one series per pod
      ↓
count by (namespace)
      ↓
number of pods in each namespace
```

So **`count by (namespace)` is the PromQL aggregation**, while Grafana's Bar chart is only responsible for visualizing the result.

Once this is working, we'll move to **LAB 4 — Pod Restart Count (Stat)**.
