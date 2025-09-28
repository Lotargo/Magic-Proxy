# Technical Documentation: prompt_constructor.py Module

## 1. Executive Summary

`prompt_constructor.py` is a utility module that implements a strategically important function for hierarchically building system prompts. Its main task is to programmatically construct the final prompt for an AI agent by combining instructions from various sources (client, server, agent core) into a single, structured, and predictable directive. The module resolves the problem of conflicting instructions by establishing an explicit priority hierarchy and informing the LLM about this hierarchy.

## 2. Architectural Role and Problem Solved

As AI systems become more complex, an agent's behavior is determined by multiple layers of configuration: global server-side rules (security policies), dynamic client-side instructions (agent "personality"), and the core reasoning logic (ReAct pattern). Simply concatenating these instructions can confuse the model, lead to unpredictable results, or allow client instructions to override critical security rules.

`prompt_constructor.py` solves this problem by implementing a deterministic prompt assembly algorithm that:
*   Establishes a clear order of priority for instructions.
*   Structures the prompt with explicit Markdown separators, improving its readability for the LLM.
*   Includes a "meta-instruction" that directly tells the model how to resolve conflicts between different blocks.

## 3. Key Function and Algorithm

The module provides one main function, which is integrated into `react_driver.py` to prepare each request for the AI agent.

```python
def build_final_prompt(
    react_pattern_prompt: str,
    client_system_instruction: Optional[str] = None,
    client_manifests: Optional[List[str]] = None,
    server_system_instruction: Optional[str] = None,
    server_manifests: Optional[List[str]] = None,
) -> str:
    ...
```

**Prompt Assembly Algorithm:**
1.  **Meta-instruction:** The prompt always begins with a fixed "meta-instruction" that explains the rules of the game to the model:
    > "You are an AI agent. Follow the instructions below in the strict order they appear. If any instruction in a later section contradicts an instruction from a previous section, the instruction from the previous section has absolute priority."
2.  **Hierarchical Block Assembly:** The function sequentially adds blocks of instructions. Thanks to the meta-instruction, the order of addition determines the priority (earlier blocks are more important).
    *   **Block 1: CLIENT INSTRUCTIONS (Highest Priority)**
        *   **Content:** Combines `client_system_instruction` and the content of files from `client_manifests`.
        *   **Purpose:** Allows the end-user to customize the agent's behavior, define its "personality," role, or task-specific instructions. This block has absolute priority over all others.
    *   **Block 2: CORE REASONING FRAMEWORK (Medium Priority)**
        *   **Content:** `react_pattern_prompt`.
        *   **Purpose:** Contains the agent's core reasoning logic (e.g., ReAct), defining how to use tools, how to reason, and in what format to provide the answer.
    *   **Block 3: GLOBAL SERVER INSTRUCTIONS (Lowest Priority)**
        *   **Content:** Combines `server_system_instruction` and the content of files from `server_manifests`.
        *   **Purpose:** Contains global, unchangeable rules set on the server side. This is the "last line of defense" for implementing security policies, ethical constraints, defining the default "tone of voice," and other fundamental rules.

## 4. Example Final Prompt Structure

```
You are an AI agent. Follow the instructions below in the strict order they appear. If any instruction in a later section contradicts an instruction from a previous section, the instruction from the previous section has absolute priority.

### CLIENT INSTRUCTIONS (HIGHEST PRIORITY)

[Content of client_system_instruction and client_manifests]

### CORE REASONING FRAMEWORK

[Content of react_pattern_prompt with a description of the ReAct cycle]

### GLOBAL SERVER INSTRUCTIONS (LOWEST PRIORITY)

[Content of server_system_instruction and server_manifests with security rules]
```

## 5. Architectural Value and Business Impact

Integrating `prompt_constructor.py` into the main workflow has transformed the AI agent from a flexible prototype into a mature, reliable, and secure AI product.
*   **Improved Security and Reliability (Guardrails):** The module allows for the implementation of unchangeable server-side instructions (ethical constraints, security policies) as a "last line of defense" that are present in every request and cannot be overridden by the client.
*   **Enhanced Controllability:** Explicitly stating priorities in the prompt has significantly increased the predictability of the model's behavior, which is critical for production systems where stable response quality is required.
*   **Creating a Unique Service "Personality" (Branding):** The module allows for setting a base "tone of voice" at the server instruction level, shaping the AI agent's unique communication style while still allowing for client-side customization.
*   **Platform for Advanced Prompt Engineering:** The constructor is the foundation for implementing complex prompt structuring techniques, allowing for flexible management of the layout and format of each element to achieve optimal performance with various LLMs.