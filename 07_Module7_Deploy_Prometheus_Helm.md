# Module 7: Deploy Prometheus on Kubernetes Using Helm

---

## 7.1 What is kube-prometheus-stack?

In Module 6 we learned that Prometheus is an ecosystem of multiple components. If you were to install each component separately, you would need to:

- Install Prometheus Server
- Install Alertmanager
- Install Node Exporter as a DaemonSet on every node
- Install kube-state-metrics
- Configure all of them to talk to each other
- Configure Grafana and connect it to Prometheus
- Create all the Kubernetes RBAC roles and permissions
- Import dashboards into Grafana manually

This would take hours and is very easy to get wrong.

The kube-prometheus-stack is a single Helm chart that does ALL of the above in one command. It is maintained by the Prometheus Community and is the standard way to deploy Prometheus on Kubernetes in production.

Here is exactly what gets installed when you run this chart:

| Component | What It Does |
|-----------|-------------|
| Prometheus Operator | Watches for custom resources like ServiceMonitor and configures Prometheus automatically |
| Prometheus Server | Collects and stores metrics from all targets |
| Alertmanager | Receives alerts from Prometheus and routes notifications |
| Node Exporter | DaemonSet that runs on every node and collects CPU, memory, disk metrics |
| kube-state-metrics | Collects Kubernetes object state metrics like pod status, deployment replicas |
| Grafana | Visualization dashboards, pre-connected to Prometheus |

All of this from one helm install command.

---

## 7.2 What is the Prometheus Operator?

This is a new concept that we need to understand before installing.

In plain Kubernetes, Prometheus would need a config file called prometheus.yml that lists every target it should scrape. Every time you add a new service, you would have to edit this config file and restart Prometheus. This does not work in a dynamic Kubernetes environment where pods come and go constantly.

The Prometheus Operator solves this by introducing custom Kubernetes resources:

- ServiceMonitor: tells Prometheus to scrape a Kubernetes Service
- PodMonitor: tells Prometheus to scrape pods directly
- PrometheusRule: defines alerting and recording rules

Instead of editing a config file, you create a ServiceMonitor YAML and apply it with kubectl. The Prometheus Operator watches for these resources and automatically updates the Prometheus configuration. No restarts needed.

Think of the Prometheus Operator as the brain that keeps Prometheus configuration in sync with what is running in your cluster.

---

## 7.3 Step 1: Add the Prometheus Community Helm Repository

Run these commands on your EC2 Jump Box:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Verify the repo was added:

```bash
helm repo list
```

You should see:

```
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
ingress-nginx           https://kubernetes.github.io/ingress-nginx
```

Search to confirm the chart is available:

```bash
helm search repo kube-prometheus-stack
```

You should see the chart listed with its current version.

---

## 7.4 Step 2: Create a Custom Values File

The kube-prometheus-stack chart has thousands of configuration options. We will override only the ones that matter for our workshop. Everything else uses sensible defaults.

Create a file called prometheus-values.yaml:

```bash
cat > prometheus-values.yaml <<EOF
prometheus:
  prometheusSpec:
    retention: 7d
    resources:
      requests:
        memory: 512Mi
        cpu: 250m
      limits:
        memory: 1Gi
        cpu: 500m
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false

grafana:
  enabled: true
  adminPassword: "admin123"
  service:
    type: LoadBalancer

alertmanager:
  enabled: true

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
EOF
```

Let us understand the important settings:

| Setting | What It Does |
|---------|-------------|
| retention: 7d | Prometheus keeps 7 days of metrics data. Default is 15 days but we use 7 to save disk space |
| resources | CPU and memory limits for the Prometheus pod. Important on t3.medium nodes |
| serviceMonitorSelectorNilUsesHelmValues: false | This is critical. It tells Prometheus to discover ALL ServiceMonitors in the cluster, not just the ones created by this Helm chart. Without this, Prometheus will not find our custom ServiceMonitor in Module 8 |
| grafana adminPassword | The password to log into Grafana. Change this in production |
| grafana service type LoadBalancer | Creates an AWS Load Balancer so we can access Grafana from our browser directly |

---

## 7.5 Step 3: Create the Monitoring Namespace

```bash
kubectl create namespace monitoring
```

We always install monitoring tools in a separate namespace. This keeps them isolated from application workloads.

---

## 7.6 Step 4: Install the kube-prometheus-stack

```bash
helm install prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml \
  --wait \
  --timeout 10m
```

Let us understand each flag:

