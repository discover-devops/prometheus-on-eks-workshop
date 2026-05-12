# Module 6: Prometheus — Concepts and Architecture

---

## 6.1 What is Prometheus?

Prometheus is an open source monitoring and alerting toolkit. It was originally built at SoundCloud in 2012. In 2016, it joined the Cloud Native Computing Foundation as the second project ever accepted, right after Kubernetes itself. Today it is the most widely used monitoring system for Kubernetes environments worldwide.

In one line: Prometheus collects numbers from your systems over time, stores them, and lets you ask questions about them.

Those numbers are called metrics. For example:
- How many HTTP requests did the frontend receive in the last 5 minutes?
- What is the current memory usage of the cartservice pod?
- How many payment transactions failed in the last hour?

Prometheus answers all of these questions.

---

## 6.2 The Pull Model — How Prometheus Collects Metrics

This is the most important concept to understand about Prometheus. It works differently from most monitoring tools you may have seen before.

### Traditional Monitoring — Push Model

In older monitoring tools like Nagios or Zabbix, the application or agent on the server sends metrics to a central monitoring server. The application is responsible for pushing data out.

```
Application  ----pushes metrics---->  Monitoring Server
```

### Prometheus — Pull Model

Prometheus flips this completely. Prometheus itself goes to each application and pulls the metrics on a schedule. The application just needs to expose its metrics on a URL endpoint. Prometheus comes and collects them.

```
Prometheus  ----scrapes metrics---->  Application /metrics endpoint
```

This scraping happens on a regular interval. By default every 15 seconds. This is called a scrape interval.

### Why Pull is Better for Kubernetes

The pull model is better for Kubernetes for several reasons.

First, Prometheus controls what it monitors. If a pod is down, Prometheus knows because it cannot scrape it. The absence of data is itself a signal.

Second, it is easier to debug. You can curl the metrics endpoint of any pod manually to see exactly what Prometheus sees.

Third, it scales cleanly. Prometheus decides when and how often to scrape. Pods do not need to know anything about the monitoring system.

### What Does a Metrics Endpoint Look Like?

Every application that Prometheus monitors must expose a URL endpoint, typically at the path /metrics, that returns metrics in plain text format. Here is an example of what that looks like:

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1234
http_requests_total{method="POST",status="200"} 567
http_requests_total{method="GET",status="500"} 12

# HELP node_memory_MemAvailable_bytes Available memory in bytes
# TYPE node_memory_MemAvailable_bytes gauge
node_memory_MemAvailable_bytes 2147483648
```

This is human readable. You can open this URL in your browser and see raw metrics. Prometheus reads this format automatically.

---

## 6.3 Prometheus Architecture — All the Components

Prometheus is not just a single binary. It is an ecosystem of components that work together. Let us go through each one.

```
                        +------------------+
   Short lived jobs     |   Pushgateway    |
   push metrics here -->|                  |
                        +--------+---------+
                                 |
                                 | Prometheus scrapes Pushgateway
                                 |
+----------------+    scrape    +------------------+    alerts    +------------------+
|  Targets       |<-------------|                  |------------->|   Alertmanager   |
|                |              |  Prometheus      |              |                  |
| /metrics       |              |  Server          |              | Routes to email  |
| endpoints on   |              |                  |              | Slack, PagerDuty |
| pods, nodes,   |              | TSDB (storage)   |              +------------------+
| services       |              |                  |
+----------------+              +--------+---------+
                                         |
                                         | PromQL queries
                                         |
                                +--------+---------+
                                |     Grafana      |
                                |                  |
                                |   Dashboards     |
                                +------------------+
