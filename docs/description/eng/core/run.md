# Technical Documentation: run.py Module

## 1. Executive Summary

`run.py` is a universal orchestrator script for local development with "hot-reload" support. Its purpose is to launch all the services that make up the "Universal AI Gateway" application as independent subprocesses, aggregate their logs into a single stream, and automatically restart them when changes in the source code are detected. This provides the fastest and most comfortable iterative development experience.

## 2. Architectural Role and Problem Solved

A modern microservice application consists of several components that must run simultaneously. During development, manually restarting each service after every minor code change is a slow, tedious, and error-prone process.

`run.py` solves this problem by creating a lightweight "all-in-one" environment that:
*   **Simplifies startup:** Launches the entire system with a single command (`python run.py`).
*   **Centralizes logging:** Gathers logs from all services in one console.
*   **Implements "Hot-Reload":** (Key feature). The script launches `uvicorn` and `mcp_server` in modes that monitor `.py` files for changes. As soon as you save a file with changes, the corresponding service automatically reloads, immediately applying the new code. This drastically speeds up the "code -> test" cycle.
*   **Ensures proper termination:** Guarantees that all processes will be correctly stopped with a Ctrl+C signal.

**Important:** `run.py` is not intended for a production environment. The `--reload` flag is inefficient and unsafe for real-world operation. Its role is to be a developer's tool. For production deployment, `docker-compose.yml` should be used.

## 3. Key Components and Logic

### 3.1. Starting Subprocesses (start_process)

*   **Mechanism:** Instead of simply importing and calling functions, the script uses `subprocess.Popen` to launch each service (`main.py` via `uvicorn` and `mcp_server.py`) in a separate, isolated operating system process. This emulates a real microservice architecture as closely as possible.
*   **Encoding Handling:** The script forcibly sets the `PYTHONIOENCODING='utf-8'` environment variable for each subprocess. This solves common problems with displaying Unicode characters (e.g., Cyrillic) in logs on Windows OS.

### 3.2. Real-Time Log Aggregation (stream_output)

*   **Mechanism:** For each running subprocess, a separate thread (`threading.Thread`) is created, which reads the `stdout` of that process line by line in a loop.
*   **Daemon Threads:** The logger threads are daemonic (`daemon = True`), which means they will not prevent the main script from exiting.

### 3.3. Startup Orchestration and "Hot-Reload"

The central element is the configuration of commands for launching the subprocesses.

```python
# Snippet from main() in run.py
mcp_command = [sys.executable, "-u", str(project_root / "mcp_server.py")]
main_proxy_command = [
    sys.executable, "-u",
    "-m", "uvicorn",
    "main:app",
    "--host", "0.0.0.0",
    "--port", "8001",
    "--reload"  # <--- KEY FLAG
]
```

**How "hot-reload" works:**
*   **For main-proxy:** The startup command explicitly includes the `--reload` flag for `uvicorn`. Uvicorn automatically starts monitoring all `.py` files in the current directory and its subdirectories. When any of these files are saved, `uvicorn` gracefully restarts the worker with `main:app`, applying the changes.
*   **For mcp-server:** Although `mcp_server.py` is run directly, the `uvicorn` instance inside it (in the `if __name__ == "__main__"` block) also uses the `reload=True` option. Thus, it has the same behavior.
*   **Applying changes "on the fly":** This function is especially useful when working with the UI. For example, if you change the configuration through the admin panel (which edits `proxy_config.yaml`), the `main-proxy` server won't notice. But if you make a change to the code that processes this configuration, the server will instantly restart and begin working with the new logic.

## 4. Usage

To launch the entire application ecosystem in a rapid iterative development mode, simply run one command:

```bash
python run.py
```

Now you can:
1.  Open any `.py` file of the project in your editor.
2.  Make a change and save the file.
3.  Watch in the console where `run.py` is running as the corresponding service (e.g., `[Main Proxy]`) automatically reloads, displaying restart messages.
4.  Immediately test the new behavior via the API, without any manual actions.

To stop the entire system, just press `Ctrl+C` in the same terminal window.