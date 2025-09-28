# Technical Documentation: key_manager.py Module

## 1. Executive Summary

`key_manager.py` implements the `ApiKeyManager` class—a high-level, thread-safe, and self-healing manager for API key pools. Its purpose is to ensure maximum reliability when interacting with external API providers. The module encapsulates complex logic for managing the key lifecycle: it automatically classifies errors, isolates temporarily failing keys in "quarantine," and permanently retires compromised ones. This provides graceful degradation and autonomous system recovery.

## 2. Architectural Role and Problem Solved

When working with paid external APIs at an industrial scale, several critical problems arise:
*   **Temporary Failures:** API keys often have rate limits. Exceeding the limit on one key should not paralyze the entire system.
*   **Permanent Failures:** Keys can be revoked, expire, or be invalid. The system must identify such keys and permanently remove them from rotation.
*   **High Concurrency:** In an asynchronous environment, multiple requests may try to use the same key pool, leading to race conditions without proper synchronization.
*   **Operational Complexity:** Manually managing dozens of keys, tracking their status, and returning them to service in a timely manner requires significant effort.

`ApiKeyManager` solves these problems by creating an abstraction of an intelligent, fault-tolerant key "provider."

## 3. Key Components and Architectural Patterns

### 3.1. Data Structure and State Machine

At the core of the manager is the `_pools` dictionary, which stores the state of keys for each provider. Each key can be in one of three states (pools):
*   **available:** A list of keys ready for use. It operates on a FIFO (First-In, First-Out) basis.
*   **quarantined:** A dictionary of keys temporarily isolated due to non-critical errors (e.g., rate limit). It stores the key, the reason, and the end time of the quarantine.
*   **retired:** A dictionary of keys permanently taken out of service due to a fatal error (e.g., invalid token).

The transition between states is managed by the `quarantine_key` and `retire_key` methods.

### 3.2. Key Lifecycle Management

*   `get_key(provider)`: Atomically retrieves the first available key from the `available` pool.
*   `release_key(provider, key)`: Returns a used key to the end of the `available` pool.
*   `quarantine_key(provider, key, reason)`: Moves a key from `available` to `quarantined` for a specified duration (`QUARANTINE_DURATION_SECONDS`). **Important:** This logic is only activated if the global flag `ENABLE_QUARANTINE` is set to `True`. Otherwise, the key is immediately returned to `available` via `release_key`.
*   `retire_key(provider, key, reason)`: Permanently moves a key from any state to `retired`, excluding it from further use.

### 3.3. Background Worker Pattern (_check_quarantine)

The module runs an asynchronous background task that periodically (every 10 seconds) checks the `quarantined` pool. If a key's isolation period has expired, it is automatically returned to the `available` pool. This implements a self-healing pattern, reducing the need for manual intervention. The worker only runs if `ENABLE_QUARANTINE = True`.

### 3.4. Thread Safety (asyncio.Lock)

All operations that modify the state of the pools (`_pools`) are protected by an `asyncio.Lock`. This ensures atomicity and prevents race conditions in a high-load asynchronous FastAPI environment.

### 3.5. Loading and Configuration

*   **`load_all_keys()`**: On application startup, this method scans the `keys_pool/` directory for files named `keys_pool_{provider}.env`. Each file contains a list of API keys (one per line).
*   **Centralized Configuration**: Key parameters are defined as constants at the beginning of the file:
    *   `ENABLE_QUARANTINE`: A global switch that completely enables or disables the quarantine mechanism. It allows for quick changes to the error handling strategy without a restart.
    *   `QUARANTINE_DURATION_SECONDS`: The duration of key isolation.
    *   `PERMANENT_KEY_ERROR_MESSAGES`, `REQUEST_CONTENT_ERROR_MESSAGES`: Sets of string markers used in `main.py` to decide which manager method to call (`quarantine_key`, `retire_key`, or `release_key`).

## 4. Setup and Configuration

Before using `ApiKeyManager`, you need to structure the API key files correctly.

### 4.1. File Structure

1.  In the root directory of your project, create a folder named `keys_pool/`.
2.  Inside `keys_pool/`, create separate `.env` files for each provider. The filename must follow the format `keys_pool_{provider_name}.env`.

**Important:** For all Google services (`google-embedding`, `google-stt`, `google-tts`), the same file corresponding to the main `google` provider is used.

**Example Structure:**
```
.
├── keys_pool/
│   ├── keys_pool_google.env
│   └── keys_pool_openai.env
├── main.py
└── key_manager.py
```

### 4.2. File Contents

Each file should contain a list of API keys, one key per line. Empty lines and surrounding whitespace are ignored.