| Flag | What It Does |
|------|-------------|
| prometheus | This is the release name. All resources will be prefixed with this name |
| prometheus-community/kube-prometheus-stack | The chart from the repository |
| --namespace monitoring | Install into the monitoring namespace |
| --values prometheus-values.yaml | Use our custom values file |
| --wait | Hold the terminal until all pods are running |
| --timeout 10m | Wait up to 10 minutes before timing out |

This command will take 3 to 5 minutes. You will see a success message like this when it finishes:

```
NAME: prometheus
LAST DEPLOYED: <date>
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
```

---

## 7.7 Step 5: Verify the Installation

### Check All Pods Are Running

```bash
kubectl get pods -n monitoring
```

You should see all pods in Running status:

```
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          3m
prometheus-grafana-xxxxxxxxxx-xxxxx                      3/3     Running   0          3m
prometheus-kube-prometheus-operator-xxxxxxxxxx-xxxxx     1/1     Running   0          3m
prometheus-kube-state-metrics-xxxxxxxxxx-xxxxx           1/1     Running   0          3m
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          3m
prometheus-prometheus-node-exporter-xxxxx                1/1     Running   0          3m
prometheus-prometheus-node-exporter-xxxxx                1/1     Running   0          3m
```

Note that Node Exporter shows two pods. This is because we have 2 worker nodes and Node Exporter runs as a DaemonSet, meaning one pod per node.

### Check All Services

```bash
kubectl get svc -n monitoring
```

You will see several services. The two important ones are:

```
NAME                                      TYPE           CLUSTER-IP       EXTERNAL-IP
prometheus-grafana                        LoadBalancer   10.100.xx.xx     xxxxxx.elb.amazonaws.com
prometheus-kube-prometheus-prometheus     ClusterIP      10.100.xx.xx     <none>
prometheus-kube-prometheus-alertmanager   ClusterIP      10.100.xx.xx     <none>
```

Grafana has a LoadBalancer service because we set that in our values file. Prometheus and Alertmanager are ClusterIP because we will access them via port-forwarding.

### Check the Helm Release

```bash
helm list -n monitoring
```

You should see:

```
NAME        NAMESPACE   REVISION   STATUS     CHART
prometheus  monitoring  1          deployed   kube-prometheus-stack-xx.x.x
```

---

## 7.8 Step 6: Access the Prometheus UI

We will use kubectl port-forwarding to access the Prometheus web UI on our local machine. Port-forwarding creates a secure tunnel from your laptop to the pod inside the cluster.

Run this command on your EC2 Jump Box:

```bash
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090 &
```

The ampersand at the end runs it in the background so you can keep using the terminal.

However, since you are on a Windows laptop connecting via PuTTY to the EC2 Jump Box, you need one more step. You need to forward that port from your EC2 machine to your laptop using PuTTY SSH tunneling.

### Setting Up PuTTY SSH Tunnel

Step 1: Open your saved PuTTY session settings. Do not connect yet.

Step 2: On the left panel, go to Connection, then SSH, then Tunnels.

Step 3: Fill in these values:

| Field | Value |
|-------|-------|
| Source port | 9090 |
| Destination | localhost:9090 |
| Type | Local |

Step 4: Click Add, then click Open to connect.

Step 5: Now open your browser on your Windows laptop and go to:

```
http://localhost:9090
```

You should see the Prometheus web UI.

---

## 7.9 Exploring the Prometheus UI

### The Targets Page

The most important page in Prometheus UI for a beginner is the Targets page.

Go to: Status, then Targets

This page shows every endpoint that Prometheus is currently scraping. You will see targets for:

- Kubernetes API server
- kubelet on each node
- Node Exporter on each node
- kube-state-metrics
- Alertmanager
- Prometheus itself

All targets should show State as UP. If any target shows DOWN, Prometheus cannot reach that endpoint and metrics are not being collected from it.

### The Service Discovery Page

Go to: Status, then Service Discovery

This page shows how Prometheus discovered its targets. You will see that it is using Kubernetes service discovery, finding pods and services automatically through the Kubernetes API.

### Running Your First Query

Click on the Graph tab at the top.

In the query box, type this and click Execute:

```
up
```

This is the simplest possible query. It returns 1 for every target that Prometheus can successfully scrape and 0 for any target that is down. You should see a long list of results, all showing value 1.

Now try this query to see CPU usage across all nodes:

```
node_cpu_seconds_total
```

You will see raw counter values. Now try the rate function:

```
rate(node_cpu_seconds_total[5m])
```

This shows the CPU usage rate per second over the last 5 minutes. Switch to the Graph tab and you will see a live chart.

