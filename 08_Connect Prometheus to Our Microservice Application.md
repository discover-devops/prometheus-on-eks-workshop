# Module 8: Connect Prometheus to Our Microservice Application

---

## 8.1 Where We Are Right Now

At the end of Module 7, Prometheus is running and already collecting:

- Node level metrics from Node Exporter
- Kubernetes object metrics from kube-state-metrics
- Container metrics from cAdvisor

But if you go to the Prometheus UI right now and search for metrics from our Online Boutique application, you will find nothing. Prometheus has no idea our application exists. It does not know it should scrape it.

This module fixes that. We will tell Prometheus to discover and scrape metrics from our Online Boutique services using a Kubernetes resource called a ServiceMonitor.

---

## 8.2 Do the Online Boutique Services Expose Metrics?

Before we configure Prometheus to scrape our application, we need to confirm that the application actually exposes metrics in the Prometheus format.

This is a great question to ask in any real project. Not every application exposes Prometheus metrics by default. Some need extra libraries added to the code. Some need exporters. Some expose metrics on a non-standard path or port.

Let us check the Online Boutique frontend service directly.

First find the name of the frontend pod:

```bash
kubectl get pods -n onlineboutique -l app=frontend
```

Copy the pod name and run:

```bash
kubectl port-forward pod/YOUR-FRONTEND-POD-NAME 8080:8080 -n onlineboutique &
```

Now curl the metrics endpoint:

```bash
curl http://localhost:8080/metrics | head -40
```

You should see output like this:

```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 4.5e-05
go_gc_duration_seconds{quantile="0.25"} 6.2e-05
# HELP http_requests_total Total number of HTTP requests made.
# TYPE http_requests_total counter
http_requests_total{code="200",method="get"} 1523
http_requests_total{code="500",method="get"} 3
# HELP request_duration_seconds Time spent serving HTTP requests.
# TYPE request_duration_seconds histogram
request_duration_seconds_bucket{le="0.005"} 1200
request_duration_seconds_bucket{le="0.01"}  1400
```

The Online Boutique services expose metrics natively at the /metrics path on port 8080. This is because they were built with Prometheus client libraries already included.

This is a great teaching moment for students. In a real project, if your application does not expose /metrics natively, you would need to either add the Prometheus client library to the code or use an exporter. We will discuss this in the FAQ section at the end of this module.

Stop the port-forward:

```bash
kill %1
```

---

## 8.3 How ServiceMonitor Works — The Full Picture

Before writing any YAML, let us understand exactly what a ServiceMonitor does and how all the pieces connect.

Here is the chain of connections:

ServiceMonitor selects a Kubernetes Service. The Service selects Pods. Prometheus scrapes the Pods.

Step by step:

Step 1: You create a ServiceMonitor YAML. Inside it, you write a label selector. This selector looks for Kubernetes Services that have matching labels.

Step 2: The Prometheus Operator watches for ServiceMonitor resources. When it finds one, it reads the selector and finds the matching Kubernetes Service.

Step 3: The Prometheus Operator generates a scrape configuration and gives it to Prometheus. Prometheus now knows which service to hit, which path to scrape, and how often to do it.

Step 4: Prometheus starts scraping all pods behind that Service and stores the metrics in its time series database.

Step 5: You can now query those metrics in the Prometheus UI and Grafana.

The important thing to understand is that the ServiceMonitor does NOT select pods directly. It selects a Kubernetes Service. Prometheus then discovers the pods behind that service automatically.

---

## 8.4 Step 1: Check the Labels on the Online Boutique Services

The ServiceMonitor uses label selectors to find the right Kubernetes Services. We need to know what labels our services have before writing the ServiceMonitor.

```bash
kubectl get svc -n onlineboutique --show-labels
```

You will see output like this:

```
NAME                    TYPE        CLUSTER-IP      PORT(S)     LABELS
frontend                ClusterIP   10.100.xx.xx    80/TCP      app=frontend
cartservice             ClusterIP   10.100.xx.xx    7070/TCP    app=cartservice
checkoutservice         ClusterIP   10.100.xx.xx    5050/TCP    app=checkoutservice
```

All services have an app label but each one has a different value. We need a common label that matches ALL services at once so we can monitor all of them with a single ServiceMonitor.

For this workshop we will add a new common label to all services and write one ServiceMonitor that matches that common label.

---

## 8.5 Step 2: Add a Common Label to All Online Boutique Services

We will add the label monitor: online-boutique to all services in the onlineboutique namespace.

```bash
kubectl label svc --all monitor=online-boutique -n onlineboutique
```

Verify the labels were added:

```bash
kubectl get svc -n onlineboutique --show-labels | grep monitor
```

You should see monitor=online-boutique on every service in the namespace.

---