```

### Component 1: Prometheus Server

This is the brain of the entire system. The Prometheus server does three things:

First, it scrapes metrics from all configured targets at regular intervals.

Second, it stores all collected metrics in its built-in time series database called TSDB. Each metric is stored with a timestamp so you can see how it changed over time.

Third, it evaluates alerting rules. You write rules like "if error rate is above 5 percent for 2 minutes, fire an alert." The Prometheus server checks these rules continuously and sends alerts to the Alertmanager when conditions are met.

### Component 2: Exporters

Not every application can expose metrics natively in the Prometheus format. Exporters are special programs that sit next to an application, collect its metrics, translate them into the Prometheus format, and expose them on a /metrics endpoint.

Think of an exporter as a translator. The application speaks its own language. The exporter translates it into the Prometheus language.

Common exporters you will use in Kubernetes:

| Exporter | What It Monitors | Port |
|----------|-----------------|------|
| Node Exporter | CPU, memory, disk, network of each worker node | 9100 |
| kube-state-metrics | State of Kubernetes objects like pods, deployments, nodes | 8080 |
| cAdvisor | Resource usage of each container, built into the kubelet | 8080 |
| Blackbox Exporter | External endpoints, checks if URLs are reachable | 9115 |
| MySQL Exporter | MySQL database metrics | 9104 |
| Redis Exporter | Redis metrics | 9121 |

In our workshop, when we install kube-prometheus-stack via Helm, it automatically installs Node Exporter and kube-state-metrics for us. We get node and cluster metrics without any extra work.

### Component 3: Alertmanager

Prometheus decides when to fire an alert. Alertmanager decides what to do with it.

The separation exists for a good reason. If 20 pods fail at the same time, you do not want 20 separate alert emails flooding your inbox. Alertmanager handles:

Deduplication: If the same alert fires 20 times, it sends just one notification.

Grouping: Related alerts are grouped together into a single notification.

Routing: Different alerts go to different channels. Database alerts go to the database team. Frontend alerts go to the frontend team.

Silencing: During a planned maintenance window, you can silence all alerts so your on-call team is not woken up unnecessarily.

Supported notification channels include email, Slack, PagerDuty, OpsGenie, webhooks, and more.

### Component 4: Pushgateway

Remember we said Prometheus uses the pull model. Prometheus scrapes targets. But what about a batch job that runs for 30 seconds and then exits? Prometheus scrapes every 15 seconds. By the time Prometheus tries to scrape, the job is already gone.

The Pushgateway solves this. Short lived batch jobs push their metrics to the Pushgateway before they exit. The Pushgateway holds onto those metrics. Prometheus then scrapes the Pushgateway as a regular target.

Think of Pushgateway as a post box. The batch job drops its metrics in the post box before it leaves. Prometheus collects from the post box later.

Important: Pushgateway is only for short lived batch jobs. Do not use it for long running services. For those, just expose a /metrics endpoint and let Prometheus scrape directly.

### Component 5: Service Discovery

In a static environment, you configure Prometheus with a fixed list of IP addresses to scrape. This is called static configuration.

In Kubernetes, pods come and go constantly. Their IP addresses change. You cannot maintain a static list. This is where service discovery comes in.

Prometheus has built-in Kubernetes service discovery. It talks to the Kubernetes API server and automatically discovers all pods, services, and nodes in the cluster. When a new pod starts, Prometheus automatically starts scraping it. When a pod dies, Prometheus automatically removes it from its scrape targets. Zero manual configuration needed.

### Component 6: Grafana

Prometheus has a built-in web UI for running queries and seeing basic graphs. But it is minimal and not suitable for sharing with a team or management.

Grafana is a separate tool that connects to Prometheus as a data source and lets you build rich, interactive dashboards. You can create dashboards with multiple panels, each showing a different metric. You can set time ranges, create variables, and share dashboards with your team.

When we install kube-prometheus-stack via Helm in the next module, Grafana is included and pre-configured with dozens of ready-made dashboards for Kubernetes.

---

## 6.4 Metric Types in Prometheus

Prometheus has four types of metrics. Understanding these is important because the type of metric determines how you query it and what questions you can ask about it.

### Type 1: Counter

A counter is a number that only goes up. It never decreases. The only time it resets to zero is when the application restarts.

Use a counter for things you count and accumulate over time.

Real examples:
- Total number of HTTP requests received since the app started
- Total number of errors since the app started
- Total number of bytes sent over the network

The metric name for a counter always ends in _total by convention.

```
http_requests_total{method="GET", status="200"} 45231
http_requests_total{method="GET", status="500"} 89
```

Since counters only go up, you rarely look at the raw value. You use the rate() function to find how fast the counter is increasing per second.

```
rate(http_requests_total[5m])
```

This gives you the number of requests per second, averaged over the last 5 minutes.

### Type 2: Gauge

A gauge is a number that can go up and down freely. It represents the current state of something at this moment.

Use a gauge for things that have a current value that fluctuates.

Real examples:
- Current memory usage of a pod
- Current number of running pods in a namespace
- Current CPU temperature
- Current number of active connections to a database

```
container_memory_usage_bytes{pod="frontend-abc123"} 104857600
node_load1 0.72
kube_deployment_status_replicas_available{deployment="frontend"} 3
```

With gauges, you look at the current value directly. No rate() needed.

### Type 3: Histogram

A histogram measures how values are distributed across ranges called buckets.

This is best explained with an example. Imagine you want to know how long HTTP requests take. You do not just want the average. You want to know:

- How many requests completed in under 100ms?
- How many requests completed in under 500ms?
- How many requests took more than 1 second?

A histogram answers all of these. You define buckets like 0.1 seconds, 0.25 seconds, 0.5 seconds, 1 second, and 2.5 seconds. Every request is counted in the appropriate bucket.

```
http_request_duration_seconds_bucket{le="0.1"}  8900
http_request_duration_seconds_bucket{le="0.25"} 9500
http_request_duration_seconds_bucket{le="0.5"}  9800
http_request_duration_seconds_bucket{le="1.0"}  9950
http_request_duration_seconds_bucket{le="+Inf"} 10000
http_request_duration_seconds_sum   1523.4
http_request_duration_seconds_count 10000
```

The le label means less than or equal to. So 8900 requests completed in 100ms or less.

You use histogram_quantile() to calculate percentiles from a histogram.

```
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