Try this query to see memory available on each node:

```
node_memory_MemAvailable_bytes
```

To see it in gigabytes, divide by 1024 three times:

```
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

---

## 7.10 Step 7: Access Grafana

### Get the Grafana Load Balancer URL

```bash
kubectl get svc prometheus-grafana -n monitoring
```

Copy the EXTERNAL-IP value. Open your browser and go to:

```
http://YOUR-GRAFANA-LOAD-BALANCER-DNS
```

### Log Into Grafana

| Field | Value |
|-------|-------|
| Username | admin |
| Password | admin123 |

This is the password we set in the prometheus-values.yaml file.

### Exploring the Built-In Dashboards

Grafana comes pre-loaded with dozens of dashboards for Kubernetes. To find them:

Step 1: Click the four-square grid icon on the left sidebar (Dashboards).

Step 2: Click Browse.

Step 3: You will see a folder called Default or Kubernetes. Open it.

Some key dashboards to explore with students:

| Dashboard Name | What It Shows |
|---------------|--------------|
| Kubernetes / Compute Resources / Cluster | Overall cluster CPU and memory usage |
| Kubernetes / Compute Resources / Namespace | Resource usage broken down by namespace |
| Kubernetes / Compute Resources / Pod | CPU and memory of individual pods |
| Node Exporter / Nodes | Detailed metrics for each worker node including disk and network |
| Kubernetes / Networking / Cluster | Network traffic in and out of the cluster |

Open the Kubernetes / Compute Resources / Namespace dashboard. You will see metrics for three namespaces: monitoring, onlineboutique, and ingress-nginx. Click on onlineboutique to see the resource usage of our microservice application.

---

## 7.11 What is Being Monitored Right Now?

Let us take stock of what Prometheus is already collecting without us doing anything extra.

### Cluster Level Metrics (from kube-state-metrics)

- Number of pods running, pending, or failed in each namespace
- Number of available replicas for each deployment
- Node status and conditions
- Resource requests and limits for each container
- Job completion status

### Node Level Metrics (from Node Exporter)

- CPU usage per core per node
- Memory usage, free memory, cached memory
- Disk read and write throughput
- Disk space used and available
- Network bytes in and out per interface
- System load average

### Container Level Metrics (from cAdvisor built into kubelet)

- CPU usage per container
- Memory usage per container
- Container restart count
- Container filesystem usage

### Kubernetes API Server Metrics

- API request rate and latency
- API errors by verb and resource type
- etcd latency

All of this is available automatically the moment you install the kube-prometheus-stack. Zero extra configuration needed.

---

## 7.12 Common Questions from Students

### Question: I see some targets as DOWN in the Prometheus UI. Is this a problem?

Not always. Some targets may be marked as DOWN if Prometheus cannot reach the Kubernetes control plane components directly. This is normal on managed Kubernetes clusters like EKS because AWS controls the control plane and some internal endpoints are not directly accessible. Your application metrics will still work fine.

### Question: The Grafana LoadBalancer EXTERNAL-IP is still pending after 5 minutes.

Check if there are any errors on the service:

```bash
kubectl describe svc prometheus-grafana -n monitoring
```

Look at the Events section. If you see errors about IAM permissions, your EKS worker node IAM role may be missing the EC2 permissions needed to create a Load Balancer.

### Question: Can I change the Grafana password later?

Yes. Update your prometheus-values.yaml with the new password and run:

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml
```

### Question: How do I completely remove the monitoring stack?

```bash
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

This removes all pods, services, and configmaps created by the chart.

### Question: Why are there 2 containers in some pods (READY shows 2/2)?

The Prometheus pod shows 2/2 and the Alertmanager pod shows 2/2. This is because each of these pods runs two containers:

- The main application container (Prometheus or Alertmanager)
- A config-reloader sidecar container that watches for configuration changes and reloads the main container automatically without a restart

---

## Summary of Module 7

By the end of this module you have done the following:

- Understood what kube-prometheus-stack includes and why it is the standard way to deploy Prometheus
- Understood what the Prometheus Operator is and why it replaces manual config file editing
- Created a custom values file for our EKS cluster
- Installed the full monitoring stack with one Helm command
- Verified all pods and services are running
- Accessed the Prometheus UI and ran your first PromQL queries
- Accessed Grafana and explored the built-in Kubernetes dashboards
- Understood what metrics are being collected automatically right now

In the next module we will connect Prometheus to our Online Boutique microservice application using a ServiceMonitor so we can see application-level metrics in addition to the infrastructure metrics we already have.

---
