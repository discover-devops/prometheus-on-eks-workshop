## LAB 3 — Running Pods per Namespace

### Step 1 — Add visualization

In `Node Overview`:

**Add → Visualization**

Data source:

```text
Prometheus
```

### Step 2 — PromQL

Use the correct metric for pod phase:

```promql
count by (namespace) (
  kube_pod_status_phase{phase="Running"} == 1
)
```

This specifically counts pods whose current phase is `Running`.

### Step 3 — Visualization

Select:

```text
Bar chart
```

### Step 4 — Panel settings

**Title:**

```text
Running Pods by Namespace
```

**X-axis:**

```text
namespace
```

**Legend:**

```text
Hidden
```

### Step 5 — Apply

Click **Apply**.

You should see namespaces such as:

```text
kube-system
monitoring
online-boutique
```

with the number of Running pods in each.

### Teaching point

This is a useful PromQL lesson:

```text
kube_pod_status_phase
        ↓
phase="Running"
        ↓
== 1
        ↓
count by (namespace)
```

`kube_pod_info` tells us **information about the pod**, but it doesn't have the `phase` label we initially assumed.

`kube_pod_status_phase` is the correct metric when we specifically want to filter by pod phase.

**Lab 3 complete once the bar chart shows the namespace counts.**
