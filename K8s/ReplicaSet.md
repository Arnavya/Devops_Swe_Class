==============================================================

Lecture Title : Deep dive into Kubernetes ReplicatSet

Lecture Date : 3rd Dec 2025

==============================================================

# ***Q1. Replicaset in Kubernetes***

### Task 1 : Create a Namespace (replica-set-namespace.yaml)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: replica-set-namespace
```

#### Apply & Verify

```bash
kubectl get namespaces

kubectl apply -f replica-set-namespace.yaml

kubectl get namespaces
```

### Task 2: Create a Pod with a Label (labeled-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  namespace: replica-set-namespace
  labels:
    app: my-app
spec:
  containers:
    - name: nginx-container
      image: nginx
      ports:
        - containerPort: 80
```

#### Apply & Verify

```bash
kubectl get pods -n replica-set-namespace

kubectl apply -f labeled-pod.yaml

kubectl get pods -n replica-set-namespace
```

### Task 3: Create a Replica Set (replica-set.yaml)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-replicaset
  namespace: replica-set-namespace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx-container
          image: nginx
          ports:
            - containerPort: 80
```

#### Apply & Verify

```bash
# Check ReplicaSet
kubectl get rs -n replica-set-namespace

# Check Pods
kubectl get pods -n replica-set-namespace

# Describe ReplicaSet
kubectl describe rs my-app-replicaset -n replica-set-namespace
```

---

# Pre-Requisite Concepts (ReplicaSet)

This section explains the minimum concepts required to correctly understand and write ReplicaSet YAMLs.

## 0. ReplicaSet Structure

```
ReplicaSet
├── apiVersion
├── kind
├── metadata
│   ├── name
│   ├── namespace
│   └── labels (optional)
└── spec
    ├── replicas
    ├── selector
    │   └── matchLabels
    └── template
        ├── metadata
        │   └── labels
        └── spec
            ├── containers
            │   ├── name
            │   ├── image
            │   ├── ports
            │   ├── env (optional)
            │   └── volumeMounts (optional)
            ├── volumes (optional)
            ├── restartPolicy (optional)
            └── nodeSelector / tolerations (optional)
```

## 1. `apiVersion: v1` vs `apps/v1`

| Resource        | apiVersion |
|-----------------|------------|
| Pod             | `v1`       |
| Namespace       | `v1`       |
| Service         | `v1`       |
| ReplicaSet      | `apps/v1`  |
| Deployment      | `apps/v1`  |

**Why?**
- Core resources live in `v1`
- Workload controllers (ReplicaSet, Deployment) live in the `apps` API group

❌ This is invalid:
```yaml
apiVersion: v1
kind: ReplicaSet
```

✅ Correct:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
```

## `labels` vs `matchLabels` (Pod Adoption)

#### `labels`
- Applied to Pods
- Used for identification

```yaml
metadata:
  labels:
    app: my-app
```

#### `matchLabels`
- Used by ReplicaSet
- Selects Pods to manage

```yaml
selector:
  matchLabels:
    app: my-app
```

**Important Concept — Pod Adoption**
- If a Pod already exists
- And its labels match matchLabels
- ReplicaSet adopts that Pod automatically

**This is why:**
- Labels must match exactly
- Selector cannot be changed later

## 3. Why Labels Do NOT Use `-`?

> **Reason:** Labels are a MAP (key → value), not a list.

✅ Correct:

```yaml
labels:
  app: my-app
  env: prod
```

❌ This is invalid:
```yaml
labels:
  - app: my-app
  - env: prod
```

**Rule:**
- `-` → Lists (containers, ports, env)
- No `-` → Maps (labels, annotations, selectors)