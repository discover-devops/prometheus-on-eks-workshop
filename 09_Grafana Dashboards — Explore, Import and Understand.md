# Module 9: Grafana Dashboards — Explore, Import and Understand

---

## 9.1 What We Have Right Now

At the end of Module 8 we built a custom three panel dashboard for our Online Boutique application. But Grafana is much more powerful than that.

When we installed the kube-prometheus-stack via Helm, it automatically loaded dozens of pre-built dashboards into Grafana. These dashboards were built by the Kubernetes and Prometheus community and cover every layer of your cluster from the physical node all the way down to individual containers.

In this module we will:

- Explore all the built-in dashboards that came with the installation
- Understand what each important dashboard shows and when to use it
- Import a community dashboard from grafana.com using a dashboard ID
- Understand Grafana variables and how they make dashboards dynamic
- Learn how to read dashboards under pressure during an incident

---

## 9.2 Navigating Grafana — The Basics

Open Grafana in your browser and log in with admin and admin123.

Here is a quick map of the Grafana interface:

| Location | What Is There |
|----------|--------------|
| Left sidebar, four squares icon | Dashboards — browse all dashboards |
| Left sidebar, compass icon | Explore — run ad-hoc PromQL queries |
| Left sidebar, bell icon | Alerting — view and manage alerts |
| Left sidebar, gear icon | Settings — data sources, users, preferences |
| Top right of any dashboard | Time range picker — change the time window |
| Top right of any dashboard | Refresh interval — auto refresh rate |
| Top left of any dashboard | Dashboard variables — filter by namespace, pod, node |

The two most important things to learn first are the time range picker and the dashboard variables. These are what you use during an incident to zoom into the exact time window and the exact service where the problem started.

---

## 9.3 The Built-In Dashboards — Full Tour

Click the four squares icon on the left sidebar. Click Browse. You will see a folder structure. Open the folder named after your Helm release which is usually called Default or Kubernetes.

Here are the most important dashboards and what they show:

### Dashboard 1: Kubernetes / Compute Resources / Cluster

This is your starting point for any investigation. It shows the overall health of the entire cluster in one view.

What you will see:

- CPU Usage: total CPU being used across all pods in the cluster
- CPU Requests Commitment: what percentage of total CPU has been requested by all pods. If this is near 100 percent, new pods may not be schedulable
- CPU Limits Commitment: what percentage of total CPU limit has been allocated
- Memory Usage: total memory being used across all pods
- Memory Requests Commitment: same as CPU but for memory
- Memory Limits Commitment: same as CPU limits but for memory

When to use this dashboard: First thing every morning. Also the first dashboard to open when someone says the cluster feels slow.

### Dashboard 2: Kubernetes / Compute Resources / Namespace

This breaks down the cluster view by namespace. You can select a specific namespace using the dropdown at the top of the dashboard.

Select the onlineboutique namespace. You will see CPU and memory usage broken down by each workload running inside that namespace. This immediately tells you which microservice is consuming the most resources.

When to use this dashboard: When you know there is a problem in a specific namespace and you want to see which service is responsible.

### Dashboard 3: Kubernetes / Compute Resources / Pod

This goes one level deeper and shows metrics for a specific pod. Select a namespace and then select a specific pod from the dropdown.

What you will see:

- CPU usage of each container inside the pod
- Memory usage of each container inside the pod
- CPU throttling percentage — this is very important. If a container is being throttled, it means it is trying to use more CPU than its limit allows and Kubernetes is slowing it down
- Network bytes received and transmitted by the pod

When to use this dashboard: When you have narrowed down the problem to a specific pod and want to understand exactly what is happening inside it.

### Dashboard 4: Node Exporter / Nodes

This shows the raw infrastructure metrics for each worker node. Select a node from the dropdown at the top.

What you will see:

- CPU usage broken down by mode: user, system, idle, iowait
- Memory: total, used, cached, free, available
- Disk I/O: read and write bytes per second
- Disk usage: space used and available on each filesystem
- Network: bytes received and transmitted per second on each interface
- System load average: 1 minute, 5 minute, 15 minute averages

When to use this dashboard: When you suspect the problem is at the infrastructure level rather than the application level. High disk I/O wait or high system load can cause slowness that looks like an application problem.

### Dashboard 5: Kubernetes / Networking / Cluster

This shows network traffic patterns across the entire cluster.

What you will see:

- Total bytes received and transmitted per namespace
- Network bandwidth usage by pod
- Packets dropped — dropped packets are a sign of network congestion or misconfiguration

When to use this dashboard: When users report intermittent connectivity issues or when you suspect network saturation between services.

### Dashboard 6: Kubernetes / Compute Resources / Workload

