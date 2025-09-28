# Deployment and Monitoring Guide with Docker Compose

## 1. Architecture Overview

This `docker-compose.yml` file describes and orchestrates a complete environment for the development and monitoring of "MagicProxy". It automatically configures and launches not only the application itself but also the entire necessary stack for asynchronous work and centralized log collection.

The system consists of the following services:
*   **app-runner**: The main application service. It runs the `run.py` script, which in turn starts `magic-proxy` and `mcp-server` in a single process.
*   **kafka & redis**: Infrastructure services for asynchronous tasks, caching, and real-time messaging.
*   **Observability Stack**:
    *   **loki**: The "database" for your logs.
    *   **promtail**: A log collector that automatically discovers all running containers and sends their logs to Loki.
    *   **grafana**: A visualization platform where you will view and analyze logs from Loki.

## 2. Prerequisites (Checklist)

Before the first launch, ensure that the following files and folders exist in the root directory of your project. `docker-compose` expects to find them to correctly build and run the system.

*   [ ✅ ] `Dockerfile`: The "recipe" for building the Docker image of your Python application.
*   [ ✅ ] `requirements.txt`: A list of Python dependencies that will be installed in the image.
*   [ ✅ ] `keys_pool/`: Directory with API key files for `key_manager`.
*   [ ✅ ] `env/`: Directory with key files for tools (`mcp_server`).
*   [ ✅ ] `proxy_config.yaml`: The main configuration file for your gateway.
*   [ ✅ ] `config/`: Directory with configuration files for the monitoring stack. The structure should be as follows:
    ```
    config/
    ├── loki/
    │   └── local-config.yaml
    ├── promtail/
    │   └── promtail-config.yml
    └── grafana/
        └── provisioning/
            └── datasources/
                └── loki.yml
    ```

## 3. Key Architectural Decisions in docker-compose.yml

Understanding these nuances is critical for successful operation.

### 3.1. app-runner and Centralized Logging

Instead of running `uvicorn` directly, the `app-runner` service executes `python run.py`. This is intentional: `run.py` aggregates logs from `magic-proxy` and `mcp-server` into a single output stream. This allows Promtail to collect all application logs from one source, which significantly simplifies their subsequent analysis in Grafana.

### 3.2. Reliable Startup via healthcheck and depends_on

A simple `depends_on` directive only guarantees the startup order of containers, not their readiness. Our file uses a more reliable mechanism:

*   **healthcheck**: In the `kafka` and `redis` services, commands are defined that Docker periodically executes inside the container to check that the service is not just running, but fully operational.
*   **condition: service_healthy**: In the `app-runner` service, this instruction says: "Do not start until the healthchecks for `kafka` and `redis` have passed successfully."

This solves the critical "race condition" problem, preventing `Connection refused` errors that occur when the application tries to connect to a database or message broker that is not yet ready.

### 3.3. Kafka Network Configuration

The Kafka service is configured with two "listeners":

*   `EXTERNAL://localhost:9092`: For connecting to Kafka from your host machine (e.g., for debugging).
*   `INTERNAL://kafka:9093`: For connecting to Kafka from other containers within the Docker network.

Therefore, in `app-runner`, we use `KAFKA_BROKER=kafka:9093`. This ensures correct and efficient network communication within Docker.

## 4. Starting and Managing the System

All commands should be executed in the directory where the `docker-compose.yml` file is located.

### 4.1. Initial Startup

```bash
docker-compose up -d
```

This command will build the image, download dependencies, and start all containers. Startup may take up to a minute, as `app-runner` will wait for Kafka to be fully ready.

### 4.2. Viewing Logs in Grafana

1.  Open `http://localhost:3000` in your browser.
2.  Log in (default login/password: `admin/admin`).
3.  Go to the "Explore" section (compass icon).
4.  The `Loki` data source should already be selected.
5.  Use the "Log browser" field to query logs.

**Example Queries:**
*   All logs from your application: `{container_name="app-runner"}`
*   Only errors: `{container_name="app-runner"} |= "ERROR"`
*   Logs from Kafka: `{container_name="kafka"}`

### 4.3. Stopping the System

```bash
docker-compose down
```

### 4.4. Rebuilding the Application Image

If you have changed the `Dockerfile` or `requirements.txt`, use this command:

```bash
docker-compose up -d --build
```