Kubernetes Demonstration on AKS

# Introduction

This document details demonstration of key Kubernetes features on **Azure Kubernetes**

**Service (AKS)**. It includes detailed steps, YAML configurations, and commands. The focus was on understanding ephemeral containers, init containers, volume-per-pod deployments, blue- green deployments, and high availability using Zone Redundant Storage (ZRS).

**Core Concepts**

Here are the concepts:

1. **Ephemeral Containers**: Temporary containers added to existing pods for debugging and troubleshooting without modifying the operational container.
2. **Init Containers**: Containers that run before the main application container to perform setup tasks like testing disk or network performance.
3. **Volume-per-Pod Deployment**: Provisioning individual persistent volumes for each pod without using StatefulSets.
4. **Blue-Green Deployment**: Deploying two application versions (Blue and Green) and switching traffic seamlessly between them using a load balancer.
5. **Taking a Pod Out of Service**: Scaling down or cordoning pods in a StatefulSet or Deployment to test workload redistribution.
6. **ZRS with Failover**: Demonstrating storage resilience by replicating data across zones and testing failover recovery.

**Steps and Execution**

# Step 1: Ephemeral Containers

**Objective**: Use ephemeral containers for debugging without affecting the operational containers in a pod.

## Adding an Ephemeral Container

used the following command to add an ephemeral container to a running pod:

> kubectl debug -it $(kubectl get pods -l app=sample-app -o name) -- image=busybox --target=main-app -- sh

## Inside the Ephemeral Container

Once inside the container, verified its functionality by checking available tools:

> which curl exit

## Verification

described the pod to confirm the ephemeral container was attached:

> kubectl describe pod &lt;pod-name&gt;

\- The \`Ephemeral Containers\` section confirmed its successful attachment.

# Step 2: Init Containers

**Objective**: Set up an init container to test disk performance before running the main application.

## Creating a PersistentVolumeClaim

created a PVC to provision the storage for the init container.

## File

### pvc-init-test.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: init-test-pvc
spec: null
accessModes:
  - ReadWriteOnce
resources:
  requests: null
  storage: 5Gi

```
Command:

> kubectl apply -f pvc-init-test.yaml

## Deployment with Init Container

defined a deployment that included an init container to test disk performance.

## File

init-container-test.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: init-container-test
spec: null
replicas: 1
selector:
  matchLabels:
    app: init-container-test
template:
  metadata:
    labels:
      app: init-container-test
  spec:
    volumes:
      - name: init-test-volume
persistentVolumeClaim: null
claimName: init-test-pvc
initContainers:
  - name: disk-perf-test
image: busybox
command:
  - sh
  - '-c'
  - >-
    dd if=/dev/zero of=/mnt/volume/testfile bs=1M count=100 && echo 'Disk
    performance test completed'
volumeMounts:
  - name: init-test-volume
mountPath: /mnt/volume
containers:
  - name: main-container

```


Command:

> kubectl apply -f init-container-test.yaml

## Logs and Verification

checked the logs of the init container to ensure the disk test ran successfully:

> kubectl logs &lt;pod-name&gt; -c disk-test

The output confirmed the performance test was complete.

# Step 3: Volume-per-Pod Deployment

**Objective**: Dynamically provision individual persistent volumes for each pod without using StatefulSets.

## Create a PersistentVolumeClaim: File

dynamic-pvc.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
  labels:
    app: volume-per-pod
spec:
  accessModes:
    - ReadWriteOnce resources: null
requests:
  storage: 5Gi
  storageClassName: standard

```

\- Command:

> kubectl apply -f dynamic-pvc.yaml

1. **Create Pods**:

## File

pod-with-volume.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  labels:
    app: volume-per-pod
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: storage
volumes:
  - name: storage
    persistentVolumeClaim: null
    claimName: dynamic-pvc
---
apiVersion: v1
kind: Pod
metadata: null
name: pod-2
labels:
  app: volume-per-pod
spec:
  containers:
    - name: nginx
      image: nginx
volumeMounts:
  - mountPath: /usr/share/nginx/html
    name: storage
volumes:
  - name: storage
persistentVolumeClaim:
  claimName: dynamic-pvc
```


Command:

> ```kubectl apply -f pod-with-volume.yaml ```

## Verification

logged into

to check the mounted volume:

pod-1

kubectl exec -it pod-1 -- /bin/sh cd /usr/share/nginx/html

echo "Pod 1 file" > testfile.txt

The file persisted across pod restarts.

# Step 4: Blue-Green Deployment

**Objective**: Deploy two versions of an application (Blue and Green) and switch traffic between them using a LoadBalancer Service.

## Create Blue Deployment

\- created a deployment for the Blue & Green version and applied it:

blue-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo-app
    version: blue
  name: blue-app
replicas: 2
selector:
  matchLabels:
    app: demo-app
    version: blue
spec:
  template:
    metadata:
      labels:
        app: demo-app
        version: blue
    spec:
      agentpool: nodepool1
      containers:
      - image: nginx:1.19
        name: blue-app
    nodeSelector: 
      ports:
      - containerPort: 80

```
green-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-app
  labels:
    app: demo-app
    version: green
spec:
  replicas: 2
  selector: null
matchLabels:
  app: demo-app
  version: green
template:
  metadata:
    labels:
      app: demo-app
      version: green
    spec:
      nodeSelector:
        agentpool: greenpool
        containers:
        - name: green-app
            image: 'nginx:1.20'
            ports:
            - containerPort: 80

```

## Switch Traffic to Green

After deploying Green, updated the service selector to point traffic to the Green deployment:

> kubectl patch service demo-service -p '{"spec":{"selector":{"app":"demo- app","version":"green"}}}'

## Verification

accessed the external IP of the LoadBalancer to confirm traffic routing.

# Step 5: Taking a Pod Out of Service

**Objective**: Remove a pod from service traffic. statefulset.yaml
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: test-statefulset
spec: null
serviceName: test-service
replicas: 3
selector: 
  matchLabels:
    app: test-app
template:
  metadata: null
  labels:
    app: test-app
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```
## Scaling Down

scaled down the StatefulSet:

> kubectl scale statefulset test-statefulset --replicas=2

## Verification

confirmed that the removed pod was no longer running:

> kubectl get pods

# Step 6: ZRS and Failover

**Objective**: Test Zone Redundant Storage (ZRS) for resilience and data availability during failover.

## Create a ZRS PVC

**File**:

pvc-zrs.yaml
```yaml
apiVersion: storage.k8s.io/v1 kind: StorageClass

metadata:

name: zrs-storageclass provisioner: disk.csi.azure.com parameters:

skuName: Premium_ZRS reclaimPolicy: Retain

volumeBindingMode: WaitForFirstConsumer
```
Command:

> kubectl apply -f pvc-zrs.yaml

1. **Attach to Pod**: pod-zrs.yaml
```yaml
apiVersion: v1 kind: Pod metadata:

name: zrs-pod spec:

containers:

\- name: nginx image: nginx volumeMounts:

\- mountPath: "/usr/share/nginx/html"

name: zrs-volume volumes:

\- name: zrs-volume persistentVolumeClaim:

claimName: zrs-pvc
```
created a pod that used the ZRS PVC and applied it:

> kubectl apply -f pod-zrs.yaml

## Simulate Failover

wrote a test file, deleted the pod, and recreated it. The file persisted across the failover:

> kubectl exec -it zrs-pod -- /bin/sh cat /usr/share/nginx/html/testfile.txt

