# Module 10: Wrap-Up, What We Built, and What Comes Next

---

## 10.1 Take a Moment — Look at What You Built Today

Two hours ago you started with an empty AWS account and a Windows laptop with PuTTY.

Look at what is running right now:

A 2-node production-grade Kubernetes cluster on AWS EKS. An 11-service microservice e-commerce application receiving real traffic. An NGINX Ingress Controller routing that traffic through a single AWS Load Balancer. A complete monitoring stack with Prometheus collecting metrics from every layer of the system. Grafana with dozens of dashboards showing you exactly what is happening inside your cluster in real time. A custom ServiceMonitor connecting Prometheus directly to your application. Your own custom Grafana dashboard showing request rate, error rate, and latency for the Online Boutique application.

This is not a toy setup. This is the exact same architecture that companies run in production. The only difference between what you built today and what Netflix or Airbnb runs is scale.

---

## 10.2 The Full Architecture — Everything We Built

Here is the complete picture of everything running in your AWS account right now.

### Layer 1: Your Laptop

You used a Windows laptop with PuTTY to SSH into the EC2 Jump Box. Your laptop only needed a browser and PuTTY. Everything else ran on the cloud.

### Layer 2: EC2 Jump Box

An Ubuntu EC2 instance that served as your command center. It had eksctl, kubectl, helm, aws cli, and git installed. All commands were run from here.

### Layer 3: AWS EKS Cluster

A 2-node Kubernetes cluster created with eksctl using a YAML configuration file. Each node is a t3.medium EC2 instance with 20GB gp2 disk. The control plane is fully managed by AWS.

### Layer 4: Networking

An AWS Network Load Balancer created by the NGINX Ingress Controller. The NLB receives all external traffic and passes it to the NGINX Ingress Controller pod inside the cluster. The Ingress Controller reads routing rules and forwards traffic to the correct service.

### Layer 5: The Application

Google Online Boutique deployed using a single kubectl apply command. 11 microservices running as pods across 2 nodes. All services communicate internally using gRPC over ClusterIP services. A loadgenerator pod automatically sends realistic traffic to the application 24/7.

### Layer 6: The Monitoring Stack

Installed using a single helm install command from the kube-prometheus-stack chart.

| Component | What It Does |
|-----------|-------------|
| Prometheus Operator | Watches for ServiceMonitor resources and configures Prometheus automatically |
| Prometheus Server | Scrapes metrics from all targets every 15 seconds and stores them |
| Node Exporter | Runs on every node and collects CPU, memory, disk, network metrics |
| kube-state-metrics | Collects Kubernetes object state like pod status and deployment replicas |
| Alertmanager | Receives alerts from Prometheus and routes them to notification channels |
| Grafana | Visualizes all metrics through interactive dashboards |

### Layer 7: Application Monitoring

A ServiceMonitor resource in the monitoring namespace that tells Prometheus to scrape all Online Boutique services in the onlineboutique namespace. A custom Grafana dashboard with three panels showing the RED metrics for the application.

---

## 10.3 The Journey We Took

Let us trace back through the learning journey of this session.

We started with a blank slate and asked: what are we building and why?

We set up a command center on an EC2 instance and installed all our tools.

We created a real Kubernetes cluster on AWS in about 15 minutes using one command.

We learned what Helm is and why it exists: because managing raw YAML across environments is painful and error-prone.

We deployed a real 11-service microservice application and exposed it to the world using an Ingress Controller.

We asked the honest question: why can we not just monitor this like a monolith? And we discovered that dynamic, distributed systems need dynamic, distributed monitoring.

We learned how Prometheus works: the pull model, the components, the metric types, the labels, and PromQL.

We installed the complete Prometheus and Grafana stack with one Helm command and saw our cluster come alive with metrics.

We connected Prometheus to our application using a ServiceMonitor and wrote real queries against real application data.

We explored Grafana dashboards, learned the RED method, and understood how to use these tools under pressure during a real incident.

---

## 10.4 Key Concepts to Remember Forever

If you remember nothing else from today, remember these five things.

### 1: Kubernetes Runs Your Containers, It Does Not Monitor Them

Kubernetes is an orchestrator. It schedules, restarts, and scales your containers. But it has no built-in ability to show you what is happening inside them over time. That is the job of Prometheus and Grafana.

### 2: The Pull Model is the Right Model for Dynamic Systems

Prometheus goes to your applications and pulls metrics on a schedule. Applications just need to expose a /metrics endpoint. This works perfectly with Kubernetes because Prometheus can discover and drop targets automatically as pods come and go.

### 3: ServiceMonitor is the Bridge Between Your App and Prometheus

