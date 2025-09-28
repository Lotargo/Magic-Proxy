# Technical Documentation: `tools/available_tools.py` — The Tool Registry

## 1. Overview and Architectural Role
`tools/available_tools.py` is a **dynamic registry** and "user manual" for all tools available to the AI agent. Its main task is not to *execute* logic, but to **describe (declare)** the tool functions.

This module acts as a **"bridge" between thought (`Thought`) and action (`Action`)** in the ReAct architecture. For an LLM to generate a correct "action" (a tool call in JSON format), it must be provided with a "menu" of available tools beforehand. `available_tools.py` is the source of this "menu."

It solves two key tasks:
1.  **For `react_driver`**: It provides the `get_tool_descriptions_for_llm()` function, which returns a machine-readable JSON list of all tools. This JSON is inserted into the agent's system prompt, "teaching" it what tools exist and how to call them.
2.  **For `mcp_server`**: It serves as an implicit **"contract"**. The name of each tool function (e.g., `web_search`) directly corresponds to an endpoint (`/tools/web_search`) that must be implemented in `mcp_server.py`.

## 2. Implementation: Generating a Schema from Docstrings
The module implements a powerful mechanism for **automatically generating a JSON schema** based on standard Python docstrings and type annotations. This drastically simplifies the process of adding new tools.

### 2.1. Function Stubs
Each available tool is represented as a **function stub**.

```python
def web_search(query: str, max_results: int = 5) -> str:
    """
    Performs a general web search for a given query.
    Use for finding up-to-date information or facts.

    :param query: The search query.
    :param max_results: The maximum number of results to return.
    :return: A string with the search results.
    """
    pass
```

*   **Function Body**: Intentionally empty (`pass`). The actual execution logic resides in `mcp_server.py`. This file only serves to describe the interface.
*   **Function Signature**: `web_search(query: str, max_results: int = 5)` is a strict contract.
    *   **Name (`web_search`)**: Becomes the `tool_name` in the JSON schema.
    *   **Arguments (`query`, `max_results`)**: Become keys in the `parameters` object of the schema.
    *   **Type Annotations (`str`, `int`)**: Used to determine the parameter type in the JSON schema (`string`, `integer`).
    *   **Default Values (`= 5`)**: Used to determine if a parameter is required.
*   **Docstring**: This is the source of text descriptions for the LLM. It has a strict format:
    *   **First part (before `:param`)**: Becomes the general `description` of the tool. This is the most important field that helps the LLM choose the right tool.
    *   **`:param name: description` sections**: Used to generate descriptions for each parameter in the JSON schema.

### 2.2. Automatic Schema Generator (`get_tool_descriptions_for_llm`)
This is the "magic" part of the module. The function uses the standard Python libraries `inspect` and `typing` for **introspection (self-analysis)** of the function stubs.

**How it works:**
1.  Gets a list of function stubs.
2.  Iterates through each function and, using introspection, reads its name, docstring, parameters, type annotations, and default values.
3.  Based on all this information, it **dynamically constructs a JSON object** that conforms to the format understood by modern LLMs (e.g., OpenAI Functions/Tools).

## 3. Guide: Adding a New Tool
The module is designed so that adding new tools is as simple as possible.

**Step 1: Describe the tool as a function stub**

Create a new function in `available_tools.py`, following the format. For example, for a calculator:

```python
def calculator(expression: str) -> float:
    """
    Calculates a mathematical expression.
    Use for any mathematical calculations.

    :param expression: A mathematical expression as a string (e.g., "2+2*5").
    :return: A floating-point number that is the result of the calculation.
    """
    pass
```

**Step 2: Register the function in the list**

Add the name of the new function to `tools_list` inside the `get_tool_descriptions_for_llm` function.

```python
def get_tool_descriptions_for_llm():
    tools_list = [web_search, calculator] # <--- Added here
    # ... rest of the code unchanged
```

**Step 3: Implement the endpoint in `mcp_server.py`**

Don't forget to create a corresponding `/tools/calculator` endpoint in `mcp_server.py` that will perform the actual calculation logic. (See the `mcp_server.py` documentation for details).

**Done!** The next time `react_driver` starts, it will call `get_tool_descriptions_for_llm()`, which will automatically generate a JSON schema for the new "calculator" tool and include it in the system prompt. The LLM will immediately "learn" about the new capability.