===============================================

Lecture Title : Create Your Own Docker Images

Lecture Date : 14th Nov 2025 (Friday)

===============================================

## Q1. Docker File for a Java App

```
# Base Image
# FROM openjdk:latest

# The openjdk image has been deprecated from the Docker Hub
FROM eclipse-temurin:17-jdk

# Set the working directory
WORKDIR /app

# Copy the Java file to the working directory
COPY HelloWorld.java /app

# Compile the Java file
RUN javac HelloWorld.java

# Run the Java file
CMD ["java", "HelloWorld"]
```

### Build the Docker Image

```
docker build -t hello-world .
```

OR

```
docker build -t hello-world -f Dockerfile .
```

#### Command Breakdown:
- `-t`: Tag (name) of the image.
- `-f`: Path to the Dockerfile (if not named `Dockerfile`).
- `.`: The build context (current directory).

## Q2. Building a Docker Image

```
FROM alpine:latest

CMD ["echo", "Hello DevOps"]

```

Build the Image:

```
docker build -t hello-devops .
```

Run the Container:

```
docker run --rm hello-devops
```

Executes the container, prints Hello DevOps, and removes the container after it exits.

