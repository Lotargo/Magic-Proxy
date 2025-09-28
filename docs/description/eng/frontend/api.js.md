# Technical Documentation: api.js Module

## 1. Executive Summary
`api.js` is the Data Access Layer for the frontend application. Its sole purpose is to encapsulate all network interaction logic with the "MagicProxy" backend API. The module provides a set of asynchronous, typed functions that abstract away fetch requests, error handling, and URLs, providing the rest of the code (e.g., `ui.js`) with a clean and semantically clear interface for working with data.

## 2. Architectural Role and Purpose
Using this module follows the principle of Separation of Concerns. Instead of scattering `fetch` requests throughout the `ui.js` code, all network logic is centralized in one place. This provides several key advantages:
*   **Maintainability:** If an API URL or response format changes, the changes only need to be made in one file.
*   **Readability:** The code in `ui.js` becomes declarative. Instead of `fetch(...)`, it calls `api.fetchConfig()`, which is self-explanatory.
*   **Error Handling:** The module provides uniform error handling. Each function checks `response.ok` and, in case of an unsuccessful HTTP status, throws an `Error` with an informative message, which simplifies debugging.
*   **Configuration:** The base API URL (`API_BASE_URL`) is defined as a constant in one place, making it easy to switch between environments (e.g., from `localhost` to a `production-server`).

## 3. API Function Reference
The module's functions are logically grouped by areas of responsibility, corresponding to the UI tabs and features.

### 3.1. Configuration and Admin Panel Management
These functions are used to initialize and operate the "Configuration" tab.

#### `async function fetchConfig()`
*   **Endpoint:** `GET /admin/config`
*   **Purpose:** Loads the current content of the `proxy_config.yaml` file from the server.
*   **Returns:** `{ content: "..." }`

#### `async function fetchProviderModels()`
*   **Endpoint:** `GET /admin/provider_models`
*   **Purpose:** Loads the list of available models for each provider, used to populate the dropdowns in the `model_list` editor.
*   **Returns:** `{ "google": ["gemini-1.5-pro", ...], "mistral": [...] }`

#### `async function saveConfig(content, restartAfterSave)`
*   **Endpoint:** `POST /admin/config`
*   **Purpose:** Sends the new content of `proxy_config.yaml` to the server to be saved.
*   **Parameters:**
    *   `content: string`: The new content of the file.
    *   `restartAfterSave: boolean`: If `true`, the `restartServer()` function will be called after saving.

#### `async function restartServer()`
*   **Endpoint:** `POST /admin/restart`
*   **Purpose:** Sends a command to the server for an immediate restart.

### 3.2. Prompts and Manifests Management
These functions are used for the "Prompts" tab.

#### `async function fetchPromptsAndManifests()`
*   **Endpoint:** `GET /admin/prompts`
*   **Purpose:** Loads lists of all available prompt and manifest files.
*   **Returns:** `{ "prompts": ["path/to/prompt1.txt", ...], "manifests": [...] }`

#### `async function fetchFileContent(path)`
*   **Endpoint:** `GET /admin/prompt_content`
*   **Purpose:** Loads the text content of a specific file by its path.
*   **Returns:** `{ "path": "...", "content": "..." }`

#### `async function saveFileContent(path, content)`
*   **Endpoint:** `POST /admin/prompt_content`
*   **Purpose:** Saves new content for the specified file.

### 3.3. 'Playground' Logic
These functions are used to initialize and run sessions in the "Playground" tab.

#### `async function fetchRunnableModels()`
*   **Endpoint:** `GET /v1/models/all-runnable`
*   **Purpose:** Loads a list of all `model_alias` available for running, including their metadata (e.g., `is_agent`).
*   **Returns:** `[{ "id": "gemini-agent", "name": "gemini-agent", "is_agent": true, ... }, ...]`

#### `async function fetchReactPatterns()`
*   **Endpoint:** `GET /admin/react_patterns`
*   **Purpose:** Loads a list of all automatically discovered ReAct patterns.

#### `async function runSimpleChat(payload)`
*   **Endpoint:** `POST /v1/chat/completions`
*   **Purpose:** Initiates a simple, non-agent chat request.
*   **Returns:** `Response`. Important: This function does not read the response body but returns the raw `Response` object, as `ui.js` will read streaming data from it (`response.body`) itself.

#### `async function runAgent(payload)`
*   **Endpoint:** `POST /v1/react/sessions`
*   **Purpose:** Initiates a complex, agent-based ReAct session.
*   **Returns:** `Response`. Similar to `runSimpleChat`, it returns the `Response` object for subsequent stream processing in `ui.js`.