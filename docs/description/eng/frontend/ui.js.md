# Technical Documentation: Frontend Logic (ui.js)

## 1. Executive Summary
`ui.js` is the main JavaScript module that brings the entire UI Sandbox to life. It manages the application's state, handles user interactions, communicates with the backend API, and dynamically updates the DOM to display data. The module is built around an asynchronous model with a clear separation into initialization, event handling, and component rendering.

## 2. Architecture and State Management

### 2.1. Global State
At the top of the file, a number of global variables are defined that serve as an in-memory cache and the source of truth for the entire application.
*   `ALL_MODELS_DATA`, `ALL_PROMPTS`, `ALL_MANIFESTS`, `ALL_PATTERNS`: These arrays are populated once when the page loads. All subsequent operations (rendering lists, starting sessions) use this cached data, which minimizes the number of API requests and speeds up the interface.
*   `originalConfigContent`, `originalPromptContent`, `originalManifestContent`: These variables store the "reference" state of the configuration files at the time they were loaded. They are used for comparison (diff) to determine if changes have been made.

### 2.2. `api.js` Module (API Abstraction)
All code responsible for HTTP requests (`fetch`) is moved to a separate `api.js` module. `ui.js` never works with `fetch` directly but calls high-level functions like `api.fetchRunnableModels()` or `api.saveConfig()`. This is a classic "Data Access Layer" pattern that makes the main code cleaner and easier to test.

## 3. Application Lifecycle

*   **`DOMContentLoaded` (Entry Point):**
    *   The script starts working after the HTML is fully loaded.
    *   `loadAllInitialData()` is called first.
*   **`loadAllInitialData()` (State Initialization):**
    *   This asynchronous function simultaneously (via `Promise.all`) requests all necessary data from the backend: the list of models, prompts, manifests, and ReAct patterns.
    *   After being received, the data is written to global variables.
*   **Tab Initialization (`init*`):**
    *   After the data is loaded, the `initPromptsTab()`, `initAdminPanel()`, and `initPlayground()` functions are called sequentially. Each of these functions is responsible for rendering the initial state and attaching event handlers for its respective tab.

## 4. Key Logic Blocks

### 4.1. "Playground" Logic (`initPlayground`, `handleSessionRequest`)
*   **Rendering:** `renderModelCheckboxes()` and `renderPlaygroundReactOverrides()` dynamically create HTML for the model lists and ReAct settings based on data from `ALL_MODELS_DATA`, `ALL_PROMPTS`, and `ALL_MANIFESTS`.
*   **Starting a Session (`handleSessionRequest`):**
    *   Gathers all data from the UI: the request text, IDs of selected models, security settings, and ReAct prompt "overrides".
    *   **Distinguishes between agents and simple chats:** Checks the `modelData.is_agent` flag.
        *   If `true`, it forms a complex `payload` for the `/v1/react/sessions` endpoint and calls `api.runAgent()`.
        *   If `false`, it forms a simple `payload` for the `/v1/chat/completions` endpoint and calls `api.runSimpleChat()`.
    *   Creates a separate column in the UI for each running model (`createAgentColumn`).
    *   Passes the streaming response body (`response.body`) to specialized parser functions (`processReActStream` or `processOpenAIStream`).

### 4.2. Stream Processing (`processReActStream`, `processOpenAIStream`)
These asynchronous functions are the "heart" of the interactivity.
*   **`processOpenAIStream`:** A relatively simple parser. It reads the SSE stream, extracts `content` from the JSON chunks, and gradually accumulates it, updating the UI. At the end, it renders the full response as Markdown.
*   **`processReActStream`:** Significantly more complex. It is a state machine that parses structured events from the `sse_driver`.
    *   It tracks the current state (whether a "thought," "action," or "final answer" is in progress).
    *   Dynamically creates and updates `details` (`<summary>`) elements for each reasoning step.
    *   Reacts to different `event_type`s (`AgentThoughtStream`, `AgentToolCallStart`, `FinalAnswerStreamEnd`, etc.) to correctly render different parts of the UI.
    *   Has a built-in timeout: if no data is received from the stream for a long time, it forcibly terminates the processing to avoid a "frozen" UI.

### 4.3. Admin Panel Logic (`initAdminPanel`, `handleSaveConfig`)
*   **Structured Editing:** `model_list` is not edited as raw YAML. The `renderModelListEditor` function parses this array and creates an interactive table where each field is a separate `input` or `select`. This protects the user from YAML syntax errors.
*   **"Live" Diff:** On any change in the editors (both in the table and in the text fields), `renderDiff` is called. This function:
    *   Gathers the current UI state into a new JavaScript object (`getCurrentConfigAsObject`).
    *   Converts it to a YAML string.
    *   Compares it with `originalConfigContent`.
    *   Uses the `Diff2HtmlUI` library to render a clear side-by-side comparison.
*   **Saving and Restarting:** `handleSaveConfig` sends the full new content of `proxy_config.yaml` to the backend. The `restartAfterSave` parameter controls which endpoint will be called (`/admin/config` or `/admin/restart`).

### 4.4. Prompt Editor Logic (`initPromptsTab`)
Works on a similar principle to the admin panel: it loads the file content, stores it in `original*Content`, and on changes in the `textarea`, it renders a `diff` and enables the save button.