Without a ServiceMonitor, Prometheus does not know your application exists. With a ServiceMonitor, Prometheus automatically discovers every pod behind a Kubernetes Service and starts collecting metrics from them. No config file editing. No restarts.

### 4: Labels are the Superpower of Prometheus

Every metric in Prometheus has labels. Labels let you filter, group, and aggregate metrics in any dimension. By namespace, by pod, by node, by status code, by version. This is what makes PromQL so powerful.

### 5: The RED Method is Your Incident Playbook

Rate, Errors, Duration. For any service that is misbehaving, check these three things first. They will point you in the right direction every time.

---

## 10.5 What You Should Do This Week

Do not let the learning stop here. Here is a practical plan for the next 7 days.

### Day 1 and 2: Repeat What You Built Today

Go through every module again on your own without looking at the notes. If you can rebuild the entire setup from memory, you truly own the knowledge.

### Day 3: Explore PromQL Deeper

Spend an hour in the Prometheus UI writing queries. Try to answer questions like:

- Which pod in the cluster is using the most memory right now?
- Which node has the highest CPU usage over the last 6 hours?
- How many times has the cartservice pod restarted in the last 24 hours?
- What is the 99th percentile latency of the frontend service?

### Day 4: Build More Grafana Dashboards

Import three more community dashboards from grafana.com. Try dashboard IDs 1860, 3119, and 6417. Explore what each one shows and understand which metrics power each panel.

### Day 5: Set Up an Alert

Write a PrometheusRule that fires when the error rate of the frontend service exceeds 1 percent for more than 2 minutes. Configure Alertmanager to send that alert to a Slack channel or email. This is how you build real production alerting.

### Day 6: Read About What Comes Next

Read about the three topics listed in section 10.6. Even a basic understanding of these will make you significantly more effective in a production Kubernetes environment.

### Day 7: Teach Someone Else

The best way to solidify learning is to teach. Explain what you built to a colleague, a friend, or write a blog post about it. If you can explain it simply, you understand it deeply.

---

## 10.6 What Comes Next — The Roadmap Beyond This Workshop

This workshop covered metrics monitoring. But full observability in Kubernetes requires more. Here is the complete roadmap of what to learn next.

### Next Step 1: AlertManager Deep Dive

We installed Alertmanager today but barely touched it. In production, Alertmanager is critical. You need to learn:

- How to write PrometheusRule resources to define alerting conditions
- How to configure routing rules so different alerts go to different teams
- How to set up receivers for Slack, email, and PagerDuty
- How to configure silences during maintenance windows
- How to group related alerts to avoid notification flooding

### Next Step 2: Centralized Logging with Loki and Grafana

Prometheus handles metrics. But when an alert fires and you want to know WHY, you need logs. The modern stack for Kubernetes logging is:

| Component | Role |
|-----------|------|
| Promtail | Collects logs from every pod and node and ships them to Loki |
| Loki | Stores logs and indexes them by labels, not by full text |
| Grafana | Already connected — just add Loki as a second data source |

The powerful thing about Loki is that it uses the same label system as Prometheus. When you are looking at a Grafana dashboard and you see a spike in errors, you can click directly through to the Loki logs for that exact pod at that exact time. Metrics and logs in the same tool.

### Next Step 3: Distributed Tracing with Tempo or Jaeger

Logs tell you what happened. Traces tell you where in the request chain it happened.

When a user clicks checkout and it takes 8 seconds, a trace shows you:

- 10ms in frontend
- 20ms in checkoutservice
- 7800ms in paymentservice  -- this is where the problem is
- 10ms in emailservice

Without traces, finding that 7800ms in paymentservice would require guesswork. With traces, it takes 30 seconds.

Tools to learn: Grafana Tempo (integrates natively with Grafana), Jaeger (open source, widely used).

### Next Step 4: Long Term Storage with Thanos or Cortex

Prometheus stores data locally. By default it keeps 15 days. For a production environment, you need months or years of metrics for capacity planning and trend analysis.

Thanos extends Prometheus with:

- Object storage backend (S3, GCS) for unlimited retention
- Global query view across multiple Prometheus instances
- High availability with multiple Prometheus replicas
- Downsampling of old data to save storage costs

### Next Step 5: Prometheus Adapter and Horizontal Pod Autoscaling

Kubernetes can automatically scale your pods up and down based on CPU and memory using the built-in Horizontal Pod Autoscaler. But you can make it smarter by using your own custom metrics from Prometheus.

For example, scale the frontend pods based on the number of active user sessions, not just CPU. Or scale the checkoutservice based on the number of pending orders in the queue.

The Prometheus Adapter connects Prometheus metrics to the Kubernetes custom metrics API so the HPA can use them for scaling decisions.

