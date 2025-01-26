### Overview

A load balancer written in Go that distributes HTTP traffic across multiple backend servers. It includes health checks,
retries, and round-robin scheduling.

#### Features

* **Round-Robin Scheduling**: Distributes requests evenly.
* **Active Health Checks**: Periodic TCP checks (every 2 minutes).
* **Retry Mechanism**: 3 retries for failed requests.
* **Dynamic Backend Management**: Auto-marks unreachable backends as down.

**Requirements**

* Docker installed
* Backend servers running, see [testing](#testing)

**Quick Start**

1. Build the docker file:

```bash
docker build -t load-balancer .
```

2. Run the load balancer:

```bash
docker run -p 3030:3030 load-balancer \
  -backends "http://host.docker.internal:8080,http://host.docker.internal:8081"
```

* `-backends`: Comma-separated URLs of backend servers.
* `-port`: Port for the load balancer (default: 3030).

**Network Considerations**

* Host Backends: Use `host.docker.internal` (macOS/Windows) or your machine’s IP (Linux) to reference host-based
  backends.
* Docker Network: For containerized backends, create a shared network (e.g., `docker network create lb-network`) and use
  service names as backend URLs.

**Testing**

1. **Start Backend Servers:**

```bash
python3 -m http.server 8080 &
python3 -m http.server 8081 &
```

Use simple HTTP servers on different ports:

2. **Send Requests:**

```bash
curl http://localhost:3030
```

Requests will alternate between 8080 and 8081.

3. **Simulate Failures:**

Kill one backend.

```bash
kill <pid_of_8080>
```

The load balancer will detect it and route traffic to the remaining server.

4. **Load Testing:**
   Use Apache Bench to simulate traffic:

```bash
ab -n 1000 -c 10 http://localhost:3030/
```

**Health Checks**

* Active: Every 2 minutes, the load balancer checks backend reachability.
* Passive: Failed requests trigger immediate retries and status updates.

**Advanced Deployment**

Docker compose example

```yaml
version: '3'
services:
  load_balancer:
    image: load-balancer
    ports:
      - "3030:3030"
    command: -backends "http://backend1:8080,http://backend2:8080"

  backend1:
    image: my-backend-app
    ports:
      - "8080:8080"

  backend2:
    image: my-backend-app
    ports:
      - "8081:8080"
```

**Logs**
The load balancer logs:

* View container logs

```bash
docker logs <container-id>
```

* Sample logs

```
http://backend1:8080 [up]
http://backend2:8080 [down] (3 retries failed) 
```

**Example Workflow**

* Start two backends on ports 8080 and 8081.
* Send requests to http://localhost:3030; observe traffic distribution.
* Stop a backend; the load balancer skips it after 3 retries.
* Restart the backend; health checks will re-enable it.

**Limitations**

* Health checks are TCP-based (not HTTP). A backend may be marked "up" if the port is open, even if the app is failing.
* No weighted round-robin or advanced algorithms.

**Changelog**

* v1.1: Added Docker support.
* v1.0: Initial release with round-robin and health checks.