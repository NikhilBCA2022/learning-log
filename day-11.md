# Spring Boot Demo: Docker and Compose

A Spring Boot (Java 21, Maven) demo app packaged as a multi-stage Docker image and run with Docker Compose alongside a PostgreSQL container.

## What's in this repo

| File | Purpose |
|---|---|
| `Dockerfile` | Multi-stage build: compiles the app with Maven, then copies only the jar into a slim Java 21 runtime image |
| `docker-compose.yml` | Runs the app and a PostgreSQL 16 database on a private network, with a named volume for database data |

## Architecture

```
 Host :8082  --->  app container :8080  (Spring Boot / Tomcat)
                        |
                        |  private Compose network, reached by service name: db
                        v
                   db container :5432  (postgres:16-alpine)
                        |
                        v
                   named volume: db-data
```

- The `app` port is published to the host (`8082:8080`).
- The `db` port is **not** published. Only containers on the Compose network can reach it.
- The app reaches the database by its service name, for example `jdbc:postgresql://db:5432/<dbname>`. Compose's built-in DNS maps `db` to the container's current IP, so nothing depends on IP addresses.

## Prerequisites

- Docker Engine with the Compose plugin (`docker compose version` should work)
- Port `8082` free on the host

## Run it

```bash
# start everything in the background
docker compose up -d

# check status
docker compose ps

# follow the app logs
docker compose logs -f app
```

Test the app:

```bash
curl -i http://localhost:8082
```

The app has no route at `/`, so a `404` JSON response from Spring Boot means the port mapping and the app are working.

## Stop it

```bash
# stop and remove containers and the network, keep the database volume
docker compose down

# also delete the database volume (data is lost)
docker compose down -v
```

## Docker image

The image is published on Docker Hub:

```bash
docker pull nikhilj1402/javapp:v1
```

Build it locally instead:

```bash
docker build -t nikhilj1402/javapp:v1 .
```

## Verified behaviour

- **Name-based networking:** `docker compose exec app ping -c 2 db` resolves and reaches the database container.
- **Persistence:** a row inserted into PostgreSQL is still there after `docker compose down` followed by `docker compose up -d`, because data lives in the named volume.
- **Registry round trip:** with the local image removed, `docker compose up -d` pulls `nikhilj1402/javapp:v1` from Docker Hub.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `port is already allocated` | Something else uses 8082 | Run `docker ps -a` to find it, or change the left side of `ports` |
| `curl` resets or returns empty reply | Container port in `ports` does not match the port the app listens on (8080) | Make the right side of `ports` `8080` |
| `pull access denied` | Image name typo, or the image is private | Check the name (`javapp`, one "a") and the Docker Hub repo visibility |
| `bad address 'db'` inside the app container | Services are not on the same Compose network, or the name is misspelled | Check service names in `docker-compose.yml` |
| Container exits straight away | App crashed on startup | `docker compose logs app`, then check `CMD` vs `ENTRYPOINT` |

## Troubleshooting drills

### Drill 1: Port mapping mismatch

**Break it.** In `docker-compose.yml`, change the app's port mapping from `"8082:8080"` to `"8082:9999"`, then apply it:

```bash
docker compose up -d
curl -i http://localhost:8082
```

**Symptom.** `curl` fails with `Connection reset by peer` or `Empty reply from server`. Docker still forwards host port 8082, but nothing inside the container listens on 9999.

**Diagnose.**

```bash
docker compose ps
docker compose logs app | grep "started on port"
```

`ps` shows `8082->9999`, while the Tomcat log says the app is on `8080`. That mismatch is the root cause.

**Fix.** Set the mapping back to `"8082:8080"` and run `docker compose up -d`. `curl` returns the Spring Boot `404` JSON again.

**Lesson.** The right side of `ports` must match the port the app actually listens on. `EXPOSE` in the Dockerfile is documentation only and does not publish or fix anything. The app must also bind to `0.0.0.0`, not `localhost`, or it is unreachable from outside the container.

### Drill 2: Large image

Check the image size:

```bash
docker images nikhilj1402/javapp
```

The multi-stage build keeps the Maven and JDK build tools in the first stage. Only the jar and a slim Java runtime go into the final image, which keeps it small.

### Drill 3: Service name does not resolve

If `docker compose exec app ping -c 2 db` returns `bad address 'db'`, the services are not on the same Compose network or the service name is misspelled. Check the service names in `docker-compose.yml`.

## Notes

- This is a learning setup. The database credentials in `docker-compose.yml` are demo values. For anything real, keep secrets out of the file (use an `.env` file that is git-ignored, or Docker secrets).
- The app does not use the database yet. The `db` service demonstrates Compose networking and volume persistence.

## Tech stack

Java 21 · Spring Boot · Maven · Docker (multi-stage builds) · Docker Compose · PostgreSQL 16
