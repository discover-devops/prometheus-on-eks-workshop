## What is kubernetes-manifests.yaml?

Yes, your understanding is exactly right.

A manifest file is simply a collection of all the Kubernetes object definitions needed to run an application, combined into one single file.

In our case, kubernetes-manifests.yaml contains all of the following for all 11 services in one file:

```
For each of the 11 microservices, it defines:

  Deployment      --> how many replicas, which container image, resource limits
  Service         --> how other pods inside the cluster talk to this service
```

So instead of having 22 separate files (11 deployments + 11 services), Google combined everything into one file. You apply it once and all 11 services come up together.

You can actually look inside it:

```bash
cat ./release/kubernetes-manifests.yaml | grep "^kind:"
```

You will see output like this:

```
kind: Deployment
kind: Service
kind: Deployment
kind: Service
kind: Deployment
kind: Service
... repeated 11 times
```

---

## What is istio-manifests.yaml?

This is where your observation is sharp.

The repo has two manifest files:

| File | Purpose |
|------|---------|
| kubernetes-manifests.yaml | Plain Kubernetes deployment, no service mesh, this is what we use |
| istio-manifests.yaml | Additional objects needed ONLY if you are using Istio service mesh |

We are NOT using Istio in this workshop. We are using the plain kubernetes-manifests.yaml only.

---

## What is Istio and Why Does it Exist?

Since you asked, let me explain it clearly so students are not confused when they see this file.

Istio is a Service Mesh. Let us understand what that means.

### The Problem Istio Solves

Remember our 11 microservices. They talk to each other over the network using gRPC.

```
frontend --> checkoutservice --> paymentservice --> emailservice
```

Now think about these questions:

- What if you want to encrypt all traffic between services? (mTLS)
- What if you want to see exactly how long each service-to-service call takes? (tracing)
- What if you want to send 10 percent of traffic to a new version and 90 percent to the old version? (canary deployment)
- What if you want to retry failed calls automatically? (resilience)

Without Istio, you would have to write all this logic inside every single application. That means every developer on every team has to implement security, tracing, and traffic management in their own code.

Istio solves this by injecting a small proxy container called Envoy as a sidecar into every pod. This proxy intercepts all network traffic going in and out of the pod and handles all of the above automatically, without changing any application code.

```
Without Istio:
frontend pod  --------gRPC-------->  checkoutservice pod

With Istio:
frontend pod --> envoy proxy  --------gRPC-------->  envoy proxy --> checkoutservice pod
                (handles mTLS,                        (handles mTLS,
                 retries, tracing)                     retries, tracing)
```

### Why We Are NOT Using Istio Today

| Reason | Explanation |
|--------|-------------|
| Complexity | Istio is a significant topic on its own. It deserves a separate full workshop |
| Time | Setting up Istio properly takes 30 to 45 minutes |
| Our goal | We are here to learn Prometheus monitoring, not service mesh |
| Prometheus works without Istio | We get full metrics visibility without needing Istio at all |

---

## How All These Things Are Connected

Let me give you the complete picture of everything running in our cluster right now:

```
                        INTERNET
                           |
                    AWS Network LB
                           |
               NGINX Ingress Controller
                           |
                    frontend Service
                           |
                     frontend Pod
                    /            \
                   /              \
        productcatalog        checkoutservice
           Service                Service
              |                      |
        productcatalog          checkoutservice
            Pod                     Pod
                                /        \
                               /          \
                       payment           shipping
                       Service            Service
                          |                  |
                       payment           shipping
                         Pod               Pod

  (cartservice uses Redis inside the cluster)
  (loadgenerator pod automatically sends traffic to frontend)
```

All communication between pods goes through Kubernetes Services using ClusterIP. No Istio. No service mesh. Plain Kubernetes networking.

Prometheus, when we install it in Module 7, will sit alongside all of this and scrape metrics from every pod, every node, and the Kubernetes API itself.

---

## Summary of Your Three Questions

| Question | Answer |
|----------|--------|
| What is kubernetes-manifests.yaml? | One file containing Deployment and Service definitions for all 11 microservices |
| Are we using Istio? | No. istio-manifests.yaml is in the repo but we ignore it. We use plain Kubernetes |
| How are things connected? | Pods talk to each other via ClusterIP Services. External traffic comes through NLB and NGINX Ingress to the frontend |

