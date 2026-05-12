# Module 2: Kubernetes Cluster Setup on AWS EKS

---

## 2.1 What is EKS and Why Are We Using It?

### The Problem First

Let us say you want to run Kubernetes. You have two choices.

Choice 1: You set up Kubernetes yourself. You provision servers, install the control plane, configure etcd, set up networking, manage certificates, handle upgrades, and monitor everything. This is called a self-managed Kubernetes cluster. It takes days to get right and weeks to get stable.

Choice 2: You tell AWS, "Give me a Kubernetes cluster." AWS provisions and manages the control plane for you. You only manage the worker nodes where your applications run. This is Amazon EKS, which stands for Elastic Kubernetes Service.

For this workshop, we are using EKS because:

- We want to focus on learning Prometheus, not on setting up Kubernetes internals
- EKS is what most companies use in production on AWS
- It works reliably and quickly for a hands-on session

### How EKS Works

EKS has two parts.

The first part is the Control Plane. This is managed entirely by AWS. You do not see these servers. AWS runs them, patches them, and keeps them available. The control plane includes the API server, the scheduler, etcd, and the controller manager.

The second part is the Worker Nodes. These are regular EC2 instances in your AWS account. Your applications (pods) run on these nodes. You are responsible for these machines, but eksctl makes creating and managing them simple.

Here is a simple way to think about it:

AWS manages the brain of Kubernetes. You manage the hands that do the work.

---

## 2.2 What is eksctl?

eksctl is a command line tool that creates and manages EKS clusters. Without eksctl, creating an EKS cluster would require you to:

- Create a VPC manually
- Create subnets across multiple availability zones
- Create IAM roles for the control plane and worker nodes
- Create security groups
- Register the nodes with the cluster
- Configure networking

This would take 30 to 45 minutes of manual clicking in the AWS console and a high chance of making mistakes.

With eksctl, you run one command and it does all of the above automatically using AWS CloudFormation behind the scenes. The same cluster is ready in about 15 minutes.

---

## 2.3 Step 1: Launch the EC2 Jump Box

We will first launch an EC2 instance on AWS. This instance will be our command center for the rest of the workshop. Everything we do, we do from this machine.

### Go to AWS Console and Launch an EC2 Instance

Use the following settings when launching the instance:

| Setting | Value |
|---------|-------|
| Name | eks-jumpbox |
| AMI | Ubuntu Server 24.04 LTS |
| Instance Type | t2.micro (free tier) or t3.medium |
| Key Pair | Create a new key pair, download the .pem file |
| Security Group | Allow SSH (port 22) from your IP |
| Storage | 20 GB gp2 |

After the instance is running, note the Public IP address. You will need it to connect.

### Convert the Key File for PuTTY (Windows Users)

AWS gives you a .pem file. PuTTY does not understand .pem files. You need to convert it to a .ppk file using PuTTYgen.

Step 1: Open PuTTYgen on your Windows laptop.

Step 2: Click Load and select your .pem file. You may need to change the file filter to show All Files.

Step 3: Click Save private key. Save it as eks-jumpbox.ppk. You can ignore the warning about no passphrase.

### Connect Using PuTTY

Step 1: Open PuTTY on your Windows laptop.

Step 2: In the Host Name field, type the following. Replace the IP with your actual EC2 public IP.

```
ubuntu@YOUR_EC2_PUBLIC_IP
```

Step 3: On the left panel, go to Connection, then SSH, then Auth, then Credentials. Browse and select your eks-jumpbox.ppk file.

Step 4: Click Open. If you see a security warning, click Accept.

You should now see a terminal prompt like this:

```
ubuntu@ip-172-31-XX-XX:~$
```

You are now inside your EC2 Jump Box. All commands from this point forward are typed in this terminal.

---

## 2.4 Step 2: Install All Required Tools on the Jump Box

We will install four tools: AWS CLI, kubectl, eksctl, and helm. We install helm here and explain it in detail in Module 3.

Run all commands in your PuTTY terminal.

### Update the System First

```bash
sudo apt-get update -y
```

This updates the list of available packages. Always do this first on a fresh Ubuntu machine.

### Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install unzip -y
unzip awscliv2.zip
sudo ./aws/install
```

Verify the installation:

```bash
aws --version
```

You should see output like this:

```
aws-cli/2.x.x Python/3.x.x Linux/x86_64
```

### Configure AWS CLI with Your Credentials

```bash
aws configure
```

You will be asked four questions. Answer them as follows:

```
AWS Access Key ID: YOUR_ACCESS_KEY_ID
AWS Secret Access Key: YOUR_SECRET_ACCESS_KEY
Default region name: ap-south-1
Default output format: json
```

Note: Replace ap-south-1 with the AWS region you want to use. ap-south-1 is the Mumbai region, which is closest to Hyderabad. You can also use us-east-1 (N. Virginia) or any region your account supports.

To verify AWS is configured correctly:

```bash
aws sts get-caller-identity
```

You should see your AWS account ID, user ID, and ARN. If you see an error, your credentials are incorrect.

### Install kubectl

kubectl is the command line tool to talk to Kubernetes. Think of it as the remote control for your cluster.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify the installation:

```bash
kubectl version --client
```

You should see the kubectl version printed. It is fine if you see a warning about the server not being reachable. We have not created the cluster yet.

### Install eksctl

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

Verify the installation:

```bash
eksctl version
```

You should see the eksctl version number.

### Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify the installation:

```bash
helm version
```

We will explain what Helm is and how it works in Module 3. For now, just get it installed.

### Final Check: All Tools Installed

Run this to confirm everything is ready:

```bash
aws --version
kubectl version --client
eksctl version
helm version
```

All four should print their version numbers without errors. If any one fails, reinstall that tool before moving forward.

---

## 2.5 Step 3: Attach an IAM Role to the EC2 Jump Box

This is a step that many beginners miss and then get confused when eksctl fails with permission errors.

When eksctl runs from your EC2 machine, it needs permission to create AWS resources like VPCs, subnets, IAM roles, and EC2 instances. Instead of storing your AWS credentials on the machine permanently, we use an IAM Role attached to the EC2 instance. This is the correct and secure way to do it in AWS.

If you already ran aws configure with your access keys in the previous step, that will also work for this workshop. But understand that in production, using an IAM role attached to EC2 is the right approach.

### Create an IAM Role for the Jump Box (If Not Using Access Keys)

Step 1: Go to IAM in the AWS Console.

Step 2: Click Roles, then click Create Role.

Step 3: Select AWS Service, then select EC2. Click Next.

Step 4: Attach the following policies:

- AdministratorAccess (for this workshop only. In production, use least privilege.)

Step 5: Name the role eks-jumpbox-role and create it.

Step 6: Go back to EC2, select your eks-jumpbox instance, click Actions, then Security, then Modify IAM Role. Select eks-jumpbox-role and save.

After attaching the role, if you used aws configure earlier, you can keep it as is. Both methods work.

---

## 2.6 Step 4: Create the EKS Cluster Using eksctl

We will create the cluster using a configuration file. This is better than passing everything as command line flags because it is readable, repeatable, and easy to share with the team.

### Create the Cluster Configuration File

```bash
cat > eks-cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: prometheus-lab-cluster
  region: ap-south-1
  version: "1.31"

nodeGroups:
  - name: worker-nodes
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 2
    volumeSize: 20
    volumeType: gp2
    amiFamily: AmazonLinux2023
    ssh:
      allow: false