This shows metrics grouped by workload type: Deployment, DaemonSet, or StatefulSet. Select a namespace and then select a specific deployment.

This dashboard is particularly useful because it shows you all replicas of a deployment together. If you have 3 frontend replicas and one of them is consuming far more CPU than the others, this dashboard will show you that imbalance immediately.

When to use this dashboard: When you have multiple replicas of a service and want to compare them side by side.

### Dashboard 7: Alertmanager / Overview

This shows the current state of all Alertmanager alerts.

What you will see:

- Active alerts grouped by severity
- Alert firing time
- Alert labels and annotations

When to use this dashboard: During an incident, to see what alerts are currently firing and how long they have been active.

---

## 9.4 The RED Method — How to Read Dashboards During an Incident

When something breaks in production, you do not have time to explore every dashboard. You need a systematic approach. The RED method gives you that.

RED stands for:

| Letter | Metric | Question It Answers |
|--------|--------|-------------------|
| R | Rate | How many requests per second is the service handling? |
| E | Errors | How many of those requests are failing? |
| D | Duration | How long is each request taking? |

For any service that is misbehaving, open the Prometheus UI or Grafana Explore tab and run these three queries. Replace frontend with the service you are investigating.

Rate:

```
sum(rate(http_requests_total{namespace="onlineboutique", job="frontend"}[5m]))
```

Errors:

```
sum(rate(http_requests_total{namespace="onlineboutique", job="frontend", code=~"5.."}[5m]))
```

Duration:

```
histogram_quantile(0.95, rate(request_duration_seconds_bucket{namespace="onlineboutique", job="frontend"}[5m]))
```

If Rate is normal, Errors is zero, and Duration is low, the service is healthy. If any one of these looks wrong, you have found the signal. Now dig deeper into logs and traces to find the root cause.

---

## 9.5 Grafana Variables — Making Dashboards Dynamic

One of the most powerful features of Grafana is dashboard variables. Look at the top of any built-in dashboard and you will see dropdown menus for things like datasource, cluster, namespace, and pod.

These are variables. When you change a variable, every panel in the dashboard updates automatically to show data for the selected value. You do not have to edit a single query.

### How Variables Work

Variables are defined in the dashboard settings under Variables. Each variable runs a PromQL query to populate its values dynamically.

For example, the namespace variable runs a query like this:

```
label_values(kube_pod_info, namespace)
```

This query asks Prometheus: give me all unique values of the namespace label from the kube_pod_info metric. Prometheus returns a list of all namespaces that currently exist in the cluster. Grafana shows them in the dropdown.

When you select onlineboutique from the namespace dropdown, Grafana substitutes the variable into every panel query like this:

```
sum(rate(http_requests_total{namespace="$namespace"}[5m])) by (pod)
```

The $namespace becomes onlineboutique and every panel updates.

### Why This Matters

Variables mean one dashboard works for all namespaces, all nodes, all pods. You do not need to create a separate dashboard for each service. This is how production teams manage dozens of microservices with just a handful of dashboards.

---

## 9.6 Importing a Community Dashboard from Grafana.com

The community has built thousands of dashboards that you can import with a single dashboard ID. Let us import the Node Exporter Full dashboard which is one of the most popular Kubernetes dashboards available.

### Step 1: Get the Dashboard ID

The Node Exporter Full dashboard ID on grafana.com is 1860. This is a widely used, well-maintained dashboard that shows very detailed node metrics.

### Step 2: Import in Grafana

Step 1: In Grafana, click the plus icon on the left sidebar.

Step 2: Click Import dashboard.

Step 3: In the field that says Import via grafana.com, type 1860 and click Load.

Step 4: On the next screen, select Prometheus as the data source from the dropdown.

Step 5: Click Import.

The dashboard will load immediately. You will see a much more detailed view of your node metrics compared to the built-in Node Exporter dashboard. It includes:

- CPU basic and advanced metrics
- Memory detailed breakdown including swap
- Disk I/O with read and write operations separated
- Network traffic with error rates
- System processes and file descriptors
- Temperature sensors if available

### Other Useful Dashboard IDs to Import

| Dashboard ID | Name | What It Shows |
|-------------|------|--------------|
| 1860 | Node Exporter Full | Comprehensive node metrics |
| 3119 | Kubernetes cluster monitoring | Overview using cAdvisor |
| 6417 | Kubernetes Cluster Prometheus | Pod and container metrics |
| 7249 | Kubernetes Cluster | Multi-cluster overview |
| 15661 | K8S Dashboard | Microservices resource details |

---

## 9.7 The Grafana Explore Tab — Ad-Hoc Queries

The Explore tab is where you go when you do not have a dashboard for what you need to investigate. It is like a query workbench.