## 8.6 Step 3: Check the Port Names on the Services

ServiceMonitor references ports by NAME, not by number. This is an important detail that trips up many beginners.

Let us check the port name on the frontend service:

```bash
kubectl get svc frontend -n onlineboutique -o yaml | grep -A5 "ports:"
```

You will see something like:

```yaml
ports:
- name: http
  port: 80
  targetPort: 8080
  protocol: TCP
```

The port name is http. This is the name we will use in the ServiceMonitor. The service listens on port 80 and forwards to port 8080 on the pod where the /metrics endpoint lives.

---

## 8.7 Step 4: Create the ServiceMonitor

Now we write the ServiceMonitor. Create a file called online-boutique-servicemonitor.yaml:

```bash
cat > online-boutique-servicemonitor.yaml <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: online-boutique-monitor
  namespace: monitoring
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - onlineboutique
  selector:
    matchLabels:
      monitor: online-boutique
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
EOF
```

Let us understand every single field in this file:

| Field | Value | What It Does |
|-------|-------|-------------|
| apiVersion | monitoring.coreos.com/v1 | Custom resource from the Prometheus Operator |
| kind | ServiceMonitor | The type of resource we are creating |
| name | online-boutique-monitor | The name of this ServiceMonitor |
| namespace | monitoring | ServiceMonitor lives in monitoring namespace where Prometheus runs |
| labels release | prometheus | Critical label. Prometheus Operator uses this to match ServiceMonitors to this Prometheus instance |
| namespaceSelector matchNames | onlineboutique | Look for services in the onlineboutique namespace |
| selector matchLabels | monitor: online-boutique | Find services with this label which we added in Step 2 |
| endpoints port | http | Scrape the port named http on the matching services |
| endpoints path | /metrics | The URL path where metrics are exposed |
| endpoints interval | 15s | Scrape every 15 seconds |

Apply the ServiceMonitor:

```bash
kubectl apply -f online-boutique-servicemonitor.yaml
```

Verify it was created:

```bash
kubectl get servicemonitor -n monitoring
```

You should see:

```
NAME                      AGE
online-boutique-monitor   10s
```

---

## 8.8 Step 5: Verify Prometheus is Scraping the Application

After applying the ServiceMonitor, give Prometheus about 30 seconds to pick up the new configuration.

Go to the Prometheus UI. Navigate to Status then Targets.

Scroll down and look for targets in the onlineboutique namespace. You should see entries like:

```
monitoring/online-boutique-monitor/0   onlineboutique/frontend:http     UP
monitoring/online-boutique-monitor/1   onlineboutique/cartservice:http  UP
```

All targets should show UP status. If you see them here, Prometheus is now successfully scraping our Online Boutique services.

If you do not see any onlineboutique targets after 1 minute, go to the troubleshooting section 8.11.

---

## 8.9 Step 6: Query Application Metrics in Prometheus

Go to the Prometheus UI Graph tab and try these queries one by one.

### Query 1: Total HTTP Requests to the Frontend

```
http_requests_total{namespace="onlineboutique"}
```

This shows the total number of HTTP requests received by each service since it started.

### Query 2: Request Rate Per Second

```
rate(http_requests_total{namespace="onlineboutique", job="frontend"}[5m])
```

This shows how many requests per second the frontend is receiving averaged over the last 5 minutes. Since the loadgenerator is running you should see a non-zero value.

### Query 3: Error Rate

```
rate(http_requests_total{namespace="onlineboutique", code=~"5.."}[5m])
```

This shows the rate of 5xx errors. The code=~"5.." uses a regex to match any status code starting with 5.

### Query 4: 95th Percentile Latency

```
histogram_quantile(0.95, rate(request_duration_seconds_bucket{namespace="onlineboutique"}[5m]))
```

This shows that 95 percent of requests are completing faster than this value in seconds. If this number spikes, users are experiencing slow page loads.

### Query 5: Request Rate Across All Services

```
sum(rate(http_requests_total{namespace="onlineboutique"}[5m])) by (job)
```

This shows the request rate for each service side by side. Useful to understand which services are the busiest.

Switch to the Graph view for each query and you will see live charts updating every few seconds as the loadgenerator continuously sends traffic.

---

## 8.10 Step 7: Create a Grafana Dashboard for the Application

Now let us visualize these metrics in Grafana.

Open Grafana using the LoadBalancer URL from Module 7. Log in with admin and admin123.

### Create a New Dashboard

Step 1: Click the plus icon on the left sidebar and select New Dashboard.

Step 2: Click Add visualization.

Step 3: Make sure Prometheus is selected as the data source.

### Panel 1: Request Rate

In the query box type:

```
sum(rate(http_requests_total{namespace="onlineboutique"}[5m])) by (job)
```