This gives you the 95th percentile request latency over the last 5 minutes. It means 95 percent of requests completed faster than this value.

### Type 4: Summary

A summary is similar to a histogram but calculates percentiles inside the application itself before sending to Prometheus.

The key difference:
- Histogram: buckets are collected, percentiles calculated at query time by Prometheus
- Summary: percentiles are calculated inside the app and sent as pre-calculated values

For this workshop, you will mostly encounter histograms. Use histograms over summaries unless you have a specific reason not to, because histograms can be aggregated across multiple instances and summaries cannot.

### Quick Reference: Which Metric Type to Use?

| What You Want to Measure | Use This Type |
|--------------------------|--------------|
| Total requests, total errors, total bytes sent | Counter |
| Current memory, current CPU, current pod count | Gauge |
| Request duration, response size distribution, latency percentiles | Histogram |
| Pre-calculated percentiles from a single instance | Summary |

---

## 6.5 Labels — The Power of Prometheus

Labels are key-value pairs attached to every metric. They are what makes Prometheus so powerful for filtering and grouping data.

Look at this metric:

```
http_requests_total{method="GET", status="200", service="frontend"} 45231
http_requests_total{method="POST", status="200", service="frontend"} 1234
http_requests_total{method="GET", status="500", service="frontend"} 89
http_requests_total{method="GET", status="200", service="cartservice"} 9821
```

This is all the same metric name http_requests_total but with different labels. Using labels you can ask:

- Show me only GET requests: filter by method="GET"
- Show me only errors: filter by status="500"
- Show me only frontend requests: filter by service="frontend"
- Show me requests grouped by status code: group by status
- Compare frontend vs cartservice request rates: group by service

In Kubernetes, Prometheus automatically adds labels like pod, namespace, node, and container to every metric it collects. This lets you filter metrics by any Kubernetes dimension.

---

## 6.6 PromQL Basics — Querying Prometheus

PromQL stands for Prometheus Query Language. It is the language you use to ask questions of Prometheus.

You do not need to master PromQL today. But you need to understand the basics to work with dashboards and alerts.

### Instant Query

Returns the current value of a metric right now.

```
http_requests_total
```

Returns the current value of all time series with that metric name.

### Filter by Label

```
http_requests_total{status="500"}
```

Returns only the time series where the status label equals 500.

### Rate of Increase

```
rate(http_requests_total[5m])
```

Calculates how fast the counter is increasing per second, averaged over the last 5 minutes. This is the most common operation on counters.

### Aggregation

```
sum(rate(http_requests_total[5m])) by (service)
```

Sums up the request rate across all pods and groups the result by service name. This gives you the total request rate per service.

### Percentile from Histogram

```
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

Gives you the 95th percentile request latency over the last 5 minutes.

### Alert Condition Example

```
rate(http_requests_total{status="500"}[5m]) > 0.1
```

This fires when the rate of 500 errors exceeds 0.1 per second, which is 6 errors per minute.

We will use these queries in the Prometheus UI and Grafana in the next modules.

---

## 6.7 How Prometheus Fits Into Our Workshop

Here is how all the components we have discussed map to what we will actually run in our cluster:

| Prometheus Component | What Runs in Our Cluster |
|---------------------|------------------------|
| Prometheus Server | prometheus-server pod in monitoring namespace |
| Node Exporter | node-exporter DaemonSet, runs on every worker node |
| kube-state-metrics | kube-state-metrics pod, talks to Kubernetes API |
| cAdvisor | Built into the kubelet on each node, no extra install needed |
| Alertmanager | alertmanager pod in monitoring namespace |
| Grafana | grafana pod in monitoring namespace |

All of these will be installed automatically in the next module with a single Helm command using the kube-prometheus-stack chart.

---

## Summary of Module 6

By the end of this module you understand the following:

- What Prometheus is and where it came from
- How the pull model works and why it is better for Kubernetes than the push model
- All six components of the Prometheus ecosystem and what each one does
- The four metric types: Counter, Gauge, Histogram, and Summary and when to use each
- How labels work and why they make Prometheus powerful
- The basic PromQL queries you will use in the next modules

In the next module we will install the complete Prometheus and Grafana stack on our EKS cluster using a single Helm command and then explore the Prometheus UI.

---
