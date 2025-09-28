# Technical Documentation: `react_driver.py` — The Core of Cognitive Enrichment

## 1. Overview

`react_driver.py` is the **R&D Core** and central orchestrator of "Magic Proxy". This module implements a complex asynchronous framework based on the **ReAct (Reasoning and Acting)** pattern.

However, its goal is not to create an ordinary chatbot, but to perform **Cognitive Augmentation** of requests in real-time. It acts as a "smart" preprocessor that simulates or enhances the cognitive abilities of the target LLM by generating a deep, structured reasoning context *before* the model gives its final answer.

## 2. Concept: Reasoning-as-a-Service

A standard proxy server simply forwards requests. "Magic Proxy" with `react_driver` transforms them. The module offers "Reasoning-as-a-Service," solving fundamental problems of modern AI models:

*   **Endowing with thought**: Allows "weak" and inexpensive models, incapable of complex independent reasoning, to perform multi-step tasks and use external tools.
*   **Enhancing abilities**: Improves the quality, depth, and reliability of responses from even the most powerful models by forcing them to follow a predictable and verifiable reasoning process.
*   **"Virtual fine-tuning" via prompting**: Emulates the behavior of a fine-tuned model for specific tasks, but does so "on the fly" through advanced, iterative prompt engineering. This is significantly faster and cheaper than actual fine-tuning.

In essence, `react_driver` transforms the proxy from a passive router into an **active intelligent gateway** that adds value to every request.

## 3. Architecture and Key Patterns

To implement this complex task, the module uses a fault-tolerant, distributed architecture.

*   **Decoupled Execution Framework (CQRS + Kafka)**: A request for "enrichment" is not executed immediately. It is sent as a command to **Kafka**, and a separate pool of background workers asynchronously performs the resource-intensive reasoning process. This ensures high performance and scalability.
*   **Transparent Feedback (Redis Pub/Sub + SSE)**: Despite the asynchronicity, the client can observe the entire cognitive enrichment process in real-time. `react_driver` uses **Redis Pub/Sub** and **Server-Sent Events (SSE)** to stream "thoughts," tool calls, and intermediate results.
*   **Iterative Reasoning Engine (`AgentStepProcessor`)**: This is the "brain" of the process. Its goal is not just to solve the task, but to build a **"scratchpad"**—a detailed, structured trace of reasoning (`Thought -> Action -> Observation`). This scratchpad is the main artifact that is then used to generate the final, enriched answer.
*   **Fault-Tolerant LLM Call (`_execute_react_step_with_fallback`)**: Ensures that the long and complex reasoning process is not interrupted by the failure of a single API key or even an entire provider, ensuring the reliability of the entire R&D process.

## 4. Platform for Creating Reasoning Chains

`react_driver` is designed not for a single ReAct pattern, but as a flexible **platform for creating, testing, and deploying custom reasoning chains**. Instead of forcing users to learn complex frameworks, "Magic Proxy" offers an extremely simple, file-based way to define new cognitive architectures.

### 4.1. Examples of Possible Patterns

*   **Standard ReAct (already implemented)**: The classic `Thought -> Action -> Observation` cycle for using tools.
*   **"Deep Research" Pattern**: A multi-step pattern that forces the model to first create a `Plan`, then iteratively `Search` for information, `Synthesize` the findings, and only then provide the `Final Answer`.
*   **"Expert Delegation" Pattern**: A pattern where the main orchestrator model analyzes the task and, instead of solving it directly, calls other, more specialized models or agents via `mcp_server.py` as tools.

### 4.2. Guide: Creating a New Pattern in 3 Steps

The key advantage of the system is **automatic discovery**. The system does not need to be "told" about new patterns. It's enough to create a file in the right folder.

**Step 1: Create a file in `react_patterns/`**
The filename must end with `_react.py`. For example, `deep_research_react.py`.
```
.
├── react_patterns/
│   ├── basic_react.py
│   └── deep_research_react.py  # <-- Your new file
└── react_driver.py
```
The system will automatically find this file and register the `deep_research_react` pattern.

**Step 2: Define the prompt structure**
Inside the file, create a `get_prompt_structure()` function that returns a list of dictionaries describing the reasoning steps.
```python
# react_patterns/deep_research_react.py
def get_prompt_structure():
    return [
        {"tag": "PLAN", "content": "You must create a step-by-step plan..."},
        {"tag": "EXECUTION", "content": "Execute the plan..."},
        {"tag": "SYNTHESIS", "content": "Synthesize the gathered information..."},
        {"tag": "FINAL_ANSWER", "content": "Provide the final answer..."}
    ]
```

**Step 3: Activate the pattern in `proxy_config.yaml`**
Bind the new pattern to a model profile to create a new "virtual" AI intelligence.
```yaml
# proxy_config.yaml
model_list:
  - model_name: "some_powerful_model"
    provider: "openai"
    model_params:
      model: "gpt-4-turbo"
      agent_settings:
        reasoning_mode: "deep_research_react" # <--- Specify our new pattern

router_settings:
  model_group_alias:
    "gpt4-deep-research": # <--- Create an alias for the new intelligence
      - "some_powerful_model"
```
**Done!** Now, requests to `model: "gpt4-deep-research"` will force `gpt-4-turbo` to follow your new, complex "Deep Research" pattern.

### 4.3. Limitations and Future Directions

*   **Limitation**: Currently, the engine may be unstable with models that already have strong built-in ReAct patterns (e.g., Gemini Pro). Attempting to impose an external reasoning chain on a model that is already trying to reason on its own can lead to conflicts.
*   **Recommendation**: `react_driver` provides the most value when augmenting models with limited native reasoning abilities (e.g., gpt-4o-mini, models from Mistral, Llama, etc.).
*   **Future Development**: Resolving this conflict is an active area of research for the project.

## 5. Lifecycle of an "Enriched" Request

1.  The client sends a request to the `/v1/react/sessions` endpoint, specifying a `model_alias` for which `reasoning_mode` is enabled in `proxy_config.yaml`.
2.  `react_driver` sends a task to **Kafka** and opens an **SSE connection** with the client via Redis.
3.  A background worker receives the task and starts the `AgentStepProcessor`.
4.  The iterative **context-building cycle (scratchpad)** begins:
    a. `prompt_constructor` assembles the hierarchical prompt for the current step.
    b. `_execute_react_step_with_fallback` reliably calls the LLM.
    c. The worker parses the "Thought" and "Action," streaming them to the client for transparency.
    d. If there is an "Action," the tool is called via `mcp_server.py`.
    e. The `scratchpad` is updated with the result (`Observation`).
    f. The cycle repeats until the model generates the `<FINAL_ANSWER>` tag.
5.  The content of the `<FINAL_ANSWER>` tag, generated based on the entire accumulated context, is streamed to the client as the final, enriched answer.

## 6. Architectural and Business Value

*   **Cognitive Augmentation**: Allows for obtaining significantly higher quality responses from LLMs.
*   **Cost-Effectiveness**: Makes it possible to use cheaper models to solve complex tasks, reducing operational costs.
*   **Flexibility and Controllability**: Provides a powerful R&D tool for customizing model behavior without the need for fine-tuning.
*   **Unique Selling Proposition (USP)**: The key feature of the proxy is not just routing and fallback, but the ability to intelligently transform and enrich AI requests on the fly.