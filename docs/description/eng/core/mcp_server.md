# Technical Documentation: `mcp_server.py` — The Universal Tool Server

## 1. Overview

`mcp_server.py` is a specialized microservice based on FastAPI that implements the **"Universal Tool Server"** architectural pattern. Its main task is to encapsulate, secure, and provide standardized access to any executable resources, which we call "tools."

These tools can be anything:
*   External APIs (e.g., web search, weather services).
*   Internal computational functions (e.g., calculators, data processors).
*   Complex data processing pipelines (ETL, RAG).
*   Even other AI agents acting as "experts."

`mcp_server` acts as a centralized, strictly typed, and secure gateway that completely separates the tool execution logic from the main business logic of the AI agent.

## 2. Architectural Role and Problem Solved

Modern AI systems require access to a wide range of tools. Directly integrating these tools into the application's core (`react_driver.py`) would inevitably lead to several problems:

*   **Monolithicity**: Adding or updating any tool would require modifying and completely redeploying the entire application.
*   **Limited Scalability**: Direct integration scales poorly for complex pipelines or asynchronous tasks.
*   **Violation of the Single Responsibility Principle (SRP)**: The application core would be overloaded with the logic of interacting with each specific tool.
*   **Security Risks**: API keys and other credentials for all tools would be concentrated in one place, increasing the risk of them being leaked into logs or the LLM's context.

`mcp_server.py` solves these problems by moving all tool-related logic into a **separate, lightweight, and independent service**. It acts as a **digital safe** for all secrets related to the tools and provides a single, unified interface for calling them.

## 3. Key Components and Implementation

### 3.1. The "Universal Executor" Pattern

The module is designed as a universal platform for executing **any** tool that can be called via an HTTP POST request with a JSON body and returns a JSON response. This provides incredible flexibility.

### 3.2. Strict API Contract

The service provides a clear and standardized HTTP API:

*   `GET /`: An endpoint for a Health Check.
*   `POST /tools/{tool_name}`: A universal endpoint for calling any tool by its name.

**Pydantic** plays a key role in the contract. For each tool, a Pydantic model is defined to describe its input parameters. This ensures that no tool will be called with incorrect data—FastAPI will reject invalid requests at the web server level.

### 3.3. Secure Secret Storage

The logic for managing API keys and other secrets is **completely isolated** within this microservice.

**Example: Key for Tavily Web Search**
1.  On server startup, the `load_tavily_key()` function is called.
2.  It reads the API key from the `env/tavily.env` file.
3.  The key is set as the `TAVILY_API_KEY` environment variable, accessible **only** within the `mcp_server` process.

The main application (`react_driver.py`) never sees the key itself. It only sends a request to the `/tools/web_search` endpoint. This prevents the key from leaking into logs, network traffic between `react_driver` and the LLM, or the context of other modules.

### 3.4. Dependency Isolation

The service imports only the libraries necessary for its tools to function. This prevents "bloating" the dependencies of the main application and avoids potential version conflicts.

## 4. Practical Guides

### 4.1. Configuration and Startup

**Key Configuration:**
1.  Create an `env/` directory in the project root.
2.  For each tool requiring a key, create a corresponding file (e.g., `tavily.env`).
3.  Place the API key in this file.

**Starting the Server:**
Execute the command in the terminal from the project's root directory:
```bash
python mcp_server.py
```
The server will start and be available at `http://localhost:8002`.

### 4.2. Example Call via `curl`

You can easily test any tool directly:
```bash
curl -X POST http://localhost:8002/tools/web_search \
-H "Content-Type: application/json" \
-d '{
  "query": "What are the latest advancements in AI?",
  "max_results": 3
}'
```

### 4.3. Guide: Adding a New Tool

This is the key advantage of `mcp_server`—the ease of extension. Let's add a new "Expert Agent" tool that calls another AI agent.

**Step 1: Create the Tool Logic**

Create a new file `tools/expert_agent.py` with your tool's logic.

```python
# tools/expert_agent.py
import httpx

# The address can point to another endpoint of the same proxy
EXPERT_AGENT_ENDPOINT = "http://localhost:8001/v1/react/sessions"

async def get_expert_advice(query: str, domain: str) -> str:
    """Calls another ReAct agent to get advice."""
    payload = {
        "user_query": query,
        "model_alias": f"expert_{domain}_model",
        "reasoning_mode": "specialized_react_pattern",
    }
    async with httpx.AsyncClient() as client:
        response = await client.post(EXPERT_AGENT_ENDPOINT, json=payload, timeout=300.0)
        response.raise_for_status()
        return response.json().get("answer", "No expert advice available.")
```

**Step 2: Describe the Tool for the LLM**

Add its description to `tools/available_tools.py` so that the main agent (`react_driver`) knows about it and can use it correctly.

**Step 3: Register the Endpoint in `mcp_server.py`**

Edit `mcp_server.py` by adding a Pydantic model for request validation and the endpoint itself.

```python
# --- In the mcp_server.py file ---
# ... other imports ...
from tools.expert_agent import get_expert_advice # 1. Import our logic
from pydantic import BaseModel

# ... existing Pydantic models ...

# 2. Add a Pydantic model for the new tool
class ExpertAgentRequest(BaseModel):
    query: str
    domain: str

# ... existing endpoints ...

# 3. Add the new asynchronous endpoint
@app.post("/tools/expert_advice", summary="Get expert advice")
async def run_expert_agent(request: ExpertAgentRequest):
    """
    Calls a specialized expert agent to get a consultation.
    """
    try:
        advice = await get_expert_advice(query=request.query, domain=request.domain)
        return {"advice": advice}
    except Exception as e:
        # Logging the error here would be good practice
        raise HTTPException(status_code=500, detail=str(e))
```

**Done!** Now the main LLM agent can call `expert_advice` as one of its tools, and `mcp_server` will reliably and securely orchestrate this call.

## 5. Architectural and Business Value

*   **Security**: Isolating API keys and secrets in a separate service drastically reduces the attack surface.
*   **Extensibility**: Adding new tools does not require changing the system's core, which speeds up development and deployment.
*   **Scalability**: The tool server can be scaled independently of the main application.
*   **Simplicity and Code Cleanliness**: The main logic module (`react_driver`) operates with simple and unified HTTP calls, making its code cleaner and easier to maintain.

In essence, `mcp_server.py` is a key component for building a complex, modular, scalable, and secure AI system.