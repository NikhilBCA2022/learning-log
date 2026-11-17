# Day 10 – Docker Fundamentals and Hands-on Practice

## Overview

Day 10 focused on learning Docker fundamentals and understanding how containerization works in real-world DevOps environments.

I practised containerizing an application, working with Docker volumes, understanding container networking, troubleshooting common Docker errors, and cleaning up resources.

## 1. What I Learned

- Docker and containerization fundamentals.
- Difference between Docker images and containers.
- How to build Docker images using a Dockerfile.
- How to run applications inside containers.
- Docker port mapping.
- Docker volumes and persistent storage.
- Docker container networking.
- Container-to-container communication using container names.
- Docker container lifecycle and cleanup.

## 2. Practical Implementation

### A. Application Containerization

Created a Dockerfile to define how the application should be packaged into a Docker image.

Learned:
- Base images.
- Working directories.
- Copying application files.
- Installing dependencies.
- Exposing application ports.
- Defining startup commands.

Built and ran the application inside a Docker container.

### B. Docker Volumes

Practised Docker volume mounting to understand persistent storage.

**Key learning:**

Container data can disappear when a container is removed. Docker volumes allow important data to persist independently of the container lifecycle.

Verified that data remained available after removing and recreating the container.

### C. Docker Networking

Created and tested Docker container networking.

Verified that containers connected to the same Docker network can communicate using their container names instead of hardcoded IP addresses.

**Key learning:**
- Containers can communicate over a shared Docker network.
- Docker provides internal DNS-based name resolution.
- Container IP addresses do not need to be hardcoded.

### D. Port Mapping

Practised exposing container applications through host ports.

Example:

`8081:80`

- `8081` – Host machine port.
- `80` – Container port.

This allows users to access an application running inside a container through the host machine.

## 3. Troubleshooting and Runbook Log

Encountered and resolved two real-world Docker issues.

### Issue 1: Docker Socket Permission Denied

**Error:**

`permission denied while trying to connect to the Docker daemon socket`

**Root Cause:**

The current Linux user did not belong to the Docker group.

**Diagnosis:**

```bash
docker ps
groups
```

- `docker ps` – Checked whether Docker commands could communicate with the daemon.
- `groups` – Checked the groups assigned to the current user.

**Fix:**

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

**Explanation:**

- `usermod` – Modifies an existing Linux user.
- `-aG docker` – Adds the user to the Docker group without removing existing supplementary groups.
- `newgrp docker` – Starts a shell with the updated group membership.

**Learning:**

Linux permissions control who can communicate with the Docker daemon.

Note: Membership in the Docker group effectively grants root-level privileges, so it should be restricted to trusted users.

### Issue 2: Port Already Allocated

**Error:**

`Bind for 0.0.0.0:8081 failed: port is already allocated`

**Root Cause:**

An older Docker container was already using host port 8081.

**Diagnosis:**

```bash
docker ps -a
ss -tulpn
```

- `docker ps -a` – Lists all containers, including stopped containers.
- `ss -tulpn` – Displays listening network sockets and associated processes.

**Fix:**

```bash
docker rm -f <container_name>
```

**Explanation:**

- `docker rm` – Removes a container.
- `-f` – Forcefully stops and removes a running container if necessary.

**Learning:**

A host port cannot be allocated to multiple conflicting container bindings simultaneously.

Always inspect existing containers before assigning a port.

## 4. Important Docker Commands

| Command | Purpose |
|---|---|
| `docker --version` | Check Docker version |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker images` | List downloaded and built images |
| `docker build -t app .` | Build an image |
| `docker run -d app` | Run a container in detached mode |
| `docker logs <container>` | View container logs |
| `docker exec -it <container> sh` | Access a container shell |
| `docker volume ls` | List Docker volumes |
| `docker network ls` | List Docker networks |
| `docker rm -f <container>` | Forcefully remove a container |
| `docker rmi <image>` | Remove an image |
| `docker system df` | Check Docker disk usage |

## 5. Key Concepts for Interviews

| Concept | Explanation |
|---|---|
| Docker | Platform for building and running containers |
| Dockerfile | Instructions used to build an image |
| Docker Image | Read-only template used to create containers |
| Docker Container | Running instance of an image |
| Docker Volume | Persistent storage managed by Docker |
| Docker Network | Enables communication between containers |
| Port Mapping | Connects a host port to a container port |
| Docker Daemon | Background service responsible for managing Docker objects |
| Docker Group | Linux group that allows users to access the Docker socket |

## 6. Interview Questions

**Q1. What is Docker?**

Docker is a containerization platform that packages applications and their dependencies into containers.

**Q2. What is the difference between an image and a container?**

An image is a template, whereas a container is an instance created from that image.

**Q3. Why do we use Docker volumes?**

To persist important data independently of the container lifecycle.

**Q4. How do containers communicate with each other?**

Containers connected to the same Docker network can communicate using container names through Docker's internal DNS.

**Q5. What is port mapping?**

It connects a port on the host machine to a port inside a container.

**Q6. Why did Docker give a permission denied error?**

Because the current user did not have permission to access the Docker daemon socket.

**Q7. Why does Docker show port already allocated?**

Because another container or process is already using the requested host port.

## 7. Security Best Practices

- Avoid running containers as root unnecessarily.
- Use trusted base images.
- Avoid storing secrets inside Dockerfiles.
- Restrict access to the Docker socket.
- Remove unused containers and images.
- Use Docker networks to control container communication.
- Use volumes for persistent application data.

## 8. Cleanup

After completing the practical exercises:

- Verified the application functionality.
- Verified volume persistence.
- Verified container networking.
- Removed unnecessary containers.
- Cleaned up resources.
- Confirmed the environment was no longer running unnecessary containers.

## 9. What I Achieved

By completing Day 10, I gained practical experience with:

1. Building Docker images.
2. Running applications inside containers.
3. Understanding Docker port mapping.
4. Persisting data using Docker volumes.
5. Connecting containers through Docker networks.
6. Troubleshooting Linux permission issues.
7. Diagnosing port conflicts.
8. Managing and cleaning up Docker resources.

**Revision Performance:** 4 out of 7 answers fully correct, with the remaining answers close.

## 10. Next Steps

- Push the Dockerfile to the GitHub repository.
- Practise Docker Compose.
- Understand multi-stage Docker builds.
- Learn Docker image optimization.
- Explore container security scanning.

---

**Day 10 Status: COMPLETED**

**Main Takeaway:**

Docker allows applications to run consistently in isolated environments. Images package applications, containers execute them, volumes preserve their data, and networks allow containers to communicate.
