===============================================

Lecture Title : Docker Volumes

Lecture Date : 19th Nov 2025

===============================================

## Q1. Docker Volumes

```bash

# Step 1 : Create Docker Volumes
# Create app_data volume with metadata
docker volume create --label env=lab app_data
# Create logs_data volume without metadata
docker volume create logs_data

# Verify Creations
docker volume ls

# Step 2: Inspect Docker Volumes
docker volume inspect app_data

docker volume inspect logs_data

# Step 3: Run app_container with Volume Mounted
docker run -d \
    --name app_container \
    -v app_data:/usr/share/nginx/html \
    nginx

# Verify container
docker ps

# Step 4: Run log_container with Logs Volume
docker run -dit \
    --name log_container \
    -v logs_data:/logs \
    alpine sh

# Verify container
docker ps

# Step 5: Write Data to app_data Volume
# Add content from one container and verify persistence.
docker exec app_container sh -c "echo 'Hello from app_container 1' > /usr/share/nginx/html/index.html"

# Verify
docker exec app_container cat /usr/share/nginx/html/index.html

# Step 6: Share Volume with Another Container
docker run -d \
    --name app_container_2 \
    -v app_data:/usr/share/nginx/html \
    nginx

# Verify shared data
docker exec app_container_2 cat /usr/share/nginx/html/index.html

# Step 7: List All Volumes
docker volume ls
```

```bash
# Post Lab Cleanup
docker volume rm app_data logs_data

# verify removal
docker volume ls

# Container Cleanup
docker container rm app_container app_container_2 log_container

# verify removal
docker container ls -a
```

---

# Pre-requisites: Docker Volumes (Concepts & Commands)

## Important Docker Volume Commands

- `docker volume create <volume_name>` → Create a volume
- `docker volume create --label key=value <volume_name>` → Create a volume with metadata (labels)
- `docker volume ls` → List all volumes
- `docker volume inspect <volume_name>` → Inspect a volume
- `docker volume rm <volume_name>` → Remove a volume

### Mounting Volumes into Containers
- Volumes are mounted using `-v` flag

Syntax:
```bash
# -v volume_name:container_path
docker run -v <volume_name>:<container_path> <image_name>
```

Example:
```bash
# -v volume_name:container_path
docker run -v app_data:/usr/share/nginx/html nginx
```

### Sharing Volumes Between Containers
- Same volume can be mounted into multiple containers
- Data written by one container is visible to others
- Enables data persistence and sharing

## Theory

### What is a Docker Volume?
- A Docker volume is **persistent storage** managed by Docker.
- Volumes exist **independent of container lifecycle**.
- Data in volumes is **not lost** when containers stop or are removed.

---

### Why Use Docker Volumes?
- Persist application data
- Share data between multiple containers
- Avoid data loss on container removal
- Recommended over bind mounts for portability

---

### Volume vs Container Filesystem
- Container filesystem → ephemeral (temporary)
- Volume → persistent and reusable
- Multiple containers can mount the **same volume**

---

### Volume Drivers
- Default driver: `local`
- `local` driver stores data on host (Docker-managed path)

---

### Volume Metadata (Labels)
- Volumes support **labels (key-value metadata)**
- Useful for environment tagging and inspection

Example:
```bash
docker volume create --label env=lab app_data
```