### Next Step 6: GitOps for Monitoring Configuration

In a production team, monitoring configuration should be managed exactly like application code. Every ServiceMonitor, PrometheusRule, and Grafana dashboard should live in a Git repository and be deployed automatically using a GitOps tool like ArgoCD or FluxCD.

This means:

- No one manually runs kubectl apply for monitoring resources
- All changes go through a pull request with review
- The cluster state always matches what is in the Git repository
- Rolling back a bad alert rule is as easy as reverting a Git commit

---

## 10.7 The Full Observability Stack — The Complete Picture

Here is how all the tools fit together in a mature production Kubernetes environment:

| Layer | Tool | What It Collects |
|-------|------|-----------------|
| Metrics | Prometheus | Numbers over time from all pods, nodes, and services |
| Metrics Visualization | Grafana | Dashboards and graphs powered by Prometheus |
| Alerting | Alertmanager | Notification routing for metric-based alerts |
| Logs | Loki plus Promtail | Text logs from every pod and node |
| Log Visualization | Grafana | Same tool, second data source |
| Traces | Grafana Tempo or Jaeger | Request journey across microservices |
| Long Term Storage | Thanos or Cortex | Months and years of metric retention on S3 |
| Custom Autoscaling | Prometheus Adapter | Scale pods based on custom business metrics |

The beautiful thing about this stack is that Grafana ties everything together. One tool for metrics, logs, and traces. One place for your entire team to look when something goes wrong.

---

## 10.8 Before You Leave — Clean Up Your AWS Resources

This is very important. The resources we created today will cost money if left running.

Run the following commands on your EC2 Jump Box to clean up in order:

### Step 1: Delete the Online Boutique Application

```bash
cd microservices-demo
kubectl delete -f ./release/kubernetes-manifests.yaml -n onlineboutique
kubectl delete namespace onlineboutique
```

### Step 2: Uninstall Prometheus Stack

```bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

### Step 3: Uninstall NGINX Ingress Controller

```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```

### Step 4: Delete the EKS Cluster

```bash
cd ~
eksctl delete cluster -f eks-cluster.yaml
```

This will take about 15 minutes. Wait for it to complete fully. Do not close the terminal while it runs.

### Step 5: Terminate the EC2 Jump Box

Go to the AWS EC2 console. Find your eks-jumpbox instance. Select it. Click Instance State and then Terminate.

### Step 6: Verify in AWS Console

After everything is deleted, go to:

- EC2: confirm no running instances except what you had before
- EKS: confirm no clusters
- CloudFormation: confirm no stacks starting with eksctl
- Load Balancers: confirm no load balancers were left behind
- VPC: confirm the VPC created by eksctl was deleted

If any Load Balancers remain, delete them manually from the EC2 console. Occasionally eksctl does not clean these up if they were created by Kubernetes services.

---

## 10.9 Resources to Continue Learning

These are the best resources to continue your learning after this workshop.

| Resource | What It Covers | URL |
|----------|---------------|-----|
| Prometheus official docs | Complete PromQL reference, configuration, alerting | prometheus.io/docs |
| Grafana official docs | Dashboard creation, data sources, alerting | grafana.com/docs |
| Artifact Hub | Find Helm charts for any tool | artifacthub.io |
| kube-prometheus-stack | The Helm chart we used today | github.com/prometheus-community/helm-charts |
| Kubernetes official docs | Everything about Kubernetes | kubernetes.io/docs |
| eksctl docs | Creating and managing EKS clusters | eksctl.io |
| Grafana dashboards | Community dashboards to import | grafana.com/grafana/dashboards |
| CNCF landscape | Map of all cloud native tools | landscape.cncf.io |

---

## 10.10 Final Q and A

This is the time to ask any question that was not covered or not fully clear during the session.

Some questions to prompt discussion if the room is quiet:

- What would you monitor first in your own project?
- Which metric type would you use to track the number of active database connections?
- If the frontend is showing errors, what is the first PromQL query you would run?
- What is the difference between a ServiceMonitor and a PodMonitor?
- Why did we use NLB instead of ALB for the NGINX Ingress Controller?
- What happens to metrics data if the Prometheus pod restarts?
- How would you add a new microservice to the monitoring setup after today?

---

## Thank You

You came in today knowing nothing about Prometheus, Helm, or EKS monitoring. You are leaving with a working production-grade monitoring stack and the knowledge to build it again from scratch.

The most important thing now is to keep practicing. Break things on purpose. Delete pods and watch Grafana respond. Send artificial load and watch the metrics spike. Write an alert rule that fires and fix the condition that caused it.

The difference between someone who took a workshop and someone who truly knows this technology is practice. Keep building.

---
