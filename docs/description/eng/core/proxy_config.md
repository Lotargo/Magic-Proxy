# Configuration Guide: `proxy_config.yaml`

## 1. Introduction and Philosophy

The `proxy_config.yaml` file is the **single control panel** for all behavior of the "Magic Proxy". It allows you, as the system administrator, to design complex AI architectures, manage fault tolerance, and activate advanced cognitive functions without changing a single line of source code.

**Key Principles:**
*   **Declarative**: You describe the desired state of the system, you don't program it.
*   **Centralized**: All routing logic and model definitions are in this single file, which simplifies management and auditing.
*   **Explicit is better than implicit**: All behavior settings are defined explicitly.

## 2. Core Concepts

The configuration is built on two interconnected blocks: `model_list` and `router_settings`.

*   **`model_list` (Model Profile Catalog)**: This is your complete and exhaustive catalog of all "building blocks"—model profiles. Each profile describes **one specific configuration** for calling an AI model.
*   **`router_settings` (Request Routing)**: This block defines which models are available to external clients and how the system should react to failures. It uses profiles from `model_list` to build **fallback chains**.

### 2.1. Step 1: Defining a Model Profile in `model_list`

Each element in `model_list` is a **profile** describing how to call one specific model.

```yaml
model_list:
  - model_name: "openai_main_gpt4"  # 1. Unique internal profile name
    provider: "openai"              # 2. Provider (adapter) identifier
    model_params:                   # 3. Parameters for the API call
      model: "gpt-4-turbo"          #    - Model name on the provider's side
      temperature: 0.7
      # ... other parameters ...
```

**Profile Structure:**

*   `model_name` (**Required**): A unique internal name for this profile. You will reference it in `router_settings`.
*   `provider` (**Required**): A string identifier that tells the gateway which internal "protocol adapter" from the `providers/` folder to use. Available values: `google`, `openai`, `mistral`, `anthropic`, `deepseek`, `local`.
*   `model_params` (**Required**): A dictionary of parameters for calling the model.
    *   `model` (**Required**): The actual name of the model as specified in the provider's documentation (e.g., "gemini-1.5-flash-latest").
    *   `api_base` (Optional): Allows overriding the standard OpenAI URL to use OpenAI-compatible APIs from other providers (Groq, Together AI, etc.).
    *   `temperature`, `top_p`, `max_tokens`: Standard generation parameters. `null` means "use the provider's default value."

### 2.2. Step 2: Creating a Client Route in `router_settings`

Once you have defined one or more profiles, you can "publish" them for clients via `router_settings`.

```yaml
router_settings:
  model_group_alias:
    "production-gpt4": # <-- 1. The alias the client uses
      # 2. The fallback chain
      - "openai_main_gpt4"
      - "azure_backup_gpt4"
```

**How it works:**
1.  **Alias (`model_group_alias`)**: The key (`production-gpt4`) is the name that your client application should specify in the `model` field of its API request.
2.  **Priority Chain**: The value is a list of profile names (`model_name` from `model_list`) that the gateway will try to use in order.

When a request with `model: "production-gpt4"` arrives, the gateway:
1.  Takes the first profile from the list (`openai_main_gpt4`) and tries to execute the request.
2.  If the attempt fails (the API provider is unavailable or all its keys have failed), the gateway automatically takes the second profile (`azure_backup_gpt4`) and retries.
3.  This process continues until the first successful execution or the end of the list.

## 3. Advanced Configuration

### 3.1. Activating the Reasoning Engine (ReAct)

You can "upgrade" any model profile by activating the `react_driver` engine for it.

To do this, add a nested `agent_settings` block to the model profile:
```yaml
model_list:
  - model_name: "gemini-agent-profile"
    provider: "google"
    model_params:
      model: "gemini-1.5-flash-latest"
      agent_settings: # <-- Activate ReAct
        reasoning_mode: "basic_react" # <-- Specify the cognitive pattern
```
*   `reasoning_mode`: A key parameter. It specifies the name of the cognitive pattern that will be used to "enrich" the request. The name must correspond to the name of a `*_react.py` file in the `react_patterns/` system directory. If the `agent_settings` block is absent, the request is handled as a direct proxy.

### 3.2. Using Templates (YAML Anchors)

To avoid duplication, you can use YAML anchors. This is especially useful for common parameters like `api_base`.

```yaml
definitions:
  # Template with the base URL for all Mistral models
  mistral_common_params: &mistral_common_params
    api_base: https://api.mistral.ai/v1

model_list:
  - model_name: "mistral-small-profile"
    provider: "mistral"
    model_params:
      <<: *mistral_common_params # <-- The template content will be inserted here
      model: "mistral-small-latest"
  - model_name: "mistral-large-profile"
    provider: "mistral"
    model_params:
      <<: *mistral_common_params # <-- And here as well
      model: "mistral-large-latest"
```

## 4. Other System Settings

These blocks control the global behavior of the gateway.

*   `agent_settings`: Global settings for the reasoning engine.
    *   `mcp_server_url` (**Required for ReAct**): The URL of the running tool server (`mcp_server.py`).
*   `streaming_settings`: Settings for streaming responses.
    *   `typewriter_mode`: `"proxy"` (the server simulates a "typewriter" effect) or `"client"` (transparent "as-is" forwarding).
*   `cache_settings`: Global enable/disable for caching and caching rules.
*   `key_management_settings`: Settings for API key management.
    *   `enable_quarantine`: Enables the mechanism for temporary isolation of keys that have returned temporary errors (rate limit, etc.).
*   `provider_model_lists`: **(UI Only)**. This block does not affect the gateway's logic. It is intended solely for populating the dropdown lists in the admin panel UI. It must be updated manually.