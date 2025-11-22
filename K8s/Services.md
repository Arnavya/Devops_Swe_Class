==========================================================================

Lecture Title : Understanding Kubernetes Services

Lecture Date : 8th Jan 2026

==========================================================================

# Assignment : 

## ***Q1. Exposing a Deployment with Services in Kubernetes***

| Layer      | Command Purpose       |
| ---------- | --------------------- |
| Namespace  | Ensure correct scope  |
| Deployment | Desired replicas      |
| Pods       | Running state         |
| Service    | Type & ports          |
| NodePort   | External reachability |


### Step 1: Verify Namespace & Work Inside It

#### Verify namespace exists

```bash
kubectl get ns svc-lab1
```

#### Set default namespace

```bash
kubectl config set-context --current --namespace=svc-lab1
```
- Run with sudo if permission are missing.
- From now on, no `-n svc-lab1` needed.

### Step 2: Create Deployment (nginx-deploy, 3 replicas)

```bash
kubectl create secret docker-registry dockerhub-cred \
    --docker-username=YOUR_USERNAME \
    --docker-password=YOUR_PASSWORD \
    -n svc-lab1
```

#### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: svc-lab1
spec:
  replicas: 3
  selector:
    matchLabel:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      imagePullSecrets:
        - name: dockerhub-cred
      containers:
        - name: nginx-container
          image: nginx
          ports:
            - containerPort: 80
```

#### Apply & Verify
```bash
kubectl apply -f deployment.yaml

kubectl get deployment nginx-deploy

kubectl get pods

```

### Step 3: Expose Deployment as ClusterIP Service

#### Create ClusterIP Service

```bash
kubectl expose deployment nginx-deploy \
    --name nginx-clusterip \
    --port 80 \
    --target-port 80 \
    --type ClusterIP
```

### Step 4: Check Service Details

```bash
kubectl get svc nginx-clusterip
```

Expected:
```bash
TYPE        ClusterIP
```


### Step 5: Patch Service to NodePort (30080)

#### Why Patch Instead of Recreate the Service?
**Patch means:**
- Modify an existing resource
- Without deleting it
- Without changing selectors or endpoints

**This is real-world practice:**
- Zero downtime
- Faster updates
- Safer than delete + apply

```bash
kubectl patch svc nginx-clusterip -p '{
    "spec" : {
        "type" : "NodePort",
        "ports" : [{
            "port" : 80,
            "targetPort" : 80,
            "nodePort" : 30080
        }] 
    }
}'
```

### Step 6: Verify NodePort

```bash
kubectl get svc nginx-clusterip
```

Expected:
```bash
TYPE        NodePort
```

### YAML-based Service (service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
kubectl apply -f service.yaml
```

## ***Q2. Understanding Kubernetes Services***

### Step 1: Verify Namespace & Work Inside It

#### Verify namespace exists

```bash
kubectl get ns svc-lab
```

#### Set default namespace

```bash
sudo kubectl config set-context --current --namespace=svc-lab
```
- From now, no `-n svc-lab` needed.

### Step 2: Deployment YAML (web-deploy.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
  namespace: svc-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      imagePullSecrets:
        - name: dockerhub-secret
      containers:
        - name: nginx-container
          image: nginx
          ports:
            - containerPort: 80
```

#### Apply & Verify

```bash
kubectl apply -f web-deploy.yaml
```

```bash
kubectl get deployment
```

### Step 3: Service YAML (web-service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: svc-lab
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 80
```

#### Apply & Verify

```bash
kubectl apply -f web-service.yaml
```

```bash
kubectl get svc
```

### Step 4: Verify the SVC Endpoints

```bash
kubectl get endpoints web-service
```

## ***Q3. Exploring Kubernetes Service Types***

### Step 1: Verify Namespace

```bash
kubectl get ns svc-types-lab
```

#### Set default namespace

```bash
sudo kubectl config set-context --current --namespace=svc-types-lab
```
- From now, no `-n svc-types-lab` needed.

### Step 2: Deployment YAML (2 Replicas)

#### nginx-deploy.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: svc-types-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      imagePullSecrets:
        - name: dockerhub-secret
      containers:
        - name: nginx-container
          image: nginx
          ports:
            - containerPort: 80
```

#### Apply & Verify

```bash
kubectl apply -f nginx-deploy.yaml
```

```bash
kubectl get deployment
```

### Step 3: ClusterIP Service (Internal Access)

#### nginx-clusterip.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
  namespace: svc-types-lab
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

#### Apply & Verify

```bash
kubectl apply -f nginx-clusterip.yaml
```

```bash
kubectl get svc
```

### Step 4: NodePort Service (External Access)

#### nginx-nodeport.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
  namespace: svc-types-lab
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

- `nodePort` is automatically assigned if not specified.

#### Apply & Verify

```bash
kubectl apply -f nginx-nodeport.yaml
```

```bash
kubectl get svc
```

### Step 5: LoadBalancer Service (Cloud / Pending Locally)

#### nginx-lb.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  namespace: svc-types-lab
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

#### Apply & Verify

```bash
kubectl apply -f nginx-lb.yaml
```

```bash
kubectl get svc
```

### Step 6: Verify Endpoints (Most Important Concept)

```bash
kubectl get endpoints nginx-clusterip
```

```bash
kubectl get endpoints nginx-nodeport
```

```bash
kubectl get endpoints nginx-lb
```

**Expected:**
- Same 2 Pod IPs in all Services
- Confirms Services route to Pods via labels

### Final Summary (Conceptual)
| Service Type | Access Scope      | Purpose                     |
| ------------ | ----------------- | --------------------------- |
| ClusterIP    | Internal only     | Pod-to-Pod communication    |
| NodePort     | External (NodeIP) | Simple external access      |
| LoadBalancer | Cloud-managed     | Production external traffic |