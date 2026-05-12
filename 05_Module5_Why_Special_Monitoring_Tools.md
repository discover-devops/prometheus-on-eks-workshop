# Module 5: Why Do We Need Special Monitoring Tools?

---

## 5.1 Let Us Start With a Story

Imagine it is a Friday evening. Your Online Boutique application is running fine. Suddenly your phone rings. A customer says they cannot checkout. They are getting an error on the payment page.

You log into the server. Everything looks fine. The application is running. The server CPU is normal. Memory is fine.

But somewhere deep inside, the paymentservice pod is throwing errors. The checkoutservice is waiting for a response from paymentservice and timing out. The frontend is showing a vague error to the user.

You have no idea where the problem is. You have 11 services. Which one is broken? When did it start? How many users were affected?

Without the right monitoring tools, you are completely blind.

This module explains why traditional monitoring tools are not enough for microservices on Kubernetes, and what kind of tool we actually need.

---

## 5.2 How Did We Monitor Traditional Applications?

Let us go back to the old world. A monolithic application running on a single Linux server.

Monitoring was simple because everything was in one place:

| What We Monitored | How We Monitored It |
|------------------|-------------------|
| CPU usage | top, htop, nagios |
| Memory usage | free, nagios |
| Disk usage | df, nagios |
| Application running or not | Check if process is alive |
| Application logs | tail -f /var/log/app.log |
| Network traffic | netstat, iftop |

Tools like Nagios, Zabbix, or even simple shell scripts were enough. You had one server, one process, one log file. If the server was up and the process was running, things were probably fine.

This is called infrastructure monitoring. You monitor the machine and assume the application is okay.

---

## 5.3 What Changed With Microservices on Kubernetes?

Now look at what we just deployed. 11 services. Each service has its own pod. Pods run on different nodes. Pods can die and restart automatically. New pods get new IP addresses. The cluster itself has nodes, namespaces, deployments, replicasets, and more.

Here is what is fundamentally different:

### Problem 1: You Cannot SSH Into a Pod Like a Server

In the old world, you SSH into your server and check things manually.

In Kubernetes, pods are ephemeral. They come and go. They restart. They move between nodes. You cannot rely on SSHing into a pod because the pod might not be there in 5 minutes. You need a monitoring system that automatically tracks every pod, even when they restart and get new names and IPs.

### Problem 2: One Service Failure Looks Like Another Service Failure

Remember our story at the start. The payment service was failing but the symptom appeared on the frontend. In a microservice architecture, one broken service causes a chain of failures across other services.

Without monitoring each individual service separately, you cannot trace the root cause. You need metrics at the service level, not just the server level.

### Problem 3: Dynamic Scaling Makes Static Monitoring Useless

In the old world, you tell your monitoring tool the IP address of your server and it monitors that IP forever.

In Kubernetes, pods scale up and down. When load increases, Kubernetes might spin up 5 more frontend pods. When load decreases, it kills them. Each pod has a different IP. A static monitoring tool cannot keep up with this. You need a monitoring system that automatically discovers new pods as they appear and stops monitoring them when they disappear.

### Problem 4: You Need Application Level Metrics, Not Just Infrastructure Metrics

Knowing that CPU is 60 percent does not tell you:

- How many requests per second is the frontend receiving?
- What is the average checkout time?
- How many payment transactions failed in the last 5 minutes?
- Which product is being viewed the most?

These are application level metrics. They come from inside the application, not from the server. Traditional tools do not collect these.

### Problem 5: Containers Have No Persistent Logs in One Place

In a monolith, all logs go to one file on one server. You tail that file.

In Kubernetes, each pod has its own logs. When a pod restarts, its logs are gone. When you have 50 pods across 2 nodes, you have 50 different log streams. You need a centralized logging system.

---

## 5.4 The Three Pillars of Observability

The industry has settled on three things you need to truly understand what is happening inside a distributed system. These are called the Three Pillars of Observability.

### Pillar 1: Metrics

Metrics are numbers measured over time.

Examples:
- CPU usage of the frontend pod: 45 percent at 2:00 PM, 78 percent at 2:05 PM
- Number of HTTP requests per second: 120 at 2:00 PM, 340 at 2:05 PM
- Number of failed payments in last 1 minute: 0 at 2:00 PM, 47 at 2:05 PM

Metrics tell you WHAT is happening and WHEN it started.

Tool for this: Prometheus

### Pillar 2: Logs

Logs are text records of events that happened inside your application.

Examples:
- "User ID 12345 added item to cart at 2:04:32 PM"
- "Payment failed for order 67890: card declined"
- "Connection to database timed out after 30 seconds"

Logs tell you WHY something happened. They give you the detailed story.

Tool for this: EFK Stack (Elasticsearch, Fluentd, Kibana) or ELK Stack (Elasticsearch, Logstash, Kibana)

### Pillar 3: Traces

Traces follow a single request as it travels across multiple services.

Example:
A user clicks checkout. The request goes through frontend, then checkoutservice, then paymentservice, then emailservice. A trace shows you the entire journey with the time spent in each service.

