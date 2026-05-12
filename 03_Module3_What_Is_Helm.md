# Module 3: What is Helm?

---

## 3.1 The Problem Helm Solves

Before we talk about Helm, let us understand the pain it removes.

### Deploying an Application on Kubernetes Without Helm

Say you want to deploy a simple web application on Kubernetes. You need to create the following files manually:

- deployment.yaml
- service.yaml
- configmap.yaml
- ingress.yaml
- serviceaccount.yaml
- secret.yaml

That is 6 YAML files for one application. Now imagine you have 11 microservices like our Online Boutique application. That is potentially 60 plus YAML files to manage.

Now imagine your manager says, "Deploy this same application in three environments: dev, staging, and production. But in dev use 1 replica, in staging use 2, and in production use 5. Also use different database passwords in each environment."

Without Helm, you would maintain three copies of all those YAML files with tiny differences between them. One day someone edits the dev file and forgets to update production. Things break. No one knows why.

This is the problem Helm solves.

---

## 3.2 What is Helm?

Helm is the package manager for Kubernetes.

If you have used Linux before, you know apt-get or yum. You type one command and the tool figures out what to install, in what order, with what dependencies.

If you have used Node.js, you know npm. You type npm install and everything your application needs comes down automatically.

Helm does the same thing for Kubernetes applications.

Instead of writing and managing dozens of YAML files yourself, you use a Helm Chart. A chart is a pre-packaged collection of all the Kubernetes files needed to run an application. Someone has already written the YAML. You just tell Helm to install it.

Here is how simple it becomes with Helm:

Without Helm you would write 6 YAML files, apply them one by one with kubectl, manage them across environments manually, and pray nothing breaks.

With Helm you run one command:

```bash
helm install my-app prometheus-community/kube-prometheus-stack
```

And Helm installs everything. One command. Done.

---

## 3.3 Key Concepts in Helm

There are four terms you must understand before using Helm.

### Chart

A Chart is a Helm package. It contains all the YAML templates needed to deploy an application on Kubernetes. Think of it like an APT package on Ubuntu or an RPM package on RedHat. Charts can be stored locally on your machine or in a remote chart repository on the internet.

### Repository

A Repository is a place where Charts are stored and shared. It works exactly like DockerHub for Docker images, or PyPI for Python packages. Anyone can publish a Helm Chart to a public repository. You add a repository to your Helm installation, and then you can install any chart from it.

The most commonly used public Helm repositories are:

| Repository | What It Contains |
|------------|-----------------|
| Artifact Hub | Central index of charts from hundreds of sources |
| Prometheus Community | Charts for Prometheus, Grafana, Alertmanager |
| Bitnami | Charts for popular apps like MySQL, Redis, WordPress |
| Ingress-NGINX | Charts for the NGINX Ingress Controller |

### Release

A Release is a specific installation of a Chart in your Kubernetes cluster. When you run helm install, Helm creates a Release. You can install the same Chart multiple times with different names and different configurations. Each installation is a separate Release.

For example, you can install the same nginx chart twice, once as my-frontend and once as my-backend. These are two separate Releases.

### Values

Values are the configuration settings you pass to a Chart to customize it. Every Chart comes with a default values.yaml file. You can override any value when you install the chart. This is how you handle the different configurations across dev, staging, and production.

---

## 3.4 How Helm Works: The Big Picture

Here is what happens when you run helm install:

Step 1: Helm reads the Chart from the repository or your local machine.

Step 2: Helm takes your custom values and merges them with the default values in the chart.

Step 3: Helm renders the YAML templates by substituting all the variables with actual values.

Step 4: Helm sends all the final YAML files to the Kubernetes API server.

Step 5: Kubernetes creates all the resources, deployments, services, configmaps, and so on.

Step 6: Helm records this installation as a Release and tracks it. This is how it knows what to upgrade or rollback later.

---

## 3.5 Helm 2 vs Helm 3

You may read about something called Tiller when you search for Helm online. Tiller was a server-side component that Helm 2 required to be installed inside your Kubernetes cluster. It caused many security problems because it had full admin access to the cluster.

Helm 3 removed Tiller completely. There is no server-side component anymore. Helm 3 talks directly to the Kubernetes API server using your kubeconfig file, the same way kubectl does.

We are using Helm 3 in this workshop. You do not need to worry about Tiller at all.

