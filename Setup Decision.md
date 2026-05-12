# Sample Application

##  Use **Online Boutique**


- Online Boutique is a cloud-first microservices demo application — a web-based e-commerce app where users can browse items, add them to the cart, and purchase them. It works on **any** Kubernetes cluster, not just GKE/OKE/EKS/AKS.

- It supports **Helm deployment** out of the box.
- It is **actively maintained** — with 117+ dependency updates including bug fixes and security patches as recently as late 2025.
- It has a **built-in load generator** — so Prometheus will show real traffic metrics automatically, no manual curling needed!
- It is composed of 11 microservices written in different languages that communicate over gRPC — great for showing students why monitoring gets complex.


## Your Setup Decision — EC2 Jump Box  (Best for Your Class)

Since students are on **Windows + PuTTY**, here's my strong recommendation:

```
Windows Laptop (PuTTY)
        ↓  SSH
  EC2 Instance (Amazon Linux 2 / Ubuntu)   ← "Jump Box" / Bastion
        ↓  runs
  eksctl + kubectl + helm + aws cli
        ↓  creates
  EKS Cluster (2 nodes)
```

**Why EC2 Jump Box wins over VS Code / Git Bash:**

- Everyone will have the **exact same environment** — no "works on my machine" issues
- No local tool installation headaches on Windows
- You control the EC2 — pre-install all tools before class
- Students only need PuTTY (they already have it)
- If a student's laptop dies → just re-SSH, nothing lost

---

## Updated ToC Decisions:

| Item | Decision |
|---|---|
| Demo App |  Google Online Boutique |
| Student OS | Windows + PuTTY |
| Where to run commands | EC2 Jump Box (pre-configured) |
| EKS creation | `eksctl` from EC2 |
| Expose app | NGINX Ingress |
| Monitoring | Prometheus + Grafana via Helm |

