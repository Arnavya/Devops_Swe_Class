=========================================

Lecture Title : Docker Containers

Lecture Date : 17th Nov 2025 

=========================================

## Q1. Namespaces and Isolation

```bash
# Starts a container named isotest with hostname set to mycontainer.
docker run -dit --name isotest --hostname mycontainer ubuntu bash

# Check container hostname:
docker exec isotest hostname

# Check host hostname:
hostname
```

## Q2. Resource Limits with cgroups

```bash
# Runs container with 200MB RAM and half a CPU.
docker run -dit --name limit-test --memory=200m --cpus=0.5 busybox sh

# Don't use bash. It is not present in the image. Use sh instead.

# Start workload inside container:
docker exec -d limit-test yes "CPU stress test"

# Check container stats:
docker stats limit-test

```

Inspecting:

```bash
# Check container memory usage:
docker exec limit-test free -m

# Check container CPU usage:
docker exec limit-test top

# Stop container:
docker stop limit-test
```