| Feature | Helm 2 | Helm 3 |
|---------|--------|--------|
| Server component | Required Tiller | No Tiller, talks directly to Kubernetes |
| Security | Tiller had full cluster access | Uses your kubeconfig permissions |
| Release storage | Stored in ConfigMaps | Stored as Secrets in the namespace |
| Status | End of life | Current stable version |

---

## 3.6 Helm is Already Installed

We installed Helm in Module 2 on the EC2 Jump Box. Let us verify it is working.

```bash
helm version
```

You should see output like this:

```
version.BuildInfo{Version:"v3.x.x", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.x.x"}
```

---

## 3.7 Essential Helm Commands

These are the commands you will use in this workshop and in your day to day work.

### Add a Repository

Before you can install a chart, you need to tell Helm where to find it.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

This adds the Prometheus Community repository to your local Helm installation.

### Update Repositories

This refreshes the list of available charts from all your added repositories. Always run this before installing a chart.

```bash
helm repo update
```

### List Your Added Repositories

```bash
helm repo list
```

You will see something like:

```
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
```

### Search for a Chart

Search within your added repositories:

```bash
helm search repo prometheus
```

Search on Artifact Hub which is the public index of all public charts:

```bash
helm search hub prometheus
```

### Install a Chart

```bash
helm install RELEASE-NAME REPO-NAME/CHART-NAME
```

For example:

```bash
helm install my-prometheus prometheus-community/kube-prometheus-stack
```

Here:
- my-prometheus is the Release name. You choose this.
- prometheus-community is the repository name you added.
- kube-prometheus-stack is the Chart name inside that repository.

### Install a Chart with Custom Values

```bash
helm install my-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values custom-values.yaml
```

### List All Installed Releases

```bash
helm list
```

Or to see releases across all namespaces:

```bash
helm list -A
```

### Check the Status of a Release

```bash
helm status my-prometheus
```

### Upgrade a Release

When you want to change the configuration of an already installed chart:

```bash
helm upgrade my-prometheus prometheus-community/kube-prometheus-stack \
  --values custom-values.yaml
```

### Rollback a Release

If an upgrade breaks something, you can roll back to the previous version:

```bash
helm rollback my-prometheus 1
```

The number 1 is the revision number you want to go back to.

### View Release History

```bash
helm history my-prometheus
```

### Uninstall a Release

```bash
helm uninstall my-prometheus
```

This removes all Kubernetes resources that were created by this release.

---

## 3.8 Looking Inside a Chart

It is useful to understand what is inside a Helm chart. Let us pull one down and look at it.

```bash
helm pull prometheus-community/kube-prometheus-stack --untar
ls kube-prometheus-stack/
```

You will see a structure like this:

```
kube-prometheus-stack/
    Chart.yaml          - Metadata about the chart: name, version, description
    values.yaml         - All default configuration values
    charts/             - Dependency charts bundled inside
    templates/          - All the Kubernetes YAML templates
        deployment.yaml
        service.yaml
        configmap.yaml
        ...
```

The templates folder contains YAML files with variables in them like this:

```yaml
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

When you run helm install, Helm replaces all these variables with actual values from your values.yaml file and sends the final YAML to Kubernetes.

---

## 3.9 Common Questions from Students

### Question: Do I need to write my own Helm Charts?

Not for this workshop. We will use existing charts from public repositories. Writing your own charts is a more advanced topic you can explore after this session.

### Question: Where can I find charts for any application?

Go to https://artifacthub.io and search for any application. Artifact Hub is the central index for public Helm charts.

### Question: What is the difference between helm install and helm upgrade?

helm install creates a brand new release. helm upgrade updates an existing release with new configuration or a new chart version. If you are not sure whether a release exists, you can use helm upgrade --install which does both. It installs if it does not exist and upgrades if it does.

### Question: Is Helm specific to AWS or EKS?

No. Helm works with any Kubernetes cluster, whether it is EKS on AWS, AKS on Azure, GKE on Google Cloud, or even a local Minikube cluster on your laptop.

---

## Summary of Module 3

By the end of this module you understand the following:

- Why Helm was created and what problem it solves
- What a Chart, Repository, Release, and Values are
- The difference between Helm 2 and Helm 3
- The core Helm commands you will use throughout this workshop

In the next module we will use Helm to deploy the NGINX Ingress Controller and then deploy the Google Online Boutique microservice application on our EKS cluster.

---