Click the compass icon on the left sidebar to open Explore.

Make sure Prometheus is selected as the data source at the top.

You can type any PromQL query here and see the results immediately without creating a dashboard panel.

### Useful Explore Queries for Daily Operations

Check which pods are restarting frequently:

```
increase(kube_pod_container_status_restarts_total{namespace="onlineboutique"}[1h])
```

Any pod with a value above zero has restarted in the last hour. A value above 5 is a sign of a serious problem.

Check which pods are using the most memory:

```
sort_desc(container_memory_usage_bytes{namespace="onlineboutique", container!=""})
```

This sorts all containers by memory usage, highest first. Useful for finding memory leaks.

Check CPU throttling across the cluster:

```
sum(increase(container_cpu_cfs_throttled_periods_total[5m])) by (pod, namespace) > 0
```

Any pod in this result is being CPU throttled. If a critical service appears here, you need to increase its CPU limit.

Check if any node is running low on disk space:

```
node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100
```

This gives you the percentage of free disk space on each node. If any node drops below 20 percent, start investigating.

---

## 9.8 Setting Up Time Range and Refresh Interval

Two settings that students always overlook but are critical during incidents.

### Time Range

The time range picker is in the top right corner of every dashboard. By default it shows the last 1 hour.

During an incident, set this to the last 15 minutes or last 5 minutes to see what is happening right now.

After an incident, set this to cover the full incident window. For example, if the incident started at 2 PM and ended at 3 PM, set the range to 1:45 PM to 3:15 PM to see the full picture including the build up before the incident.

You can also type exact times into the From and To fields for precision.

### Refresh Interval

Click the dropdown next to the time picker that shows Off by default. Set it to 15 seconds or 30 seconds during active monitoring.

This makes the dashboard auto-refresh and you can watch metrics change in real time without manually refreshing the browser.

During the workshop, set the refresh to 30 seconds and watch the Online Boutique metrics change as the loadgenerator keeps sending traffic.

---

## 9.9 Organizing Dashboards for Your Team

In a real production environment, good dashboard organization saves time during incidents. Here are the best practices:

### Use Folders to Organize by Team or Layer

Create folders like:

- Infrastructure — node and cluster dashboards
- Applications — application specific dashboards
- Business Metrics — dashboards for business stakeholders
- On-Call — dashboards used during incidents

### Star Your Most Used Dashboards

Click the star icon on any dashboard to add it to your Starred list. During an incident you want to reach your key dashboards in one click, not browse through folders.

### Add Annotations for Important Events

Grafana lets you add annotations to dashboards. An annotation is a vertical line with a note, like Deployed version 2.0 or Started database migration. These make it easy to correlate metric changes with deployment events.

To add an annotation, click and drag on any graph panel and type your note. It will appear as a marked line on all time series panels on that dashboard.

---

## 9.10 Common Questions from Students

### Question: The built-in dashboards show some panels as No Data. Why?

On EKS, some Kubernetes control plane metrics like etcd and the scheduler are not accessible from outside the control plane. AWS manages these components and does not expose their metrics endpoints. This is normal and expected. The panels that rely on these metrics will show No Data. Everything else works correctly.

### Question: Can I edit the built-in dashboards?

Yes but it is better to make a copy first. Click the gear icon at the top of any dashboard and select Save As. Give it a new name and save it. Now edit the copy. This way the original built-in dashboard remains intact and you can always go back to it.

### Question: How do I share a dashboard with someone who does not have a Grafana account?

Grafana has a feature called Snapshot. At the top of any dashboard, click Share and then Snapshot. This creates a read-only public URL of the current state of the dashboard at that moment. Anyone with the link can view it without logging in. This is useful for sharing incident evidence with people outside the team.

### Question: How do I save a dashboard as a JSON file and commit it to Git?

Click the gear icon at the top of the dashboard. Click JSON Model. Copy the entire JSON. Save it to a .json file in your Git repository. To restore it, use the Import dashboard option in Grafana and upload the JSON file. This is how teams manage dashboards as code.

---

## Summary of Module 9

By the end of this module you have done the following:

- Toured all seven key built-in dashboards that came with kube-prometheus-stack
- Understood what each dashboard is for and when to use it during an incident
- Learned the RED method for systematic incident investigation
- Understood how Grafana variables make dashboards dynamic and reusable
- Imported the Node Exporter Full community dashboard using dashboard ID 1860
- Used the Explore tab to run ad-hoc PromQL queries for daily operations
- Learned how to set time ranges and refresh intervals for real-time monitoring
- Understood dashboard organization best practices for production teams

In the final module we will wrap up everything we built today, recap the full architecture, discuss what comes next after this workshop, and open the floor for questions.

---