EOF
```

Let us understand what each field means.

| Field | What It Means |
|-------|--------------|
| name | The name of your EKS cluster |
| region | The AWS region where the cluster is created |
| version | The Kubernetes version. 1.31 is a recent stable version |
| instanceType | The EC2 type for worker nodes. t3.medium has 2 vCPU and 4 GB RAM |
| desiredCapacity | We want exactly 2 worker nodes |
| minSize and maxSize | Both set to 2 because we do not want auto scaling for this lab |
| volumeSize | 20 GB disk per node |
| volumeType | gp2 is the standard SSD disk type on AWS |
| amiFamily | AmazonLinux2023 is the latest Amazon Linux version |

### Run the eksctl Command to Create the Cluster

```bash
eksctl create cluster -f eks-cluster.yaml
```

This command will take approximately 15 to 20 minutes to complete. This is normal. Do not interrupt it.

While it runs, you will see logs like these:

```
[i]  eksctl version x.x.x
[i]  using region ap-south-1
[i]  creating EKS cluster "prometheus-lab-cluster" in "ap-south-1" region with managed nodes
[i]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
[i]  if you encounter any issues, check CloudFormation console
[i]  building cluster stack "eksctl-prometheus-lab-cluster-cluster"
[i]  deploying stack "eksctl-prometheus-lab-cluster-cluster"
...
[i]  building managed nodegroup stack "eksctl-prometheus-lab-cluster-nodegroup-worker-nodes"
[i]  deploying stack "eksctl-prometheus-lab-cluster-nodegroup-worker-nodes"
...
[i]  kubectl command should work with "/home/ubuntu/.kube/config"
[v]  EKS cluster "prometheus-lab-cluster" in "ap-south-1" region is ready
```

The last line that says the cluster is ready is your confirmation that everything worked.

### What Happened Behind the Scenes?

While eksctl was running, it did all of the following for you automatically:

- Created a VPC with public and private subnets across 2 availability zones
- Created Internet Gateway and route tables
- Created IAM roles for the control plane and worker nodes
- Created security groups for the cluster and nodes
- Launched 2 EC2 instances (your worker nodes) in an Auto Scaling Group
- Registered the worker nodes with the EKS control plane
- Updated your kubeconfig file so kubectl can talk to the cluster

You can verify all of this by going to the CloudFormation console in AWS and looking at the two stacks that eksctl created.

---

## 2.7 Step 5: Verify the Cluster

Now let us make sure everything is working correctly.

### Check That kubectl Is Pointing to the Right Cluster

```bash
kubectl config current-context
```

You should see something like:

```
ubuntu@prometheus-lab-cluster.ap-south-1.eksctl.io
```

This confirms kubectl knows which cluster to talk to.

### Check the Nodes

```bash
kubectl get nodes
```

You should see two nodes listed like this:

```
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-XX-XX.ap-south-1.compute.internal  Ready    <none>   5m    v1.31.x
ip-192-168-XX-XX.ap-south-1.compute.internal  Ready    <none>   5m    v1.31.x
```

Both nodes must show STATUS as Ready. If a node shows NotReady, wait another 2 minutes and try again. Nodes sometimes take a little longer to fully initialize.

### Check the Namespaces

```bash
kubectl get namespaces
```

You should see the default Kubernetes namespaces:

```
NAME              STATUS   AGE
default           Active   10m
kube-node-lease   Active   10m
kube-public       Active   10m
kube-system       Active   10m
```

### Check the System Pods

```bash
kubectl get pods -n kube-system
```

This shows the internal Kubernetes components running on your cluster. You should see pods for coredns, kube-proxy, and the AWS VPC CNI plugin. All of them should be in Running status.

### Check the Cluster Info

```bash
kubectl cluster-info
```

This shows the address of your Kubernetes API server. If this command returns without errors, your cluster is healthy and your kubectl is connected correctly.

---

## 2.8 Common Questions and Errors

### Question: My eksctl command is failing with "Unable to create cluster"

Check the CloudFormation console in your AWS account. Look for stacks with names starting with eksctl-. Click on the stack that shows a FAILED status and read the Events tab. It will tell you exactly what went wrong. Common reasons are insufficient IAM permissions or hitting EC2 instance limits in the region.

### Question: I see "error: You must be logged in to the server (Unauthorized)"

This means kubectl cannot authenticate to the cluster. This happens when the IAM user or role that is running kubectl is different from the one that created the cluster. Run the following command to refresh the kubeconfig:

```bash
aws eks update-kubeconfig --region ap-south-1 --name prometheus-lab-cluster
```

### Question: The cluster creation worked but kubectl get nodes shows nothing

Make sure the worker nodes were created. Go to EC2 in the AWS console and look for instances. If you see two running instances, run the update-kubeconfig command above and try kubectl get nodes again.

### Question: Can I stop the EC2 nodes to save money during a break?

No. If you stop the worker nodes from the EC2 console, the Auto Scaling Group that eksctl created will launch new ones to replace them. To truly stop costs, you need to delete the cluster. Use this command at the end of the session:

```bash
eksctl delete cluster -f eks-cluster.yaml
```

---

## Summary of Module 2

By the end of this module you have done the following:

- Launched an EC2 Jump Box on Ubuntu
- Connected to it from your Windows laptop using PuTTY
- Installed AWS CLI, kubectl, eksctl, and helm
- Created a 2-node EKS cluster using a configuration file
- Verified the cluster is healthy using kubectl

You now have a working Kubernetes cluster running on AWS. In the next module we will learn what Helm is and understand why it is so important before we deploy any applications.

---
