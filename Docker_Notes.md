# Docker Quick Reference (Short Notes)

## Table of Contents

1. [Common `docker run` Options](#1-common-docker-run-options)
2. [Running Containers](#2-running-containers)
3. [Listing Containers](#3-listing-containers)
4. [Accessing a Running Container](#4-accessing-a-running-container)
5. [Stopping & Removing Containers](#5-stopping--removing-containers)
6. [Removing Images](#6-removing-images)
7. [Stats](#7-stats)
8. [Inspecting & Debugging Containers](#8-inspecting--debugging-containers)
9. [Help & Documentation](#9-help--documentation)
10. [Container Stress Test](#10-container-stress-test)
   - [CPU Stress Using Infinite Loop](#101-cpu-stress-using-infinite-loop)
   - [Running Stress While Monitoring Stats](#102-running-stress-while-monitoring-stats)



## 1. Common `docker run` Options
- `-e` → Set environment variables inside the container
- `-p` → Map host port to container port (`host:container`)
- `-d` → Run container in detached (background) mode
- `-it` → Interactive terminal (`-i` + `-t`)
- `--name` → Assign a custom name to the container
- `--hostname` → Assign a custom hostname to the container
- `--memory` → Set memory limit for the container (e.g., --memory=200m for 200MB)
- `--cpus` → Set CPU limit for the container (e.g., --cpus=0.5 for half a CPU)

## 2. Running Containers
- `docker pull <image_name>` → Download image from Docker Hub
- `docker run <image_name>` → Run a container from an image
- `docker run -it <image_name> /bin/bash` → Run container with interactive shell

## 3. Listing Containers
- `docker ps` → List running containers
- `docker ps -a` → List all containers (running + stopped)

## 4. Accessing a Running Container
- `docker exec -it <container_name> /bin/bash` → Open shell inside running container
- `docker exec <container_name> <command>` → Execute a command inside a running container

## 5. Stopping & Removing Containers
- `docker stop <container_name>` → Stop a running container
- `docker rm <container_name>` → Remove a stopped container

## 6. Removing Images
- `docker rmi <image_name>` → Remove a Docker image

## 7. Stats 
- `docker stats <container_name>` → Show resource usage of a container

## 8. Inspecting & Debugging Containers

- `docker inspect <container_name>` → View detailed container configuration (JSON)
- `docker inspect --format '{{.State.Status}}' <container_name>` → Check container status
- `docker inspect --format '{{.HostConfig.Memory}}' <container_name>` → Check memory limit

- `docker logs <container_name>` → View container logs
- `docker logs -f <container_name>` → Follow live logs
- `docker logs --tail 50 <container_name>` → Show last 50 log lines

## 9. Help & Documentation
- `docker --help` → Docker CLI help
- `docker run --help` → Help for `docker run`
- `docker image --help` → Image-related commands
- `docker container --help` → Container-related commands

## 10. Container Stress Test

This section explains how to generate load inside a container to observe CPU and memory limits.

---

### 10.1 CPU Stress Using Infinite Loop

#### Method 1: Using an Infinite Loop
Runs a no-operation continuously and consumes CPU.

```sh
while true; do :; done
```
- `true` → always true (infinite)
- `:` → no-op command (still uses CPU)
- `;` → shell operator that runs the commands in sequence 

Stop with: `Ctrl + C`

#### Method 2: Using yes Command
Prints output endlessly and heavily stresses CPU.

```sh
yes "CPU stress test"
```

Stop with: `Ctrl + C`

---

### 10.2 Running Stress While Monitoring Stats

#### Way 1: Detach from Container (Mac / Linux)
0. Get Inside the container:

```sh
docker exec -it limit-test sh
```

1. Start stress inside container:

```sh
while true; do :; done
```

2. Detach without **stopping container**:
- Do not press `Ctrl + C`
- Press `Ctrl + P` followed by `Ctrl + Q`

3. From host, check stats:

```sh
docker stats limit-test
```

#### Way 2: Using docker exec (Recommended)

Run stress from host without attaching:

```sh
docker exec limit-test sh -c "while true; do :; done"
```

OR using `yes` command:

```sh
docker exec limit-test yes "CPU stress test"
```

**Note:** The `-c` option is used when a command contains shell syntax such as loops or conditionals, which must be interpreted by a shell. Standalone executables like `yes` can be run directly without `-c`.

Check stats from host:

```sh
docker stats limit-test
```

### Summary
- CPU overuse → throttled
- Memory overuse → container killed
- Docker enforces limits using Linux cgroups