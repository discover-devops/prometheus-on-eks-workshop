# Module 4: Deploy a Microservice Application on EKS

---

## 4.1 What is a Microservice Architecture?

Before we deploy anything, let us take 5 minutes to understand what we are deploying and why it is different from a traditional application.

### Monolithic Application

In a monolithic application, everything lives in one big codebase. The user interface, the business logic, the database layer, and the payment processing are all part of one deployable unit. You build it once and deploy it as a single application.

This works fine when the application is small. But as the application grows, problems appear:

- One bug in the payment module can bring down the entire application
- You cannot scale just the checkout feature during a sale. You have to scale the entire application
- If 10 teams are working on the same codebase, deployments become risky and slow
- Updating one part requires redeploying and retesting everything

### Microservice Application

In a microservice application, the application is broken into small, independent services. Each service does one thing and does it well. Each service has its own codebase, its own deployment, and its own scaling.

In our Online Boutique application, instead of one big application, we have 11 separate services:

| Service | Language | What It Does |
|---------|----------|-------------|
| frontend | Go | Serves the website to users |
| cartservice | C# | Manages the shopping cart using Redis |
| productcatalogservice | Go | Returns the list of products |
| currencyservice | Node.js | Converts prices to different currencies |
| paymentservice | Node.js | Processes payment information |
| shippingservice | Go | Calculates shipping costs |
| emailservice | Python | Sends order confirmation emails |
| checkoutservice | Go | Coordinates the checkout process |
| recommendationservice | Python | Suggests related products |
| adservice | Java | Serves contextual advertisements |
| loadgenerator | Python | Simulates real user traffic automatically |

Each of these runs as a separate pod in Kubernetes. They talk to each other over gRPC, which is a high-performance communication protocol.

### Why This Makes Monitoring Complex

With a monolithic application, you have one process to monitor. One CPU, one memory, one log file.

With 11 microservices, you now have 11 separate processes. Any one of them can fail independently. A problem in the currencyservice might slow down the frontend, which looks like a frontend bug. Without proper monitoring, finding the root cause becomes very difficult.

This is exactly why we need a tool like Prometheus. We will come back to this in Module 5.

---

## 4.2 What is Ingress and Why Are We Using It?

Before we deploy the application, let us understand how we are going to expose it to the world.

### The Problem Without Ingress

In Kubernetes, there are three ways to expose a service to external traffic.

The first way is ClusterIP. This only works inside the cluster. External users cannot access it.

The second way is NodePort. This opens a port on every worker node. External users can access it using the node IP and port number. This is not ideal because ports are limited and node IPs can change.

The third way is LoadBalancer. This creates a separate AWS Load Balancer for each service. If you have 11 services, you get 11 load balancers. Each load balancer costs money. This is expensive and unmanageable.

### The Solution: Ingress

Ingress gives you one single entry point for all external traffic. You deploy one Load Balancer for the Ingress Controller and write routing rules to direct traffic to the correct service based on the path or hostname.

Here is how it looks:

The user opens a browser and visits a URL. The request hits the AWS Network Load Balancer. The Load Balancer forwards the request to the NGINX Ingress Controller running inside the cluster. The Ingress Controller reads the routing rules and forwards the request to the correct Kubernetes service. The service sends it to the correct pod.

One Load Balancer. One public IP. All traffic routed correctly. This is Ingress.

### NGINX Ingress Controller

The Ingress Controller is the component that actually implements the routing rules. We are using the NGINX Ingress Controller, which is the most widely used and well supported option. We will install it using Helm.

---

## 4.3 Step 1: Create a Namespace for the Application

It is a good practice to deploy applications in their own namespace. This keeps resources organized and separated from system components.

```bash
kubectl create namespace onlineboutique
```

Verify it was created:

```bash
kubectl get namespaces
```

You should see onlineboutique listed along with the default namespaces.

---

## 4.4 Step 2: Install the NGINX Ingress Controller Using Helm

We will install the NGINX Ingress Controller first. The application will be deployed after this, and the Ingress will route traffic to it.