**Example `keys_pool_openai.env`:**
```
sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
sk-proj-yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
sk-proj-zzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzzz
```

## 5. Integration and Usage Example (Workflow)

`ApiKeyManager` reveals its full power when integrated into request handling logic, for example, in the `main.py` module. Below is a simplified but representative example of a function that reliably executes a call to an external API using the manager.

### 5.1. Code Example
```python
# This code is a simplified example of what an orchestrator function
# using ApiKeyManager might look like.
import httpx
from fastapi import Request

async def safely_call_provider(request: Request, provider: str, payload: dict):
    """
    Reliably executes an API request using ApiKeyManager for key rotation,
    quarantine, and retirement.
    """
    key_manager = request.app.state.key_manager
    # Determine the maximum number of attempts based on available keys
    status = await key_manager.get_full_status()
    max_attempts = status.get(provider, {}).get('available', 0) + 1

    last_exception = None

    for attempt in range(max_attempts):
        # 1. Request a key from the pool
        api_key = await key_manager.get_key(provider)
        if not api_key:
            last_exception = Exception(f"All keys for '{provider}' are exhausted.")
            break # Exit the loop if no keys are left

        try:
            # 2. Execute the external API call
            async with httpx.AsyncClient() as client:
                response = await client.post("https://api.provider.com/v1/...", json=payload, headers={"Authorization": f"Bearer {api_key}"})
                response.raise_for_status() # Will raise an exception for 4xx/5xx codes

            # 3. On SUCCESS - release the key and return the result
            await key_manager.release_key(provider, api_key)
            return response.json()

        except httpx.HTTPStatusError as e:
            last_exception = e
            error_message = str(e).lower() # Analyze the error text

            # 4. On ERROR - classify it and react
            if any(msg in error_message for msg in PERMANENT_KEY_ERROR_MESSAGES):
                # Fatal key error -> "retire" it
                await key_manager.retire_key(provider, api_key, f"HTTP {e.response.status_code}")
            elif e.response.status_code in [429, 500, 502, 503]:
                # Temporary server/limit error -> quarantine it
                await key_manager.quarantine_key(provider, api_key, f"HTTP {e.response.status_code}")
            else:
                # Other error (e.g., 400 Bad Request due to a bad prompt)
                # -> just release the key, as it's not at fault
                await key_manager.release_key(provider, api_key)
                raise e # Re-raise the exception, as it's related to the request

            # ... and proceed to the next attempt with a new key

        except Exception as e:
            # Unexpected error (e.g., connection drop)
            last_exception = e
            await key_manager.release_key(provider, api_key) # Release the key just in case
            continue

    # If all attempts fail
    raise Exception(f"Failed to execute request to '{provider}' after {max_attempts} attempts. Last error: {last_exception}")
```

### 5.2. Example Breakdown

*   **Requesting a Key:** The loop starts by requesting a key via `await key_manager.get_key(provider)`. This is an atomic, thread-safe operation.
*   **API Call:** The key is used within a `try...except` block. This is critically important as it allows for intercepting and analyzing errors.
*   **Handling Success:** On a successful request (2xx code), the key is always returned to the pool via `await key_manager.release_key(...)` so other requests can use it.
*   **Handling Errors (Key Logic):**
    *   If the API returns an error that clearly indicates a problem with the key (invalid, expired), `retire_key` is called. The key is permanently removed from rotation.
    *   If the error is temporary (rate limit, server unavailable), `quarantine_key` is called. The key is temporarily isolated and will be automatically returned to the pool by the background worker.
    *   If the error is related to the request content (400 Bad Request), the key is not at fault. It is simply returned to the pool via `release_key`, and the exception is re-raised so the client receives an error message.
*   **Retry Loop:** All this logic is wrapped in a loop. If one key fails (is quarantined or retired), the loop simply moves to the next iteration and requests the next available key from the manager, ensuring seamless fallback.

## 6. Architectural Value and Business Impact

`ApiKeyManager` is a critically important component of the reliability infrastructure.
*   **Increased Availability (High Availability):** By automatically rotating and isolating failing keys, the system continues to serve requests even if some of its resources are unavailable.
*   **Reduced Operational Costs:** Automating the quarantine and retirement processes minimizes the time engineers spend on manual diagnostics.
*   **Cost Optimization:** Preventing repeated requests with known non-functional keys saves computational resources and money.
*   **Observability:** The `get_full_status` method provides the necessary telemetry for monitoring systems, allowing for real-time tracking of the key pool's "health."

Ultimately, `ApiKeyManager` transforms a chaotic set of static API keys into a dynamic, adaptive, and fault-tolerant resource, which is a mandatory requirement for building scalable cloud applications.