# Day 12: Kubernetes Cluster Setup on Amazon EKS

## Overview

Day 12 focused on understanding Kubernetes cluster architecture and setting up a Kubernetes cluster using Amazon Elastic Kubernetes Service (EKS).

Created an EKS cluster in the Mumbai region, configured an Ubuntu VM to communicate with the cluster using AWS CLI and kubectl, practised switching Kubernetes contexts, and learned how to troubleshoot common connectivity and authentication issues.

The original learning plan used Google Kubernetes Engine (GKE). I performed the same exercise using Amazon EKS because my DevOps learning environment is based on AWS.

## 1. Cluster Architecture

```text
              Ubuntu VM
                  |
                  |
          AWS CLI + kubectl
                  |
                  |
             AWS EKS API
                  |
          Kubernetes Cluster
                  |
          -------------------
          |                 |
     Control Plane      Worker Nodes
          |                 |
       Scheduler           Pods
       API Server        Containers
       Controllers
```

### Control Plane

The control plane manages the Kubernetes cluster.

Its responsibilities include:

- Managing the cluster.
- Scheduling applications.
- Monitoring the desired state.
- Managing Kubernetes resources.
- Handling API requests.

Amazon EKS manages the Kubernetes control plane.

### Worker Nodes

Worker nodes are responsible for running application workloads.

They contain:

- Kubelet.
- Container runtime.
- Kube-proxy (depending on the networking configuration).

Applications run inside containers, which are organized into Kubernetes Pods.

## 2. Cluster Configuration

| Setting | Value |
|---|---|
| Cluster Name | `devops-learning-eks` |
| Cloud Provider | AWS |
| Kubernetes Service | Amazon EKS |
| AWS Region | `ap-south-1` (Mumbai) |
| Compute | Managed Node Group |
| Number of Nodes | 3 |
| Authentication Mode | EKS API (Access Entries) |
| Endpoint Access | Public and Private |
| Add-on | metrics-server |
| Logging | CloudWatch Control Plane Logging |
| Deletion Protection | Off |

## 3. Tools Installed

Configured the Ubuntu VM with the following tools:

| Tool | Purpose |
|---|---|
| AWS CLI | Interact with AWS services |
| kubectl | Manage Kubernetes resources |
| Amazon EKS | Managed Kubernetes service |

### Verify installations

```bash
aws --version
kubectl version --client
```

## 4. Connecting to the EKS Cluster

Configured kubectl to communicate with the EKS cluster.

### Command

```bash
aws eks update-kubeconfig \
  --name devops-learning-eks \
  --region ap-south-1
```

### Explanation

- `aws eks` – Interacts with Amazon EKS.
- `update-kubeconfig` – Updates the Kubernetes configuration file.
- `--name` – Specifies the EKS cluster name.
- `--region` – Specifies the AWS region.

This command updates the kubeconfig file, allowing kubectl to communicate with the EKS cluster using AWS authentication.

## 5. Understanding Kubernetes Contexts

A Kubernetes context stores information about which cluster, user, and namespace kubectl should use.

### List available contexts

```bash
kubectl config get-contexts
```

### Switch context

```bash
kubectl config use-context <context-name>
```

### Check current context

```bash
kubectl config current-context
```

**Learning:** Contexts help manage multiple Kubernetes clusters from the same machine.

## 6. Checking the Kubernetes Cluster

### A. List Worker Nodes

```bash
kubectl get nodes
```

Displays the worker nodes connected to the Kubernetes cluster.

Expected status:

```text
NAME          STATUS
<node-name>   Ready
<node-name>   Ready
<node-name>   Ready
```

### B. List Pods Across All Namespaces

```bash
kubectl get pods -A
```

Displays Pods across all Kubernetes namespaces.

### C. Check Cluster Information

```bash
kubectl cluster-info
```

Displays information about the Kubernetes control plane and cluster services.

### D. Check Namespaces

```bash
kubectl get namespaces
```

Displays the namespaces available in the cluster.

## 7. Important Kubernetes Concepts

### A. Control Plane vs Worker Nodes

| Control Plane | Worker Nodes |
|---|---|
| Manages the cluster | Runs applications |
| Schedules Pods | Hosts Pods |
| Maintains desired state | Executes workloads |
| Managed by AWS in EKS | Provided through node groups |

### B. What is a Kubernetes Context?

A context is a saved configuration that tells kubectl which cluster and user to communicate with.

### C. What is NodePort?

NodePort is a Kubernetes Service type that exposes an application through a port on every node.

Default NodePort range:

`30000–32767`

For example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  type: NodePort
  selector:
    app: demo
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

Here:

