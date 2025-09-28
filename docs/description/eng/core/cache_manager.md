# Technical Documentation: cache_manager.py Module

## 1. Executive Summary

`cache_manager.py` is a utility module that implements a configurable, deterministic cache based on Redis. Its primary goal is to reduce costs and improve performance by saving and reusing the results of expensive API calls. The module provides functions for generating a cache key based on the request content, as well as for writing and reading data from the cache.

## 2. Architectural Role and Problem Solved

In systems that heavily use external AI APIs, many requests can be repetitive (e.g., requests to translate standard phrases, identical queries to a RAG system, etc.). Each such call costs money and time, and also consumes API limits (rate limits).

`cache_manager.py` solves this problem by introducing a caching layer that intercepts requests. If an identical request has been made recently, its result is instantly returned from Redis, bypassing the costly and slow call to the external provider.

Key problems solved:
* **Excessive costs:** Prevents repeated payments for the same computations.
* **Low performance:** Instantly returns cached responses instead of waiting for a response from an external API.
* **Limit exhaustion:** Reduces the total number of requests to providers, helping to stay within established rate limits.

## 3. Key Functions and Their Roles

### 3.1. create_cache_key(...)

This is the central function of the module, responsible for creating a unique but deterministic identifier for caching.

```python
def create_cache_key(
    request_data: BaseModel,
    model_config: Dict[str, Any],
    cache_config: Dict[str, Any]
) -> Optional[str]:
    ...
```

**Algorithm:**
1. Checks if caching is enabled in `cache_config` in the `proxy_config.yaml` file.
2. Finds a rule in `cache_config.rules` that matches the current model (`internal_model_name`).
3. If a suitable rule is found, the function creates a dictionary, including only the fields from the request body (`request_data`) that are listed in `rule.include_in_key`.
4. The `internal_model_name` is always added to this dictionary to avoid collisions between different models with identical requests.
5. The resulting dictionary is serialized into a stable JSON string (with sorted keys).
6. A SHA-256 hash is calculated from this string, which becomes the unique identifier for the request.
7. If no rule is found or if no fields from the request are included in the hash, the function returns `None`, and caching is not performed for this request.

**Key Insight:** The key is deterministic. Two requests with the exact same content in the fields specified in `include_in_key` will always generate the same key. The function does not use timestamps or random data.

### 3.2. set_to_cache(...) and get_from_cache(...)

These are simple asynchronous wrappers for interacting with Redis.

```python
async def set_to_cache(key: str, value: str, redis_client: redis.Redis, ttl_seconds: int):
    # Sets a value by key with a specified time-to-live (TTL).
    ...

async def get_from_cache(key: str, redis_client: redis.Redis) -> Optional[str]:
    # Gets a value by key. Returns None if the key is not found.
    ...
```

Both functions check for an active `redis_client` and do nothing if it is absent, making them safe to use even when Redis is unavailable.

## 4. Configuration

The cache's behavior is fully controlled through the `cache_settings` section in the `proxy_config.yaml` file.

**Configuration Example:**

```yaml
cache_settings:
  # Global switch for the entire caching system
  enabled: true

  # Prefix added to all keys in Redis
  key_prefix: "magic_proxy:cache:"

  # List of caching rules
  rules:
    # Rule #1: cache requests to embedding models
    - model_names:
        - "openai_embedding_3_large"
        - "google_embedding_004"
      # Include only the 'input' field in the key
      include_in_key:
        - "input"
      # Store the result in the cache for 1 hour (3600 seconds)
      ttl_seconds: 3600

    # Rule #2: cache requests to a specific chat model
    - model_names:
        - "openai_gpt4_main"
      # Include messages and generation parameters in the key
      include_in_key:
        - "messages"
        - "temperature"
        - "max_tokens"
      # Store the result for 10 minutes
      ttl_seconds: 600
```

## 5. Typical Workflow

The `cache_manager` module is integrated into the main request processing flow (e.g., in `main.py`) as follows:

**Before executing a request:**
1. `create_cache_key()` is called with the request data and configuration.
2. If the function returns a `key`, then...
3. `get_from_cache(key)` is called.
4. If a result (`cached_value`) is found, it is immediately returned to the client, and all further processing stops (Cache Hit).

**If the cache is empty (Cache Miss):**
1. The request is executed normally through `route_request` and `execute_request_with_key_rotation`.
2. After receiving a successful response from the AI provider...
3. `set_to_cache(key, result, ttl_seconds)` is called to save the result for future requests.
4. The result is returned to the client.

## 6. Architectural Value and Business Impact

* **Reduced operational costs:** Direct cost savings by reducing the number of paid API calls. The greatest effect is achieved on frequently repeated requests (embeddings, standard chat queries).
* **Improved user experience:** Significant response acceleration for cached requests (milliseconds instead of seconds).
* **Increased reliability:** Reducing the load on external APIs decreases the likelihood of encountering `RateLimitError` and temporary provider-side failures.
* **Flexibility:** Configuration via YAML allows for fine-tuning which models and which request parameters to cache without changing the code.