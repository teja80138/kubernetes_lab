# Kubernetes Welcome ConfigMap Controller (`exp3-controller`)

A custom Kubernetes controller written in Go that watches for namespaces labeled with `klu.edu/managed=true` and automatically reconciles a `welcome` ConfigMap inside each managed namespace.

---

## 📋 Prerequisites

- A running Kubernetes cluster (e.g., Minikube, Kind, Docker Desktop, or K3s)
- `kubectl` configured to interact with your cluster
- Go installed (`1.20+` recommended)

---

## 🚀 Step 1: Start the Kubernetes Controller

Open a terminal inside the `exp3-controller` directory and start the controller:

```bash
go run main.go
```

The controller will initialize and wait for namespace events:
```text
welcome-controller started; waiting for work...
```

---

## 🧪 Step 2: Create & Label a Managed Namespace

Open a **second terminal** to test the controller behavior. Create a namespace named `team-a` and apply the `klu.edu/managed=true` label:

```bash
kubectl create namespace team-a
kubectl label namespace team-a klu.edu/managed=true
```

Verify that the controller automatically generated the `welcome` ConfigMap:

```bash
kubectl get configmap welcome -n team-a -o yaml
```

### Expected Output:
The ConfigMap `welcome` should exist in `team-a` with the message: `Welcome to team-a - managed by KLU controller`.

![Step 2 - Creating and verifying team-a ConfigMap]

---

## 🔄 Step 3: Verify Self-Healing (Auto-Recreation)

Delete the `welcome` ConfigMap manually to test the controller's reconciliation loop:

```bash
kubectl delete configmap welcome -n team-a
kubectl get configmap welcome -n team-a
```

Because `team-a` still carries the `klu.edu/managed=true` label, the controller immediately recreates the ConfigMap (`AGE: 0s`).

---

## 📦 Step 4: Scale Across Multiple Namespaces & Test Unlabeling

1. **Create and label additional team namespaces (`team-b` and `team-c`):**

```bash
kubectl create namespace team-b
kubectl label namespace team-b klu.edu/managed=true

kubectl create namespace team-c
kubectl label namespace team-c klu.edu/managed=true
```

2. **Remove the management label from `team-a`:**

```bash
kubectl label namespace team-a klu.edu/managed-
kubectl get configmap welcome -n team-a
```

![Step 3 & 4 - Self-healing, multi-namespace creation, and unlabeling]

---

## 👀 Step 5: Observe Controller Logs

Switch back to your first terminal running `go run main.go` to observe the reconciliation logs as namespaces are created, healed, and managed:

```text
welcome-controller started; waiting for work...
created ConfigMap welcome in namespace team-a
created ConfigMap welcome in namespace team-a
created ConfigMap welcome in namespace team-b
```

![Step 5 - Controller reconciliation logs]

---

## 🧹 Step 6: Cleanup

To remove all test namespaces and resources created during this walkthrough, run:

```bash
kubectl delete namespace team-a team-b team-c
```

Stop the controller in the first terminal by pressing `Ctrl + C`.
