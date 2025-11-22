=========================================================

Lecture Title : Deep dive into Kubernetes Pods

Lecture Date : 1st Dec 2025

=========================================================

# Assignment Questions Solutions

## ***Q1. Kubernetes Namespace***

```bash
kubectl get namespaces \
    --no-headers \
    -o custom-columns=NAME:.metadata.name \
    > answer.txt
```

## ***Q2. Pods in Kubernetes***

### PHASE 1 : Namespace YAML (namespace.yaml)

**Why Namespace YAML first?**
- Kubernetes objects are **declared**, not created imperatively
- Namespace must exist before creating resources inside it

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: assignment-namespace
```

#### Apply & Verify

```bash
kubectl apply -f namespace.yaml
kubectl get namespaces
```

### PHASE 2 : Multi-Container Pod YAML (web-server-pod.yaml)

**Pod contains:**
- 2 containers
- 1 shared volume (emptyDir)

**Container responsibilities:**
| Container       | Purpose                |
| --------------- | ---------------------- |
| nginx-container | Serve web + write logs |
| log-sidecar     | Read (tail) nginx logs |

**Shared resource:**
```
emptyDir -> mounted at /var/log/nginx
```

#### To Avoid ImagePullBackOff / DockerHub Rate Limiting Error

```bash
kubectl create secret docker-registry dockerhub-secret \
    --docker-username=YOUR_USER_NAME \
    --docker-password=YOUR_PASSWORD \
    --docker-email=YOUR_EMAIL \
    -n assignment-namespace
```

#### Multi-Pod Containers Sharing the same Volume
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: web-server-pod
  namespace: assignment-namespace
spec:
  imagePullSecrets:
    - name: dockerhub-secret

  volumes:
    - name: shared-log
      emptyDir: {}

  containers:
    - name: nginx-container
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - name: shared-log
          mountPath: /var/log/nginx
    - name: log-sidecar
      image: busybox
      command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
      volumeMounts:
        - name: shared-log
          mountPath: /var/log/nginx
```

#### Apply & Verify
```bash
kubectl apply -f web-server-pod.yaml
```

Check Pod Status:
```bash
kubectl get pods -n assignment-namespace
```

Expected:
```
web-server-pod   Running
```

Verify both Containers:
```bash
kubectl describe pod web-server-pod -n assignment-namespace
```

Look for:
```
Containers:
  nginx-container   Running
  log-sidecar       Running
```

### PHASE 3 : Access Application & Verify Logs

```bash
kubectl port-forward pod/web-server-pod 8080:80 -n assignment-namespace &
```

**Why `&`?**
- `&` runs the command in the background
- Allows you to use the terminal for other commands
- Without `&`, the terminal would be blocked

Now Open:
```
http://localhost:8080
```

OR via CLI:

```
curl http://localhost:8080
```

> ✅ You should see Nginx welcome page

This generates **access logs**.

#### View Logs from Sidecar Container

```bash
kubectl logs web-server-pod -c log-sidecar -n assignment-namespace
```

Expected Output:
```
GET / HTTP/1.1" 200
```

---

# Common Mistakes & Correct Usage (`kubectl get`)

This section documents common confusions while working with `kubectl get`, especially related to headers and custom output formatting.

---

### 1. `--no-headers` Flag Confusion

#### ❌ Mistake
```bash
kubectl get namespaces --no-headers=true
```
- While this works, it creates confusion for beginners.
- It makes `--no-headers` look like it requires a value.

#### ✅ Correct & Recommended Usage

```bash
kubectl get namespaces --no-headers
```

**Explanation:**

- `--no-headers` is a boolean flag
- Writing it alone automatically means true
- This is the standard and clean syntax used in scripts and exams

---

### 2. `custom-columns` Syntax Confusion

#### ❌ Mistake 1: Missing JSON Path
```bash
kubectl get namespaces -o custom-columns=NAME
```

**Why this failed:**
- `custom-columns` requires both:
    ```bash
    COLUMN_NAME:JSON_PATH
    ```

#### ❌ Mistake 2: Using Invalid Field

```bash
kubectl get namespaces -o custom-columns=NAME:name
```

OR

```bash
kubectl get namespaces -o custom-columns=NAME:.name
```

**Why this failed:**
- Kubernetes objects do not have `.name`
- The name is always stored under `.metadata.name`

#### ✅ Correct Syntax

```bash
kubectl get namespaces --no-headers -o custom-columns=NAME:.metadata.name
```

**Explanation:**
- `NAME` → column header
- `.metadata.name` → correct JSON path for resource names

### 3. Why `.metadata.name` Is Mandatory
All Kubernetes resources follow this structure:

```yaml
metadata:
  name: resource-name
```

**That is why:**
- `.metadata.name` ✅ works
- `.name` ❌ does not exist

### 4. Understanding COLUMN_NAME vs JSON_PATH

When using `-o custom-columns`, kubectl is **formatting raw JSON data into a table**.

The syntax is always:

```bash
COLUMN_NAME:JSON_PATH
```

**This means:**
- `COLUMN_NAME` → What the column should be called in the output (for humans)
- `JSON_PATH` → Where the actual data is located inside the Kubernetes object

**Example Breakdown**
```bash
-o custom-columns=NAME:.metadata.name
```

- `NAME`
    - This is just a label shown in the output table
    - You can change it to anything (e.g., `NS_NAME`, `NAMESPACE`)

- `.metadata.name`
    - This tells kubectl where to fetch the value from inside the object

Kubectl does **not guess**:

- what data you want
- what you want to call it

So both parts are mandatory.

**Analogy (SQL-style Thinking)**
```sql
SELECT metadata.name AS NAME FROM namespaces;
```

---

# Basic kubectl get Commands

```bash
# List all namespaces
kubectl get namespaces

# List pods in current namespace
kubectl get pods

# List pods in all namespaces
kubectl get pods -A

# Get detailed pod listing
kubectl get pods -o wide

# Get services
kubectl get services

# Get deployments
kubectl get deployments

# Get nodes in the cluster
kubectl get nodes

# Get resources without headers (script-friendly)
kubectl get pods --no-headers

# Get only resource names
kubectl get pods -o name
```