Set the visualization type to Time series. Title it Online Boutique Request Rate per Service. Click Apply.

### Panel 2: Error Rate

Click Add panel. In the query box type:

```
sum(rate(http_requests_total{namespace="onlineboutique", code=~"5.."}[5m])) by (job)
```

Set the visualization type to Time series. Title it Online Boutique Error Rate. Click Apply.

### Panel 3: 95th Percentile Latency

Click Add panel. In the query box type:

```
histogram_quantile(0.95, sum(rate(request_duration_seconds_bucket{namespace="onlineboutique"}[5m])) by (le, job))
```

Set the visualization type to Time series. Title it Online Boutique P95 Latency in Seconds. Click Apply.

### Save the Dashboard

Click the save icon at the top right. Name it Online Boutique Monitoring. Click Save.

You now have a custom monitoring dashboard for your microservice application.

---

## 8.11 Troubleshooting: ServiceMonitor Not Working

Work through these checks in order if targets are not showing up in Prometheus.

### Check 1: Verify the ServiceMonitor exists

```bash
kubectl get servicemonitor -n monitoring
```

If it is not listed, the apply failed. Run kubectl apply again and look for errors.

### Check 2: Verify the label on services

```bash
kubectl get svc -n onlineboutique --show-labels | grep monitor
```

If you do not see monitor=online-boutique, run the label command from Step 2 again.

### Check 3: Verify the release label on the ServiceMonitor

```bash
kubectl get servicemonitor online-boutique-monitor -n monitoring -o yaml | grep -A3 "labels:"
```

You must see release: prometheus in the labels. Without this the Prometheus Operator will ignore the ServiceMonitor.

### Check 4: Verify the Prometheus Operator setting

```bash
kubectl get prometheus -n monitoring -o yaml | grep serviceMonitorSelector -A5
```

If this shows a strict selector that does not match your ServiceMonitor, go back to prometheus-values.yaml, confirm serviceMonitorSelectorNilUsesHelmValues is set to false, and run helm upgrade.

### Check 5: Check the Prometheus Operator logs

```bash
kubectl logs -l app.kubernetes.io/name=prometheus-operator -n monitoring --tail=50
```

Look for any errors mentioning the ServiceMonitor name. The operator logs tell you exactly why a ServiceMonitor was rejected.

### Check 6: Test the metrics endpoint directly

```bash
kubectl port-forward svc/frontend 8080:80 -n onlineboutique &
curl http://localhost:8080/metrics | head -10
kill %1
```

If this returns metrics, the application is working and the problem is in the ServiceMonitor configuration. If this returns an error, the application is not exposing metrics correctly.

---

## 8.12 Common Questions from Students

### Question: Why does the ServiceMonitor live in the monitoring namespace but selects services in onlineboutique?

The ServiceMonitor must be in the same namespace as Prometheus. Prometheus only looks for ServiceMonitors in its own namespace by default. But the namespaceSelector field inside the ServiceMonitor tells it which namespace to look for the services in. So the ServiceMonitor lives in monitoring but points at onlineboutique.

### Question: What is the difference between ServiceMonitor and PodMonitor?

ServiceMonitor selects a Kubernetes Service and discovers pods through it. PodMonitor selects pods directly using pod labels without going through a Service. Use ServiceMonitor when your application has a Kubernetes Service defined which is almost always the case. Use PodMonitor only when you need to scrape pods that do not have a Service such as sidecar containers or batch job pods.

### Question: What if my application does not expose a /metrics endpoint?

You have two options. The first option is to add a Prometheus client library to your application code. Libraries exist for Go, Python, Java, Node.js, Ruby and most other languages. The library adds a /metrics endpoint automatically. The second option is to use a Prometheus exporter. An exporter runs alongside your application, queries it using its native protocol, and translates the output into Prometheus format. For example the MySQL exporter queries MySQL and exposes its metrics in Prometheus format.

### Question: Can I change the scrape interval for different services?

Yes. You can set different intervals for different services by creating separate ServiceMonitors. The 15 second interval is a good default. Very high frequency scraping of many services can put load on both the application and Prometheus.

---

## Summary of Module 8

By the end of this module you have done the following:

- Confirmed that Online Boutique services expose metrics natively at the /metrics endpoint
- Understood exactly how a ServiceMonitor connects to a Kubernetes Service and how Prometheus discovers it
- Added a common label to all Online Boutique services
- Written and applied a ServiceMonitor that tells Prometheus to scrape all Online Boutique services
- Verified the targets are showing UP in the Prometheus UI
- Run five meaningful PromQL queries against the application metrics
- Built a custom Grafana dashboard with three panels showing request rate, error rate, and latency

In the next module we will explore Grafana dashboards more deeply and understand how to use the built-in Kubernetes dashboards for day to day cluster monitoring.

---
