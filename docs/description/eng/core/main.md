# Technical Documentation: `main.py` — The Central Gateway Orchestrator

## 1. Overview

`main.py` is the **central entry point** and **orchestrator** for the entire "Magic Proxy" application. It initializes, configures, and launches the web application based on FastAPI. The module acts as a "conductor," connecting all independent components into a single, fault-tolerant system.

## 2. Architectural Role and Problems Solved

*   **Orderly Initialization**: Ensures that by the time the first HTTP request is received, all dependencies (configuration, Redis, Kafka, key manager) and systems (logging, tracing) are ready for operation.
*   **Centralized Routing**: Provides a single, OpenAI-compatible API (`/v1/...`) that "transparently" redirects requests to the best available AI provider based on the configuration.
*   **Ensuring High Availability**: Abstracts away failures of individual API keys or entire providers by automatically switching to backup options.
*   **Manageability**: Provides an administrative UI and API (`/admin/...`) for viewing and modifying the configuration and prompts on the fly without restarting the application.

## 3. Key Mechanisms and Logic

### 3.1. Lifespan Management

The FastAPI `lifespan` context manager is used to manage resources. This ensures a correct and orderly sequence of actions during application startup and shutdown.

**On server startup:**
1.  The configuration is loaded from `proxy_config.yaml`.
2.  The OpenTelemetry tracing system is initialized.
3.  The `ApiKeyManager` is started, which loads all keys from `keys_pool/` and starts background tasks (e.g., returning keys from "quarantine").
4.  All available providers (adapters from `providers/`) are registered.
5.  Connections to Redis and Kafka are established.
6.  A background worker is started to process tasks from Kafka (if `react_driver` is enabled).

**On server shutdown:**
All resources (connections to Kafka, Redis, background tasks) are correctly released.

### 3.2. Centralized Routing (`route_request`)

This is the core of the "transparent" proxy. All requests to LLMs (`/v1/chat/completions`, `/v1/embeddings`, etc.) are ultimately handled by this function.

**Algorithm:**
1.  The model alias (e.g., `gpt-4-turbo`) is extracted from the request body.
2.  The corresponding **priority_chain** for this alias is found in `proxy_config.yaml`. Example: `['openai_main_gpt4', 'azure_backup_gpt4']`.
3.  If the chain is not found, a 404 error is returned.
4.  The system iterates through each profile name (`internal_model_name`) in the chain.
5.  For each profile, its provider (e.g., "openai") is determined, and the corresponding adapter function (e.g., `proxy_openai_compat_chat`) is found.
6.  Control is passed to the fault tolerance mechanism (`execute_request_with_key_rotation`), which attempts to execute the request with the current provider.
7.  **On success**: The result is immediately returned to the client.
8.  **On failure**: A warning is logged, and the loop proceeds to the next provider in the chain.
9.  If all providers in the chain are unavailable, a 503 Service Unavailable error is returned.

### 3.3. Fault Tolerance Orchestration (`execute_request_with_key_rotation`)

This function encapsulates the logic for executing a request to a **single** provider, ensuring the iteration through all its available API keys.

**Algorithm:**
1.  The provider (e.g., "google") is determined, and the `ApiKeyManager` is queried for the number of available keys to know the maximum number of attempts.
2.  A loop of attempts is started:
    a. A working key is requested from the `ApiKeyManager`. If no free keys are available, the loop is broken.
    b. The provider's adapter function is called, passing it the request, key, and configuration.
    c. **On success**: The key is returned to the pool as working, and the result is passed to `route_request`.
    d. **On `httpx.HTTPStatusError`**:
        *   **401 Unauthorized / "invalid api key"**: The key is considered invalid and is **"retired"** (`retire_key`).
        *   **429 Too Many Requests / 5xx Server Error**: The key is considered temporarily unavailable and is sent to **"quarantine"** for a specified time (`quarantine_key`).
        *   **400 Bad Request**: An error in the user's request; the key is not at fault and is returned to the pool, while the error is propagated to the client.
    e. Proceed to the next attempt with a new key.
3.  If all attempts are exhausted, a `ProviderUnavailableError` exception is raised, which `route_request` uses as a signal to switch to the next provider in the chain.

## 4. Administration API and UI

The module provides a set of endpoints under the `/admin/` prefix for managing the application on the fly:

*   `GET, POST /admin/config`: Read and write the `proxy_config.yaml` file with immediate configuration reload.
*   `GET, POST /admin/prompt_content`: Read and write system prompt files.
*   `GET /admin/react_patterns`: Get a list of available ReAct patterns.
*   `GET /admin/provider_models`: Get a list of models specified in the configuration.
*   `POST /admin/restart`: Programmatically restart the server.

Additionally, a static web interface (`frontend/`) is served at the root URL `/`, providing a convenient UI for interacting with this API.

## 5. Dependencies and Configuration

### 5.1. Key Dependencies

*   **Web Server**: `fastapi`, `uvicorn`
*   **HTTP Client**: `httpx`
*   **Infrastructure**: `redis` (for cache and SSE), `aiokafka` (for ReAct)
*   **Configuration**: `pyyaml`
*   **Monitoring**: `opentelemetry-*`

### 5.2. Internal Modules

*   `key_manager`: Manages the lifecycle of API keys.
*   `providers`: Adapters for each AI provider.
*   `react_driver`, `sse_driver`, `cache_manager`: Components for advanced features.
*   `utils`, `logging_config`, `tracing`: Helper utilities.

### 5.3. Launching

To run the server in development mode, use `uvicorn`:
```bash
uvicorn main:app --host 0.0.0.0 --port 8001 --reload
```
(Note: For convenient startup of all system components, it is recommended to use `run.py`.)