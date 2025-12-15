================================================================

Lecture Title : Understanding HPA

Lecture Date: 15th Dec 2025 (Monday)

================================================================

# Q1. HorizontalPodAutoscaler (HPA)

## Core Concepts
- HPA automatically scales Pods based on metrics (CPU here).
- It targets a controller (Deployment / ReplicaSet).
- Scaling range is controlled using:
    - `minReplicas`
    - `maxReplicas`
- Stabilization window prevents rapid scale down (flapping).

### Set the Namespace Context
```bash
kubectl get ns
```

```bash
sudo kubectl config set-context --current --namespace=autoscale
```

### Verify the Deployment
```bash
kubectl get deploy apache-server
```

### HPA YAML (apache-server-hpa.yaml)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: apache-server
  namespace: autoscale
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: apache-server
  minReplicas: 1
  maxReplicas: 4
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 30
```

#### Apply the HPA
```bash
kubectl apply -f apache-server-hpa.yaml
```

#### Verify the HPA
```bash
kubectl get hpa
```

## Why `apiVersion: autoscaling/v2`?

- `autoscaling/v1` does not support specifying the downscale stabilization window.
- It only supports the minimal spec fields (minReplicas, maxReplicas, targetCPUUtilizationPercentage, scaleTargetRef).

## Why `scaleTargetRef`? Why not `selector`?

- The HPA does NOT use selectors directly because it targets a specific controller object (like a Deployment, StatefulSet, or ReplicaSet) — not the Pods themselves.
- You specify the target with the scaleTargetRef, which points exactly to the controller resource by kind and name.
- The controller (e.g., Deployment) already manages Pods using selectors internally. So the HPA simply tells the controller how many replicas to maintain, and the controller uses its selector to manage Pods.
- If HPA had selectors, it would add complexity and ambiguity because multiple controllers or Pods could match. Instead, it keeps a clear, direct link to the Deployment.

### In Shot:

| Object                  | Uses Selectors?         | Reason                                        |
| ----------------------- | ----------------------- | --------------------------------------------- |
| Deployment              | Yes                     | To manage Pods it owns                        |
| HorizontalPodAutoscaler | No (targets Deployment) | Targets controller by name, not Pods directly |