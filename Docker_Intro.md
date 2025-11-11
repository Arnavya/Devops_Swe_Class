=======================================

Lecture Title : Intro to Docker

Lecture Date : 10th Nov 2025

=======================================

## Q2. Creating Docker Container

```bash
docker pull postgres

docker run -d \
  --name my-postgres \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydatabase \
  -p 5432:5432 \
  postgres

docker ps

docker exec -it my-postgres bash

# Inside the container, access PostgreSQL
psql -U myuser -d mydatabase

CREATE TABLE test(id SERIAL PRIMARY KEY, name VARCHAR(50));
INSERT INTO test(name) VALUES ('John Doe');
SELECT * FROM test;
```

## Q3. Docker Stop Container

```bash
docker ps

# Stopping the running container
docker stop my-postgres
docker ps -a

# Removing the container instance
docker rm my-postgres
docker ps -a

# Remove the image
docker images
docker rmi postgres
# OR
docker image rm postgres

docker images
```