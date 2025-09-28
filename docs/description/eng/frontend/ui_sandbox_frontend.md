# User Guide: UI Sandbox (Frontend)

## 1. Introduction: What is the Sandbox?
The UI Sandbox is an interactive control and testing panel for "MagicProxy". It provides developers and prompt engineers with a convenient environment for sending requests, visualizing the work of AI agents, and viewing and editing the gateway's configuration in real-time.

## 2. Interface Overview
The interface consists of three main sections, switchable via the side navigation panel (sidebar):

*   **Playground (`playground-tab`)**: The main "sandbox" for launching AI sessions and observing their work.
*   **Prompts (`prompts-tab`)**: An editor for managing system prompt and manifest files.
*   **Configuration (`admin-tab`)**: A powerful editor for modifying the main `proxy_config.yaml` configuration file.

### 2.1. "Playground" Section
This is your main workspace. It is divided into three parts:

**Right Sidebar (`right-sidebar`)**: Here you configure the parameters for the upcoming session.
*   **Select models to run**: A list of all `model_alias` from `proxy_config.yaml`. You can select one or more models to run simultaneously to compare their responses to the same request.
*   **Safety Settings (for Gemini models)**: Four switches to disable Gemini's safety filters (Harassment, Hate Speech, etc.).
*   **ReAct Settings (for the current session)**: Allow you to override the system prompt and manifest for a specific agent on the fly for a single run, without changing the global configuration.

**Central Area (`race-track`)**: This is the "chat" or "event feed". The entire session execution process is displayed here in real-time: the agent's thoughts, tool calls, and the final answer.

**Input Area (`chat-input-area`)**:
*   **`prompt-input`**: A text field for entering your request.
*   **`run-button`**: A button to start the session with the settings selected in the right sidebar.

### 2.2. "Prompts" Section
This section allows you to manage the text files used in ReAct patterns.

**System Prompts Editor**:
*   **`system-prompt-select`**: A dropdown list to select a file from the `system_prompts/` directory.
*   **`system-prompt-editor`**: A text editor for changing the content of the selected prompt.
*   **`system-prompt-diff-output`**: An area where your changes are highlighted compared to the saved version.
*   **`save-prompt-btn`**: A button to save the changes to the file on the server.

**Manifests Editor**: A similar interface for working with files in the `manifests/` directory.

### 2.3. "Configuration" Section
This section provides full control over `proxy_config.yaml`.
*   **General Settings (`general-config-editor`)**: A text editor for modifying all sections except `model_list`.
*   **Model List Editor (`model-list-editor-container`)**: (Key feature) A specialized, structured interface for editing `model_list`. It allows you to conveniently add, delete, and modify model profiles without risking breaking the YAML formatting.
*   **`add-model-btn`**: A button to add a new empty model profile to the list.
*   **Changes (Diff) (`diff-output`)**: Visually shows all your changes in `proxy_config.yaml` before saving.

**Action Buttons**:
*   **`save-config-btn`**: Saves the changes to `proxy_config.yaml` on the server.
*   **`restart-btn`**: Saves the changes and sends a command to the server to restart, immediately applying the new configuration.

## 3. Practical Usage Scenarios

### Scenario 1: Running and Comparing Two Models
1.  Go to the **Playground** tab.
2.  In the right sidebar, in the "Select models to run" list, activate the switches for `gemini-agent` and `mistral-agent`.
3.  In the input area at the bottom, enter the query: "What is the latest news in AI?".
4.  Click the "Start Session" button (`run-button`).
5.  In the central area (`race-track`), two columns will appear, one for each agent. You will be able to observe and compare their thought processes and final answers in real-time.

### Scenario 2: Changing an Agent's Personality "On the Fly"
1.  Go to the **Playground** tab.
2.  In the right sidebar, in the "ReAct Settings" block, select `gemini-agent` in the first dropdown list.
3.  Below, in the "System Prompt" list, select a prompt that describes, for example, a pirate personality.
4.  In the "Select models to run" list, make sure only `gemini-agent` is active.
5.  Send the request. The `gemini-agent` will execute it using the pirate personality you selected, but this change will not be saved to `proxy_config.yaml` and will only affect the current session.

### Scenario 3: Adding a New Model and Restarting the Server
1.  Go to the **Configuration** tab.
2.  In the Model List Editor, click the "+ Add Model" button. A new empty form will appear.
3.  Fill in the fields: `model_name` (e.g., `my-new-model`), `provider` (`openai`), `model` (`gpt-4o`), etc.
4.  In the "Changes (Diff)" area, you will see the new element added to `model_list`.
5.  Click the "Save and Restart" button (`restart-btn`).
6.  The server will restart. Returning to the **Playground** tab, you will see that `my-new-model` has appeared in the list of available models to run.