- `port: 80` – Service port.
- `targetPort: 8080` – Application port inside the Pod.
- `nodePort: 30080` – Port exposed on the nodes.

### D. What is metrics-server?

metrics-server collects resource usage metrics such as CPU and memory from Kubernetes nodes and Pods.

It is useful for monitoring resource consumption and supporting Horizontal Pod Autoscaling (HPA).

## 8. Troubleshooting

Practised identifying common kubectl connection problems.

| Error | Possible Cause | Solution |
|---|---|---|
| `localhost:8080 was refused` | Missing or incorrect kubeconfig | Run `aws eks update-kubeconfig` |
| `dial tcp: i/o timeout` | Network or endpoint connectivity issue | Check endpoint access, security rules and outbound connectivity |
| `Unauthorized` | AWS identity does not have cluster access | Check AWS identity and EKS access entries |
| No nodes available in Auto Mode | Nodes might not have been provisioned yet | Check workloads and node provisioning |

### Useful troubleshooting commands

Check AWS identity:

```bash
aws sts get-caller-identity
```

Check current Kubernetes context:

```bash
kubectl config current-context
```

Check cluster connectivity:

```bash
kubectl cluster-info
```

Check nodes:

```bash
kubectl get nodes
```

Check all Pods:

```bash
kubectl get pods -A
```

## 9. Security Practices

- Used AWS IAM authentication for cluster access.
- Configured EKS access entries.
- Enabled public and private endpoint access with appropriate restrictions.
- Avoided unrestricted API endpoint access.
- Used AWS-managed EKS control plane.
- Enabled CloudWatch control plane logging.

## 10. Cost Management

Amazon EKS charges for the Kubernetes control plane while the cluster exists.

Additional charges may include:

- EC2 worker nodes.
- EBS volumes.
- NAT Gateway.
- Load Balancers.
- CloudWatch log ingestion and storage.
- Other associated AWS resources.

### Cleanup Plan

The cluster will be deleted after completing the Kubernetes learning block on Day 14.

For a managed node group, delete the node group before deleting the cluster.

Delete node group:

```bash
aws eks delete-nodegroup \
  --cluster-name devops-learning-eks \
  --nodegroup-name <nodegroup-name> \
  --region ap-south-1
```

Delete cluster:

```bash
aws eks delete-cluster \
  --name devops-learning-eks \
  --region ap-south-1
```

Verify deletion:

```bash
aws eks describe-cluster \
  --name devops-learning-eks \
  --region ap-south-1
```

Note: Deleting the EKS cluster does not automatically remove every associated AWS resource. Verify and clean up any remaining resources separately.

## 11. Interview Questions

**Q1. What is Kubernetes?**

Kubernetes is an open-source container orchestration platform used to deploy, manage, and scale containerized applications.

**Q2. What is Amazon EKS?**

Amazon Elastic Kubernetes Service is AWS's managed Kubernetes service.

**Q3. What is the difference between a control plane and a worker node?**

The control plane manages the cluster, while worker nodes execute application workloads.

**Q4. What is kubectl?**

kubectl is the command-line tool used to communicate with and manage Kubernetes clusters.

**Q5. What is a Kubernetes context?**

A context identifies the cluster, user, and namespace used by kubectl.

**Q6. What is a NodePort?**

NodePort exposes a Kubernetes Service through a specific port on every node.

**Q7. Why do we use metrics-server?**

It collects CPU and memory usage metrics from Kubernetes nodes and Pods.

**Q8. How do you connect kubectl to an EKS cluster?**

Using:

```bash
aws eks update-kubeconfig \
  --name devops-learning-eks \
  --region ap-south-1
```

## 12. What I Achieved

By completing Day 12, I learned:

1. Kubernetes cluster architecture.
2. Difference between control plane and worker nodes.
3. Amazon EKS fundamentals.
4. AWS CLI configuration for Kubernetes access.
5. Installing and using kubectl.
6. Connecting an Ubuntu VM to an EKS cluster.
7. Managing Kubernetes contexts.
8. Listing nodes, Pods, and namespaces.
9. Understanding NodePort and metrics-server.
10. Troubleshooting Kubernetes connectivity and authentication issues.
11. Understanding EKS cost management and cleanup.

## 13. Next Steps

- Learn Kubernetes Pods.
- Create and manage Deployments.
- Understand ReplicaSets.
- Explore Kubernetes Services.
- Practise scaling applications.
- Learn Kubernetes YAML manifests.

---

**Day 12 Status: COMPLETED**

**Main Takeaway:**

Amazon EKS provides a managed Kubernetes control plane, while Kubernetes worker nodes run containerized applications. Using AWS CLI and kubectl, administrators can connect to the cluster, inspect resources, and manage application workloads.
