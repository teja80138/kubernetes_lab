# Kubernetes Lab 4: `kube-scheduler` Internals & Pod Scheduling Constraints

This lab explores how the Kubernetes scheduler (`kube-scheduler`) makes placement decisions—both by inspecting the upstream Kubernetes Go source code (`v1.37.0`) and by testing real-world scheduling behaviors on a multi-node Kind cluster (`kind-klu`), including resource requests, `nodeSelector`, taints and tolerations, `hostPort` conflicts, and cordoning nodes.

## Prerequisites

* A running multi-node **Kind** cluster (`kind-klu`) with one control-plane node (`klu-control-plane`) and two worker nodes (`klu-worker`, `klu-worker2`)
* `kubectl` configured to use the `kind-klu` context
* `git` installed on your machine

## Step 1: Verify Cluster Context, Node Taints, and Sparse-Clone Kubernetes Source

1. Switch `kubectl` to the `kind-klu` context, list the nodes, and check the default taint on the control-plane node:

   ```bash
   kubectl config use-context kind-klu
   kubectl get nodes
   kubectl describe node klu-control-plane | grep "Taints"
   ```

2. Perform a shallow sparse clone of the Kubernetes `v1.37.0` repository into `~/k8s-src`:

   ```bash
   git --version
   pwd
   git clone --depth 1 --branch v1.37.0 --filter=blob:none --sparse https://github.com/kubernetes/kubernetes.git k8s-src
   ```

   ![Cluster Context and Sparse Clone](1_2.jpg)

3. Navigate into `k8s-src` and configure `git sparse-checkout` to fetch only the scheduler packages:

   ```bash
   cd k8s-src
   git sparse-checkout set pkg/scheduler staging/src/k8s.io/kube-scheduler
   ```

   ![Sparse Checkout Scheduler Packages](2.png)

## Step 2: Inspect Default Scheduler Plugins & Scheduling Cycle Source Code

1. List the contents of `pkg/scheduler` and inspect the default registered scheduler plugins in `default_plugins.go`:

   ```bash
   ls pkg/scheduler
   open -a TextEdit pkg/scheduler/apis/config/v1/default_plugins.go
   grep -n "Name: names" pkg/scheduler/apis/config/v1/default_plugins.go
   ```

   ![Inspect Default Scheduler Plugins](3_2.jpg)

2. Search across `pkg/scheduler/framework/plugins` for all `Filter` and `Score` plugin method implementations:

   ```bash
   grep -rnE --include="*.go" --exclude="*_test.go" "^func \(.*\) (Filter|Score)\(" pkg/scheduler/framework/plugins
   ```

   ![Search Filter and Score Plugin Methods](4_2.jpg)

3. Locate the core scheduling and binding cycle entry points inside `pkg/scheduler/schedule_one.go`:

   ```bash
   grep -nE "^func .*(ScheduleOne|schedulingCycle|bindingCycle|schedulePod|findNodesThatFitPod|prioritizeNodes)\(" pkg/scheduler/schedule_one.go
   ```

   ![Locate Core Scheduling Cycle Entry Points](5.jpg)

## Step 3: Test Unschedulable CPU Resource Requests (`big.yaml`)

1. Navigate to your lab workspace (`~/Documents/kuberna/lab4`) and create `big.yaml` requesting `64` CPUs (which exceeds the capacity of all worker nodes):

   ```bash
   cd ~/Documents/kuberna
   mkdir -p lab4
   cd lab4
   nano big.yaml
   ```

   **`big.yaml`:**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: big
   spec:
     containers:
       - name: app
         image: nginx:1.27
         resources:
           requests:
             cpu: "64"
   ```

2. Apply `big.yaml` and inspect the pod status and `FailedScheduling` event:

   ```bash
   kubectl apply -f big.yaml
   kubectl describe pod big
   ```

   ![Pod Big Pending Status](6.png)

   *(Notice the `FailedScheduling` event: `0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 Insufficient cpu`.)*

   ![Pod Big FailedScheduling Event](7.png)

## Step 4: Test `nodeSelector` & Node Labeling (`ssd.yaml`)

1. Create `ssd.yaml` with a `nodeSelector` requiring `disktype: ssd`, then apply and describe the pod:

   ```bash
   nano ssd.yaml
   ```

   **`ssd.yaml`:**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: needs-ssd
   spec:
     nodeSelector:
       disktype: ssd
     containers:
       - name: app
         image: nginx:1.27
   ```

   ```bash
   kubectl apply -f ssd.yaml
   kubectl describe pod needs-ssd
   ```

2. Because no node has `disktype=ssd` yet, the scheduler logs `2 node(s) didn't match Pod's node affinity/selector`. Label `klu-worker` with `disktype=ssd` and verify that `needs-ssd` schedules onto `klu-worker`. You can also verify the exact error string in `node_affinity.go`:

   ```bash
   kubectl label node klu-worker disktype=ssd
   kubectl get pod needs-ssd -o wide
   NODE_AFFINITY_FILE=$(find ~ -name "node_affinity.go" 2>/dev/null | head -n 1)
   grep -n "didn't match" "$NODE_AFFINITY_FILE"
   ```

   ![Node Label and NodeAffinity Source Check](8.jpg)

## Step 5: Test Taints, Tolerations, & `hostPort` Scheduling

1. Taint `klu-worker` (`kubectl taint node klu-worker exam=running:NoSchedule`) and deploy `spread` with 4 replicas—all 4 pods initially land on `klu-worker2`.

2. Update the deployment via `spread-tol.yaml` to add a toleration for `exam=running:NoSchedule`, allowing pods to spread across both `klu-worker` and `klu-worker2`.

3. Remove the taint from `klu-worker` and apply `hostport.yaml` to test `NodePorts` filtering:

   ```bash
   kubectl create deployment spread --image=nginx:1.27 --replicas=4
   kubectl get pods -l app=spread -o wide
   nano spread-tol.yaml
   kubectl apply -f spread-tol.yaml
   kubectl get pods -l app=spread -o wide
   kubectl taint node klu-worker exam=running:NoSchedule-
   nano hostport.yaml
   kubectl apply -f hostport.yaml
   ```

   ![Taints, Tolerations, and HostPort Deployment](9.jpg)

## Step 6: Test Cordoned/Unschedulable Nodes (`closed`) & Clean Up

1. After cordoning the worker nodes (`kubectl cordon klu-worker klu-worker2`), deploy a pod named `closed`. Because the control plane has an untolerated taint and both worker nodes are marked unschedulable, `closed` stays `Pending` with `FailedScheduling: 0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) were unschedulable`.

2. Uncordon the nodes (`kubectl uncordon klu-worker klu-worker2`) and clean up all lab resources and labels:

   ```bash
   kubectl get pod closed -o wide
   kubectl delete pod big needs-ssd closed
   kubectl delete deployment spread hostport
   kubectl label node klu-worker disktype-
   kubectl get pods
   ```

   ![Unschedulable Nodes and Cleanup](11.jpg)
   ![Unschedulable Nodes and Cleanup Copy](11_2.jpg)