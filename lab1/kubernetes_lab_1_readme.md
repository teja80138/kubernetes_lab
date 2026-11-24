# Kubernetes Lab 1: Multi-Node Cluster Setup & Application Deployment with Kind

This guide walks through creating a 3-node local Kubernetes cluster (`klu`) using **Kind (Kubernetes in Docker)**, verifying cluster health, deploying a multi-replica web application, testing self-healing and scaling, and exposing the service locally.

## Prerequisites

Before starting, ensure you have the following installed and running on your machine:

* [Docker](https://docs.docker.com/get-docker/) (Docker Desktop or Docker Engine)
* [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)

---

## Step 1: Create the Project Directory and Cluster Configuration

1. Create a working directory for the lab and navigate into it:

   ```bash
   mkdir -p ~/Documents/kuberna/k8s-lab1
   cd ~/Documents/kuberna/k8s-lab1
   ```

2. Create a configuration file named `kind-3node.yaml` to define a cluster with **1 control-plane node** and **2 worker nodes**:

   ```bash
   nano kind-3node.yaml
   ```

3. Paste the following configuration into `kind-3node.yaml`, then save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`):

   ```yaml
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   name: klu
   nodes:
     - role: control-plane
     - role: worker
     - role: worker
   ```

4. Create the cluster using the configuration file:

   ```bash
   kind create cluster --config kind-3node.yaml
   ```

5. Verify that all three nodes are up and in the `Ready` state:

   ```bash
   kubectl get nodes
   ```

![Step 1: Cluster Creation and Node Verification](Screenshot%202026-11-13%20at%2010.10.40%E2%80%AFAM.jpg)

---

## Step 2: Verify Docker Containers and System Pods

1. Check the running Docker containers to confirm each Kubernetes node is running as a container:

   ```bash
   docker ps
   ```

2. Inspect the pods running in the `kube-system` namespace to verify core Kubernetes components (CoreDNS, etcd, kindnet CNI, kube-apiserver, kube-proxy, scheduler, etc.):

   ```bash
   kubectl get pods -n kube-system
   ```

![Step 2: Docker Containers and kube-system Pods](Screenshot%202026-11-13%20at%2010.11.57%E2%80%AFAM.jpg)

---

## Step 3: Deploy an Application and Test Self-Healing

1. Create a deployment named `web` using the `nginx` image with **3 replicas**:

   ```bash
   kubectl create deployment web --image=nginx --replicas=3
   ```

2. Verify the deployment and check the running pods:

   ```bash
   kubectl get deployments
   kubectl get pods
   ```

3. View detailed pod information (including IP addresses and assigned worker nodes):

   ```bash
   kubectl get pods -o wide
   ```

4. Test Kubernetes self-healing by deleting one of the pods (replace `<pod-name>` with one of your actual pod names, e.g., `web-5fc9f4bf66-87fmn`):

   ```bash
   kubectl delete pod <pod-name>
   ```

5. Scale the deployment up to **5 replicas** and observe the new containers being created:

   ```bash
   kubectl scale deployment web --replicas=5
   kubectl get pods
   ```

![Step 3: Deployment, Self-Healing, and Scaling Up](Screenshot%202026-11-13%20at%2010.26.39%E2%80%AFAM.jpg)

---

## Step 4: Scale Down and Expose the Service

1. Scale the `web` deployment down to **2 replicas** and watch the excess pods terminate:

   ```bash
   kubectl scale deployment web --replicas=2
   kubectl get pods
   ```

2. Expose the `web` deployment internally as a ClusterIP service on port `80`:

   ```bash
   kubectl expose deployment web --port=80
   ```

3. Forward local port `8080` to port `80` of the `web` service so you can access it from your browser at `http://localhost:8080`:

   ```bash
   kubectl port-forward service/web 8080:80
   ```

   *(Press `Ctrl+C` when finished to stop port forwarding.)*

![Step 4: Scaling Down and Port-Forwarding Service](Screenshot%202026-11-13%20at%2010.26.54%E2%80%AFAM.png)

---

## Cleanup (Optional)

When you are finished with the lab, you can delete the Kind cluster to free up system resources:

```bash
kind delete cluster --name klu
```