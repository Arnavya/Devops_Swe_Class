## What is YAML?
- YAML stands for **YAML Ain’t Markup Language**
- It is used for **configuration files**
- Kubernetes uses YAML to define resources

## Basic Rules of YAML

### 1. Indentation Matters
- YAML uses **spaces**(2 spaces), not tabs
- Same level = same indentation

```yaml
spec:
  containers:
    - name: nginx
```

### 2. Key : Value Format
```yaml
key: value
```

Example:
```yaml
name: my-pod
namespace: default
```

## When Do We Use `-` (Dash)?

> Rule: Use - when the value is a LIST (array)

#### Example 1: List of Containers
- Each `-` means one item in the list
```yaml
containers:
  - name: nginx
    image: nginx
  - name: sidecar
    image: busybox
```

#### Example 2: List with Single Item
```yaml
ports:
  - containerPort: 80
```
> Even one item still needs - because it’s a list.

### ❌ Wrong (Missing `-`)

```yaml
containers:
  name: nginx
  image: nginx
```

> This is invalid because containers must be a list.

## When NOT to Use `-`

> Rule: Do NOT use - for single objects (maps)

#### Example: Metadata
```yaml
metadata:
  name: web-server-pod
  namespace: assignment-namespace
```

> ❌ No `-` because metadata is one object, not a list.

#### How to Identify List vs Object (Easy Trick)
| Type   | How it looks   | Uses `-` |
| ------ | -------------- | -------- |
| Object | key: value     | ❌ No     |
| List   | multiple items | ✅ Yes    |

#### Common Kubernetes Fields That Are Lists

```yaml
containers:
ports:
volumeMounts:
volumes:
imagePullSecrets:
env:
```

> These always use -

## Example Pod Skeleton

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: app
      image: nginx
```