Traces tell you WHERE the slowness or failure is in the chain.

Tool for this: Jaeger, Zipkin, or AWS X-Ray

---

## 5.5 The Monitoring Tools Landscape

Here is the full picture of tools used in production Kubernetes environments:

| Pillar | Problem It Solves | Popular Tools |
|--------|------------------|--------------|
| Metrics | What is happening, when did it start, is it getting worse | Prometheus plus Grafana |
| Logs | Why did it happen, what is the detailed error message | EFK Stack, ELK Stack, Loki |
| Traces | Which service in the chain is slow or broken | Jaeger, Zipkin, AWS X-Ray |

In a real production environment, you would set up all three. But in this workshop, we are focusing on Metrics using Prometheus and Grafana because:

- It is the most commonly used tool in Kubernetes environments
- It is the default monitoring stack in almost every company using Kubernetes
- It is the best starting point before moving to logs and traces
- Prometheus and Kubernetes were both born from Google's internal tools and are designed to work together natively

---

## 5.6 Why Prometheus Specifically?

You might ask: there are many monitoring tools. CloudWatch, Datadog, New Relic, Dynatrace. Why Prometheus?

Here are the reasons:

### Reason 1: Prometheus Was Built for Dynamic Environments

Traditional tools work with fixed IP addresses. Prometheus uses service discovery. It talks to the Kubernetes API and automatically finds all pods, nodes, and services. When a new pod starts, Prometheus discovers it automatically. When a pod dies, Prometheus stops collecting from it. No manual configuration needed.

### Reason 2: Kubernetes and Prometheus Share the Same DNA

Kubernetes was inspired by Google's internal cluster manager called Borg. Prometheus was inspired by Google's internal monitoring system called Borgmon. They were designed with the same philosophy. This is why Kubernetes components like the API server, kubelet, and etcd natively expose metrics in the Prometheus format. No extra work needed to get cluster level metrics.

### Reason 3: Pull Based Model is Perfect for Kubernetes

Prometheus uses a pull model. Instead of applications pushing metrics to a central server, Prometheus goes and collects metrics from each application on a schedule. This is called scraping.

In Kubernetes this is perfect because:
- Prometheus controls what it monitors
- If a pod is slow or down, Prometheus knows because it cannot scrape it
- No risk of a broken pod flooding the monitoring system with bad data

### Reason 4: Labels Match Kubernetes Labels

Everything in Kubernetes uses labels. Pods have labels. Services have labels. Nodes have labels. Prometheus also uses labels as its primary way of organizing data. This makes it natural to ask questions like:

- Give me the CPU usage of all pods with label app=frontend
- Show me error rates for all pods in namespace onlineboutique
- Compare request rates between version v1 and v2 of the payment service

### Reason 5: It is Free and Open Source

Prometheus is a CNCF graduated project. The same foundation that governs Kubernetes. It is completely free, open source, and has a massive community. No vendor lock-in. No per-host licensing fees.

### Reason 6: Grafana Makes It Beautiful

Prometheus collects the data. Grafana visualizes it. Together they give you rich, interactive dashboards that any team member can use, from developers to management.

---

## 5.7 What Prometheus Cannot Do

It is also important to be honest about what Prometheus is NOT good at:

| Limitation | Explanation |
|-----------|-------------|
| Long term storage | Prometheus stores data locally. By default it keeps 15 days of data. For longer retention you need Thanos or Cortex |
| Logs | Prometheus only handles metrics, numbers. It does not collect or store log text. Use EFK or Loki for logs |
| Distributed tracing | Prometheus cannot trace a request across services. Use Jaeger or Zipkin for this |
| 100 percent accuracy billing | Prometheus scrapes on intervals so it can miss very short lived events. Not suitable for per-request billing |

---

## 5.8 The Answer to Our Story

Remember the story at the beginning of this module. Payment service failing. You had no idea where to look.

With Prometheus in place, here is what would have happened instead:

- Grafana dashboard shows a spike in errors from paymentservice at exactly 7:43 PM
- Prometheus alert fires and sends a message to Slack: "paymentservice error rate above 5 percent for 2 minutes"
- You open Grafana, look at the paymentservice dashboard
- You see request latency jumped from 200ms to 8000ms at 7:43 PM
- You check the logs in EFK and find: "Database connection pool exhausted"
- Root cause found in under 3 minutes

That is the power of proper observability.

---

## Summary of Module 5

By the end of this module you understand the following:

- Why traditional monitoring tools are not enough for microservices on Kubernetes
- The five fundamental problems that Kubernetes creates for monitoring
- The three pillars of observability: Metrics, Logs, and Traces
- The full monitoring tools landscape and what each tool does
- Why Prometheus is the natural choice for Kubernetes metrics monitoring
- What Prometheus cannot do and where other tools fill the gap

In the next module we will dive into Prometheus itself. We will understand its architecture, its components, and how it actually collects metrics before we install it on our cluster.

---
