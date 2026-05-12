## NLB vs ALB — Why We Chose NLB for NGINX Ingress

### First, Understand What Each One Is

| | NLB (Network Load Balancer) | ALB (Application Load Balancer) |
|---|---|---|
| OSI Layer | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| Knows about | IP addresses and ports | URLs, headers, cookies, hostnames |
| Speed | Extremely fast, very low latency | Slightly slower, more processing |
| Cost | Lower | Higher |
| AWS Service | Classic AWS networking | Requires AWS Load Balancer Controller add-on |

---

### The Core Reason We Use NLB Here

When you use the **NGINX Ingress Controller**, the intelligence of routing happens **inside the cluster**, not at the Load Balancer level.

Think of it this way:

```
With NLB + NGINX Ingress:

Internet
   |
   v
NLB (just passes TCP traffic through, no routing logic)
   |
   v
NGINX Ingress Controller (does all the smart routing inside the cluster)
   |
   v
Correct Service and Pod
```

```
With ALB:

Internet
   |
   v
ALB (does the smart routing itself at the AWS level)
   |
   v
Service and Pod directly
```

So with NGINX Ingress, you are saying "I want NGINX to handle all my routing rules." The Load Balancer in front just needs to pass traffic through cleanly. NLB is perfect for this because it is a simple, fast, transparent pass-through.

---

### Why Not ALB Then?

You absolutely can use ALB on EKS. But it works differently and has a different requirement:

To use ALB on EKS you need the **AWS Load Balancer Controller** installed separately. This is an additional Kubernetes controller that watches for Ingress resources and creates a new ALB for each one automatically.

| Approach | What You Need | Routing Logic Lives |
|----------|--------------|---------------------|
| NLB plus NGINX Ingress | Just Helm install NGINX | Inside the cluster in NGINX |
| ALB | AWS Load Balancer Controller plus IAM OIDC setup | At the AWS ALB level |

Setting up ALB requires:

- Enabling OIDC provider on the EKS cluster
- Creating an IAM policy and IAM service account
- Installing the AWS Load Balancer Controller via Helm
- Annotating your Ingress resources with ALB-specific annotations

That is 4 extra steps before you even deploy your application. .

---

### When Would You Choose ALB in Production?

| Situation | Best Choice |
|-----------|------------|
| Workshop, learning, simple routing | NLB plus NGINX Ingress |
| Need path and host based routing inside cluster | NLB plus NGINX Ingress |
| Need AWS WAF integration | ALB |
| Need AWS Cognito authentication at load balancer level | ALB |
| Need SSL termination managed entirely by AWS ACM | ALB |
| Team is already using AWS-native tooling everywhere | ALB |

---

### The Simple Answer 


"NLB just opens the door and passes everything to NGINX. NGINX is the smart one that decides where traffic goes. We do not need two smart things in a row. One smart router is enough. NLB is cheap, fast, and simple. ALB needs extra setup that we do not need today."