### Add the Ingress-NGINX Helm Repository

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### Install the NGINX Ingress Controller

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb"
```

Let us understand what each flag does:

| Flag | What It Does |
|------|-------------|
| upgrade --install | Install if not present, upgrade if already installed |
| --namespace ingress-nginx | Install into the ingress-nginx namespace |
| --create-namespace | Create the namespace if it does not exist |
| controller.service.type=LoadBalancer | Tell Kubernetes to create an AWS Load Balancer |
| aws-load-balancer-type=nlb | Use Network Load Balancer instead of Classic Load Balancer |

This command will take about 2 to 3 minutes.

### Verify the Ingress Controller is Running

```bash
kubectl get pods -n ingress-nginx
```

You should see the ingress-nginx-controller pod in Running status:

```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxxxxxxx-xxxxx    1/1     Running   0          2m
```

### Get the Load Balancer DNS Name

```bash
kubectl get svc -n ingress-nginx
```

You will see output like this:

```
NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                                    PORT(S)
ingress-nginx-controller   LoadBalancer   10.100.xx.xx    xxxxxx.elb.ap-south-1.amazonaws.com            80:xxxxx/TCP,443:xxxxx/TCP
```

Copy the value under EXTERNAL-IP. This is the DNS name of your AWS Network Load Balancer. Students will use this DNS name to access the application in their browser. Save this value, you will need it shortly.

Note: It may take 2 to 3 minutes for the EXTERNAL-IP to appear. If you see pending, wait and run the command again.

---

## 4.5 Step 3: Deploy the Online Boutique Application

We will deploy the Online Boutique application using its Kubernetes manifest file. Google maintains this file and it works on any Kubernetes cluster including EKS.

### Clone the Repository

```bash
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo
```

### Deploy the Application

```bash
kubectl apply -f ./release/kubernetes-manifests.yaml -n onlineboutique
```

This command reads the manifest file and creates all 11 deployments and their services inside the onlineboutique namespace.

### Watch the Pods Come Up

```bash
kubectl get pods -n onlineboutique --watch
```

You will see the pods starting one by one. Wait until all pods show STATUS as Running. This takes about 3 to 5 minutes.

Press Ctrl+C to stop watching once all pods are running.

The final output should look like this:

```
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-xxxxxxxxxx-xxxxx               1/1     Running   0          4m
cartservice-xxxxxxxxxx-xxxxx             1/1     Running   0          4m
checkoutservice-xxxxxxxxxx-xxxxx         1/1     Running   0          4m
currencyservice-xxxxxxxxxx-xxxxx         1/1     Running   0          4m
emailservice-xxxxxxxxxx-xxxxx            1/1     Running   0          4m
frontend-xxxxxxxxxx-xxxxx                1/1     Running   0          4m
loadgenerator-xxxxxxxxxx-xxxxx           1/1     Running   0          4m
paymentservice-xxxxxxxxxx-xxxxx          1/1     Running   0          4m
productcatalogservice-xxxxxxxxxx-xxxxx   1/1     Running   0          4m
recommendationservice-xxxxxxxxxx-xxxxx   1/1     Running   0          4m
shippingservice-xxxxxxxxxx-xxxxx         1/1     Running   0          4m
```

### Verify the Services

```bash
kubectl get svc -n onlineboutique
```

You will see all 11 services. Notice that all of them are of type ClusterIP. This means none of them are directly exposed to the internet. Traffic will reach them only through the Ingress Controller.

---

## 4.6 Step 4: Create the Ingress Resource

Now we need to tell the NGINX Ingress Controller how to route traffic to our application. We do this by creating an Ingress resource.

### Create the Ingress YAML File

```bash
cat > online-boutique-ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: online-boutique-ingress
  namespace: onlineboutique
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
EOF
```

Let us understand what this file does:

| Field | What It Does |
|-------|-------------|
| kind: Ingress | This is an Ingress resource |
| namespace: onlineboutique | It belongs to the onlineboutique namespace |
| kubernetes.io/ingress.class: nginx | Use the NGINX Ingress Controller to handle this |
| path: / | Match all requests starting from the root path |
| service name: frontend | Send matched traffic to the frontend service |
| port: 80 | Send it to port 80 of the frontend service |

### Apply the Ingress Resource

```bash
kubectl apply -f online-boutique-ingress.yaml
```

### Verify the Ingress Was Created

```bash
kubectl get ingress -n onlineboutique
```

You should see:

```
NAME                      CLASS    HOSTS   ADDRESS   PORTS   AGE
online-boutique-ingress   nginx    *                 80      30s
```

---

## 4.7 Step 5: Access the Application

Now open your web browser on your Windows laptop and go to the following URL. Replace the value with your actual Load Balancer DNS name that you copied in Step 2.

```
http://YOUR-LOAD-BALANCER-DNS-NAME
```

For example:

```
http://xxxxxx.elb.ap-south-1.amazonaws.com
```

You should see the Online Boutique homepage. You can browse products, add them to the cart, and go through the checkout flow.

The loadgenerator service is also running inside the cluster. It automatically sends traffic to the application to simulate real users. This means even if no one visits the site manually, Prometheus will still see traffic and metrics.

---

## 4.8 Understanding What We Just Deployed

Let us take a moment to look at the architecture of what is now running in our cluster.

### Check All Resources in the Namespace

```bash
kubectl get all -n onlineboutique
```

This shows all pods, services, deployments, and replicasets in the onlineboutique namespace at once.

### Look at a Specific Service

```bash
kubectl describe svc frontend -n onlineboutique
```

This shows you the details of the frontend service, including its port, selector, and endpoints.

### Look at a Specific Pod

```bash
kubectl describe pod -l app=frontend -n onlineboutique
```

This shows you which node the frontend pod is running on, what image it is using, and its resource requests and limits.

### Check Pod Logs

```bash
kubectl logs -l app=frontend -n onlineboutique
```

This shows the live logs from the frontend pod. Every time a user visits the page, you will see log entries here.

---

## 4.9 Common Questions from Students

### Question: Why are all services ClusterIP and not LoadBalancer?

Because we have the Ingress Controller in front. All external traffic comes in through the one Load Balancer that the Ingress Controller created. The individual services only need to be reachable inside the cluster, so ClusterIP is correct and sufficient.

### Question: The EXTERNAL-IP is still showing as pending after 5 minutes. What do I do?

This usually means there is a permission issue and AWS could not create the Load Balancer. Check the following. First make sure your EKS worker node IAM role has the AmazonEC2FullAccess policy. Second, check the events on the service:

```bash
kubectl describe svc ingress-nginx-controller -n ingress-nginx
```

Look at the Events section at the bottom. It will tell you exactly why the Load Balancer could not be created.

### Question: Some pods are in CrashLoopBackOff status. What does that mean?

CrashLoopBackOff means the pod is starting, crashing, and Kubernetes is trying to restart it repeatedly. Check the logs of the crashing pod:

```bash
kubectl logs POD-NAME -n onlineboutique
```

Replace POD-NAME with the actual pod name from kubectl get pods. The logs will usually tell you what went wrong.

### Question: Can I delete just the application without deleting the cluster?

Yes. Run this to remove all the Online Boutique resources:

```bash
kubectl delete -f ./release/kubernetes-manifests.yaml -n onlineboutique
kubectl delete -f online-boutique-ingress.yaml
```

The cluster and the Ingress Controller will remain running.

---

## Summary of Module 4

By the end of this module you have done the following:

- Understood the difference between a monolithic and a microservice application
- Understood what Ingress is and why it is better than creating one Load Balancer per service
- Installed the NGINX Ingress Controller on EKS using Helm
- Deployed all 11 microservices of the Online Boutique application
- Created an Ingress resource to route traffic to the frontend service
- Accessed the running application from your browser

You now have a real microservice application running on Kubernetes, receiving traffic, and generating data. In the next module we will discuss why monitoring this application is fundamentally different from monitoring a monolithic application, and why we need a tool like Prometheus.

---
