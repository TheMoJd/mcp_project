# MCP Project: Chatbot with Research Paper Tools

## Overview

This project implements a chatbot application that leverages the Model Context Protocol (MCP) to interact with various backend services. The primary example service included is a "research server" that allows users to search for academic papers on arXiv, store their information, and retrieve details about them. The chatbot uses an Anthropic language model (Claude) to understand user queries and can utilize tools provided by MCP servers to fulfill requests.

## What is the Model Context Protocol (MCP)?

The Model Context Protocol (MCP) is a standardized communication protocol designed to facilitate interaction between Large Language Model (LLM) applications (referred to as **Hosts** or **Clients**) and various backend services or tools (referred to as **Servers**). It enables LLMs to access and utilize external knowledge, execute actions, and integrate with other systems seamlessly.

MCP is built on a client-server architecture:

*   **Hosts**: These are the LLM applications (e.g., a desktop application, an IDE extension) that initiate connections to MCP servers.
*   **Clients**: Reside within the host application and maintain a 1:1 connection with an MCP server. They manage the communication flow.
*   **Servers**: These are processes that provide context, tools (callable functions), and prompts (pre-defined query templates) to clients.

### Core Architectural Components

1.  **Protocol Layer**:
    *   Manages message framing, ensuring that messages are correctly structured and understood by both client and server.
    *   Handles the linking of requests to their corresponding responses.
    *   Defines high-level communication patterns (e.g., request/response, notifications).

2.  **Transport Layer**:
    *   Responsible for the actual transmission of data between the client and server.
    *   MCP is transport-agnostic, meaning it can operate over different communication mechanisms. Common transports include:
        *   **Stdio Transport**: Uses standard input (stdin) and standard output (stdout) for communication. Ideal for local processes running on the same machine.
        *   **HTTP with Server-Sent Events (SSE) Transport**: Uses HTTP POST for client-to-server messages and SSE for server-to-client messages. Suitable for remote communication.
    *   All transports typically use JSON-RPC 2.0 for structuring the messages.

### Message Types

MCP defines several main types of messages, following the JSON-RPC 2.0 specification:

1.  **Requests**: Sent by one party (client or server) expecting a response from the other.
2.  **Results**: Successful responses to requests, containing the requested data.
3.  **Errors**: Indicate that a request failed, providing details about the failure.
4.  **Notifications**: One-way messages that do not expect a response.

### Connection Lifecycle

A typical MCP connection follows these stages:

1.  **Initialization**:
    *   The client sends an `initialize` request to the server, including its protocol version and capabilities.
    *   The server responds with its own protocol version and capabilities.
    *   The client sends an `initialized` notification to acknowledge successful initialization.
2.  **Message Exchange**:
    *   Once initialized, the client and server can exchange requests, results, errors, and notifications.
    *   This is where tools are called, resources are read, and prompts are executed.
3.  **Termination**:
    *   Either the client or the server can initiate the termination of the connection (e.g., via a `close()` command or due to transport disconnection).

## Project Components

### 1. `mcp_chatbot.py` - The Chatbot Application

This is the main entry point for the user interaction. It's an asynchronous application that:

*   **Connects to MCP Servers**: Reads server configurations from [`server_config.json`](/home/mopolaria/dev/mcp-project/mcp_project/server_config.json) and establishes connections to each defined server using stdio transport.
*   **Manages Sessions**: Keeps track of active MCP sessions for each server.
*   **Discovers Capabilities**: Upon connection, it lists available tools, prompts, and resources from each server.
    *   `available_tools`: Stores information about tools that the Anthropic LLM can use.
    *   `available_prompts`: Stores information about pre-defined prompts for user convenience.
    *   `sessions`: Maps tool names, prompt names, or resource URIs to their respective MCP client sessions.
*   **Interacts with Anthropic LLM**:
    *   Uses the `anthropic` Python SDK to send user queries and conversation history to a Claude model.
    *   Passes the `available_tools` to the LLM, enabling it to request tool calls.
*   **Handles Tool Calls**:
    *   When the LLM decides to use a tool, the chatbot receives a `tool_use` content block.
    *   It identifies the correct MCP session for the requested tool and calls `session.call_tool()` with the provided arguments.
    *   The result from the tool is then sent back to the LLM as a `tool_result` to continue the conversation.
*   **Provides Resource Access**:
    *   Allows users to fetch content from MCP resources using `@<resource_uri>` syntax (e.g., `@folders`, `@llm`).
    *   It uses `session.read_resource()` to get the content.
*   **Executes Prompts**:
    *   Allows users to list and execute pre-defined prompts using `/prompts` and `/prompt <name> <arg1=value1>` commands.
    *   It uses `session.get_prompt()` to retrieve the prompt content and then processes it with the LLM.
*   **Chat Loop**: Provides a command-line interface for users to interact with the chatbot.

### 2. `research_server.py` - Research MCP Server

This Python script implements an MCP server specifically for research-related tasks, primarily interacting with the arXiv API. It uses `FastMCP` from the `mcp.server` library for quick setup.

*   **Initialization**: `mcp = FastMCP("research")` creates an MCP server instance named "research".
*   **Tools**:
    *   `@mcp.tool() search_papers(topic: str, max_results: int = 5) -> List[str]`:
        *   Searches arXiv for papers matching the given `topic`.
        *   Stores paper metadata (title, authors, summary, PDF URL, published date) in a JSON file: `papers/<topic_sanitized>/papers_info.json`.
        *   Returns a list of paper IDs found.
    *   `@mcp.tool() extract_info(paper_id: str) -> str`:
        *   Searches through all `papers_info.json` files in subdirectories of the `papers` directory.
        *   Returns a JSON string with the paper's information if found, otherwise an error message.
*   **Resources**:
    *   `@mcp.resource("papers://folders") get_available_folders() -> str`:
        *   Lists all topic folders (subdirectories within the `papers` directory that contain a `papers_info.json` file).
        *   Returns a Markdown formatted list of available topics.
    *   `@mcp.resource("papers://{topic}") get_topic_papers(topic: str) -> str`:
        *   Retrieves and formats detailed information about papers for a specific `topic`.
        *   Reads the corresponding `papers/<topic_sanitized>/papers_info.json` file.
        *   Returns a Markdown formatted string with details for each paper (title, ID, authors, published date, PDF URL, summary).
*   **Prompts**:
    *   `@mcp.prompt() generate_search_prompt(topic: str, num_papers: int = 5) -> str`:
        *   Generates a detailed prompt for an LLM (like Claude) to guide it in searching for academic papers using the `search_papers` tool and then synthesizing the findings.
*   **Server Execution**:
    *   If run directly (`if __name__ == "__main__":`), it starts the MCP server using stdio transport: `mcp.run(transport='stdio')`.

### 3. `server_config.json` - Server Configuration

This JSON file defines the MCP servers that the [`mcp_chatbot.py`](/home/mopolaria/dev/mcp-project/mcp_project/mcp_chatbot.py) application should connect to.

```json

{
    "mcpServers": {
        "filesystem": { // Example server (not fully implemented in this project's Python code)
            "command": "npx",
            "args": [
                "-y",
                "@modelcontextprotocol/server-filesystem",
                "."
            ]
        },
        "research": { // Configuration for the research_server.py
            "command": "uv", // Assumes 'uv' is used as a Python environment/runner
            "args": ["run", "research_server.py"]
        },
        "fetch": { // Example server (not fully implemented in this project's Python code)
            "command": "uvx",
            "args": ["mcp-server-fetch"]
        }
    }
}
```
The chatbot will attempt to launch and connect to each server listed under `mcpServers`. The `command` and `args` specify how to start each server process.

### 4. `papers/` Directory - Data Storage

This directory is used by the [`research_server.py`](/home/mopolaria/dev/mcp-project/mcp_project/research_server.py) to store information about academic papers.

*   When `search_papers` is called for a topic (e.g., "LLM"), a subdirectory is created (e.g., `papers/llm/`).
*   Inside this subdirectory, a `papers_info.json` file is created/updated. This file contains a dictionary where keys are paper IDs and values are objects with paper details.
    *   Example: [`papers/llm/papers_info.json`](/home/mopolaria/dev/mcp-project/mcp_project/papers/llm/papers_info.json)

### 5. `main.py`

A simple Python script that prints a greeting. It appears to be a placeholder or an initial file and is not directly used by the chatbot or server logic.

```python
def main():
    print("Hello from mcp-project!")


if __name__ == "__main__":
    main()
```

### 6. `pyproject.toml` - Project Dependencies

Defines project metadata and dependencies. Key dependencies include:
*   `anthropic`: For interacting with Claude LLMs.
*   `arxiv`: For searching and retrieving paper information from arXiv.
*   `mcp`: The Model Context Protocol library.
*   `python-dotenv`: For loading environment variables (e.g., API keys) from a `.env` file.
*   `nest-asyncio`: To allow `asyncio` event loops to be nested, which is useful when running asyncio code in environments like Jupyter notebooks or certain interactive shells.

## Setup and Usage

### Prerequisites

*   Python 3.12 or higher (as specified in [`pyproject.toml`](/home/mopolaria/dev/mcp-project/mcp_project/pyproject.toml) and [`.python-version`](/home/mopolaria/dev/mcp-project/mcp_project/.python-version)).
*   `uv` (or `pip` and a virtual environment tool). `uv` is used in the `server_config.json` for running the `research_server.py`.
*   An Anthropic API key.

### Installation

1.  **Clone the repository (if applicable).**
2.  **Set up a virtual environment (recommended):**
    ```bash
    python -m venv .venv
    source .venv/bin/activate
    # or if using uv:
    uv venv
    source .venv/bin/activate
    ```
3.  **Install dependencies:**
    ```bash
    uv pip install -r requirements.txt # If you generate one
    # or directly using pyproject.toml with uv:
    uv pip install .
    ```
4.  **Create a `.env` file** in the `mcp_project` directory and add your Anthropic API key:
    ```env
    // filepath: /home/mopolaria/dev/mcp-project/mcp_project/.env
    ANTHROPIC_API_KEY="your_anthropic_api_key_here"
    ```
    *(Ensure `.env` is listed in your [`.gitignore`](/home/mopolaria/dev/mcp-project/mcp_project/.gitignore) file to avoid committing secrets.)*

### Running the Chatbot

1.  Ensure your virtual environment is activated.
2.  Navigate to the `mcp_project` directory.
3.  Run the chatbot:
    ```bash
    python mcp_chatbot.py
    ```
    This will:
    *   Start the MCP servers defined in [`server_config.json`](/home/mopolaria/dev/mcp-project/mcp_project/server_config.json) (specifically, it will try to run `uv run research_server.py`).
    *   Connect to these servers.
    *   Present you with a `Query:` prompt.

### Interacting with the Chatbot

Once the chatbot is running, you can interact with it using the following commands and natural language queries:

*   **Natural Language Queries**: Ask questions or give instructions. The chatbot will use the Anthropic LLM to understand and respond. If appropriate, the LLM might decide to use one of the available tools (like `search_papers` or `extract_info` from the research server).
    *   Example: `Search for papers on "quantum computing" and tell me about the first result.`
*   **View Available Topics/Folders**:
    ```
    @folders
    ```
    This will use the `get_available_folders` resource from the `research_server.py`.
*   **View Papers in a Specific Topic**:
    ```
    @<topic_name>
    ```
    Replace `<topic_name>` with a folder name listed by `@folders` (e.g., `@llm`, `@education`). This uses the `get_topic_papers` resource.
*   **List Available Prompts**:
    ```
    /prompts
    ```
    This will list prompts discovered from connected MCP servers (e.g., `generate_search_prompt`).
*   **Execute a Prompt**:
    ```
    /prompt <prompt_name> <arg1=value1> <arg2=value2> ...
    ```
    *   Example: `/prompt generate_search_prompt topic="machine learning" num_papers=3`
    This will fetch the prompt content using the arguments, and then process that content with the LLM.
*   **Quit**:
    ```
    quit
    ```

## Directory Structure

```
mcp-project/
├── mcp_project/
│   ├── .gitignore
│   ├── .python-version
│   ├── main.py                     # Basic entry point, not core to chatbot
│   ├── mcp_chatbot.py              # Main chatbot application
│   ├── mcp_architecture_diagram.txt # Text-based architecture diagram
│   ├── mcp_summary.md              # Summary of MCP
│   ├── papers/                     # Directory for storing paper info
│   │   ├── education/
│   │   │   └── papers_info.json
│   │   ├── large_language_models/
│   │   │   └── papers_info.json
│   │   └── llm/
│   │       └── papers_info.json
│   ├── pyproject.toml              # Project metadata and dependencies
│   ├── README.md                   # This file
│   ├── research_server.py          # MCP server for research tools
│   └── server_config.json          # Configuration for MCP servers
└── ... (other project files like .venv, etc.)
```

## Further Development

*   Implement the `filesystem` and `fetch` servers defined in [`server_config.json`](/home/mopolaria/dev/mcp-project/mcp_project/server_config.json) or remove them if not needed.
*   Add more tools, resources, and prompts to the `research_server.py` or create new specialized MCP servers.
*   Enhance error handling and user feedback in the chatbot.
*   Explore other MCP transport mechanisms (e.g., HTTP/SSE) if remote server access is required.
```// filepath: /home/mopolaria/dev/mcp-project/mcp_project/README.md
# MCP Project: Chatbot with Research Paper Tools

## Overview

This project implements a chatbot application that leverages the Model Context Protocol (MCP) to interact with various backend services. The primary example service included is a "research server" that allows users to search for academic papers on arXiv, store their information, and retrieve details about them. The chatbot uses an Anthropic language model (Claude) to understand user queries and can utilize tools provided by MCP servers to fulfill requests.

## What is the Model Context Protocol (MCP)?

The Model Context Protocol (MCP) is a standardized communication protocol designed to facilitate interaction between Large Language Model (LLM) applications (referred to as **Hosts** or **Clients**) and various backend services or tools (referred to as **Servers**). It enables LLMs to access and utilize external knowledge, execute actions, and integrate with other systems seamlessly.

MCP is built on a client-server architecture:

*   **Hosts**: These are the LLM applications (e.g., a desktop application, an IDE extension) that initiate connections to MCP servers.
*   **Clients**: Reside within the host application and maintain a 1:1 connection with an MCP server. They manage the communication flow.
*   **Servers**: These are processes that provide context, tools (callable functions), and prompts (pre-defined query templates) to clients.

### Core Architectural Components

1.  **Protocol Layer**:
    *   Manages message framing, ensuring that messages are correctly structured and understood by both client and server.
    *   Handles the linking of requests to their corresponding responses.
    *   Defines high-level communication patterns (e.g., request/response, notifications).

2.  **Transport Layer**:
    *   Responsible for the actual transmission of data between the client and server.
    *   MCP is transport-agnostic, meaning it can operate over different communication mechanisms. Common transports include:
        *   **Stdio Transport**: Uses standard input (stdin) and standard output (stdout) for communication. Ideal for local processes running on the same machine.
        *   **HTTP with Server-Sent Events (SSE) Transport**: Uses HTTP POST for client-to-server messages and SSE for server-to-client messages. Suitable for remote communication.
    *   All transports typically use JSON-RPC 2.0 for structuring the messages.

### Message Types

MCP defines several main types of messages, following the JSON-RPC 2.0 specification:

1.  **Requests**: Sent by one party (client or server) expecting a response from the other.
2.  **Results**: Successful responses to requests, containing the requested data.
3.  **Errors**: Indicate that a request failed, providing details about the failure.
4.  **Notifications**: One-way messages that do not expect a response.

### Connection Lifecycle

A typical MCP connection follows these stages:

1.  **Initialization**:
    *   The client sends an `initialize` request to the server, including its protocol version and capabilities.
    *   The server responds with its own protocol version and capabilities.
    *   The client sends an `initialized` notification to acknowledge successful initialization.
2.  **Message Exchange**:
    *   Once initialized, the client and server can exchange requests, results, errors, and notifications.
    *   This is where tools are called, resources are read, and prompts are executed.
3.  **Termination**:
    *   Either the client or the server can initiate the termination of the connection (e.g., via a `close()` command or due to transport disconnection).

## Project Components

### 1. `mcp_chatbot.py` - The Chatbot Application

This is the main entry point for the user interaction. It's an asynchronous application that:

*   **Connects to MCP Servers**: Reads server configurations from [`server_config.json`](/home/mopolaria/dev/mcp-project/mcp_project/server_config.json) and establishes connections to each defined server using stdio transport.
*   **Manages Sessions**: Keeps track of active MCP sessions for each server.
*   **Discovers Capabilities**: Upon connection, it lists available tools, prompts, and resources from each server.
    *   `available_tools`: Stores information about tools that the Anthropic LLM can use.
    *   `available_prompts`: Stores information about pre-defined prompts for user convenience.
    *   `sessions`: Maps tool names, prompt names, or resource URIs to their respective MCP client sessions.
*   **Interacts with Anthropic LLM**:
    *   Uses the `anthropic` Python SDK to send user queries and conversation history to a Claude model.
    *   Passes the `available_tools` to the LLM, enabling it to request tool calls.
*   **Handles Tool Calls**:
    *   When the LLM decides to use a tool, the chatbot receives a `tool_use` content block.
    *   It identifies the correct MCP session for the requested tool and calls `session.call_tool()` with the provided arguments.
    *   The result from the tool is then sent back to the LLM as a `tool_result` to continue the conversation.
*   **Provides Resource Access**:
    *   Allows users to fetch content from MCP resources using `@<resource_uri>` syntax (e.g., `@folders`, `@llm`).
    *   It uses `session.read_resource()` to get the content.
*   **Executes Prompts**:
    *   Allows users to list and execute pre-defined prompts using `/prompts` and `/prompt <name> <arg1=value1>` commands.
    *   It uses `session.get_prompt()` to retrieve the prompt content and then processes it with the LLM.
*   **Chat Loop**: Provides a command-line interface for users to interact with the chatbot.

### 2. `research_server.py` - Research MCP Server

This Python script implements an MCP server specifically for research-related tasks, primarily interacting with the arXiv API. It uses `FastMCP` from the `mcp.server` library for quick setup.

*   **Initialization**: `mcp = FastMCP("research")` creates an MCP server instance named "research".
*   **Tools**:
    *   `@mcp.tool() search_papers(topic: str, max_results: int = 5) -> List[str]`:
        *   Searches arXiv for papers matching the given `topic`.
        *   Stores paper metadata (title, authors, summary, PDF URL, published date) in a JSON file: `papers/<topic_sanitized>/papers_info.json`.
        *   Returns a list of paper IDs found.
    *   `@mcp.tool() extract_info(paper_id: str) -> str`:
        *   Searches through all `papers_info.json` files in subdirectories of the `papers` directory.
        *   Returns a JSON string with the paper's information if found, otherwise an error message.
*   **Resources**:
    *   `@mcp.resource("papers://folders") get_available_folders() -> str`:
        *   Lists all topic folders (subdirectories within the `papers` directory that contain a `papers_info.json` file).
        *   Returns a Markdown formatted list of available topics.
    *   `@mcp.resource("papers://{topic}") get_topic_papers(topic: str) -> str`:
        *   Retrieves and formats detailed information about papers for a specific `topic`.
        *   Reads the corresponding `papers/<topic_sanitized>/papers_info.json` file.
        *   Returns a Markdown formatted string with details for each paper (title, ID, authors, published date, PDF URL, summary).
*   **Prompts**:
    *   `@mcp.prompt() generate_search_prompt(topic: str, num_papers: int = 5) -> str`:
        *   Generates a detailed prompt for an LLM (like Claude) to guide it in searching for academic papers using the `search_papers` tool and then synthesizing the findings.
*   **Server Execution**:
    *   If run directly (`if __name__ == "__main__":`), it starts the MCP server using stdio transport: `mcp.run(transport='stdio')`.

### 3. `server_config.json` - Server Configuration

This JSON file defines the MCP servers that the [`mcp_chatbot.py`](/home/mopolaria/dev/mcp-project/mcp_project/mcp_chatbot.py) application should connect to.

```json
// filepath: /home/mopolaria/dev/mcp-project/mcp_project/server_config.json
{
    "mcpServers": {
        "filesystem": { // Example server (not fully implemented in this project's Python code)
            "command": "npx",
            "args": [
                "-y",
                "@modelcontextprotocol/server-filesystem",
                "."
            ]
        },
        "research": { // Configuration for the research_server.py
            "command": "uv", // Assumes 'uv' is used as a Python environment/runner
            "args": ["run", "research_server.py"]
        },
        "fetch": { // Example server (not fully implemented in this project's Python code)
            "command": "uvx",
            "args": ["mcp-server-fetch"]
        }
    }
}
```
The chatbot will attempt to launch and connect to each server listed under `mcpServers`. The `command` and `args` specify how to start each server process.

### 4. `papers/` Directory - Data Storage

This directory is used by the [`research_server.py`](/home/mopolaria/dev/mcp-project/mcp_project/research_server.py) to store information about academic papers.

*   When `search_papers` is called for a topic (e.g., "LLM"), a subdirectory is created (e.g., `papers/llm/`).
*   Inside this subdirectory, a `papers_info.json` file is created/updated. This file contains a dictionary where keys are paper IDs and values are objects with paper details.
    *   Example: [`papers/llm/papers_info.json`](/home/mopolaria/dev/mcp-project/mcp_project/papers/llm/papers_info.json)

### 5. `main.py`

A simple Python script that prints a greeting. It appears to be a placeholder or an initial file and is not directly used by the chatbot or server logic.

```python
// filepath: /home/mopolaria/dev/mcp-project/mcp_project/main.py
def main():
    print("Hello from mcp-project!")


if __name__ == "__main__":
    main()
```

### 6. `pyproject.toml` - Project Dependencies

Defines project metadata and dependencies. Key dependencies include:
*   `anthropic`: For interacting with Claude LLMs.
*   `arxiv`: For searching and retrieving paper information from arXiv.
*   `mcp`: The Model Context Protocol library.
*   `python-dotenv`: For loading environment variables (e.g., API keys) from a `.env` file.
*   `nest-asyncio`: To allow `asyncio` event loops to be nested, which is useful when running asyncio code in environments like Jupyter notebooks or certain interactive shells.

## Setup and Usage

### Prerequisites

*   Python 3.12 or higher (as specified in [`pyproject.toml`](/home/mopolaria/dev/mcp-project/mcp_project/pyproject.toml) and [`.python-version`](/home/mopolaria/dev/mcp-project/mcp_project/.python-version)).
*   `uv` (or `pip` and a virtual environment tool). `uv` is used in the `server_config.json` for running the `research_server.py`.
*   An Anthropic API key.

### Installation

1.  **Clone the repository (if applicable).**
2.  **Set up a virtual environment (recommended):**
    ```bash
    python -m venv .venv
    source .venv/bin/activate
    # or if using uv:
    uv venv
    source .venv/bin/activate
    ```
3.  **Install dependencies:**
    ```bash
    uv pip install -r requirements.txt # If you generate one
    # or directly using pyproject.toml with uv:
    uv pip install .
    ```
4.  **Create a `.env` file** in the `mcp_project` directory and add your Anthropic API key:
    ```env
    // filepath: /home/mopolaria/dev/mcp-project/mcp_project/.env
    ANTHROPIC_API_KEY="your_anthropic_api_key_here"
    ```
    *(Ensure `.env` is listed in your [`.gitignore`](/home/mopolaria/dev/mcp-project/mcp_project/.gitignore) file to avoid committing secrets.)*

### Running the Chatbot

1.  Ensure your virtual environment is activated.
2.  Navigate to the `mcp_project` directory.
3.  Run the chatbot:
    ```bash
    python mcp_chatbot.py
    ```
    This will:
    *   Start the MCP servers defined in [`server_config.json`](/home/mopolaria/dev/mcp-project/mcp_project/server_config.json) (specifically, it will try to run `uv run research_server.py`).
    *   Connect to these servers.
    *   Present you with a `Query:` prompt.

### Interacting with the Chatbot

Once the chatbot is running, you can interact with it using the following commands and natural language queries:

*   **Natural Language Queries**: Ask questions or give instructions. The chatbot will use the Anthropic LLM to understand and respond. If appropriate, the LLM might decide to use one of the available tools (like `search_papers` or `extract_info` from the research server).
    *   Example: `Search for papers on "quantum computing" and tell me about the first result.`
*   **View Available Topics/Folders**:
    ```
    @folders
    ```
    This will use the `get_available_folders` resource from the `research_server.py`.
*   **View Papers in a Specific Topic**:
    ```
    @<topic_name>
    ```
    Replace `<topic_name>` with a folder name listed by `@folders` (e.g., `@llm`, `@education`). This uses the `get_topic_papers` resource.
*   **List Available Prompts**:
    ```
    /prompts
    ```
    This will list prompts discovered from connected MCP servers (e.g., `generate_search_prompt`).
*   **Execute a Prompt**:
    ```
    /prompt <prompt_name> <arg1=value1> <arg2=value2> ...
    ```
    *   Example: `/prompt generate_search_prompt topic="machine learning" num_papers=3`
    This will fetch the prompt content using the arguments, and then process that content with the LLM.
*   **Quit**:
    ```
    quit
    ```

## Directory Structure

```
mcp-project/
├── mcp_project/
│   ├── .gitignore
│   ├── .python-version
│   ├── main.py                     # Basic entry point, not core to chatbot
│   ├── mcp_chatbot.py              # Main chatbot application
│   ├── mcp_architecture_diagram.txt # Text-based architecture diagram
│   ├── mcp_summary.md              # Summary of MCP
│   ├── papers/                     # Directory for storing paper info
│   │   ├── education/
│   │   │   └── papers_info.json
│   │   ├── large_language_models/
│   │   │   └── papers_info.json
│   │   └── llm/
│   │       └── papers_info.json
│   ├── pyproject.toml              # Project metadata and dependencies
│   ├── README.md                   # This file
│   ├── research_server.py          # MCP server for research tools
│   └── server_config.json          # Configuration for MCP servers
└── ... (other project files like .venv, etc.)
```

## Further Development

*   Implement the `filesystem` and `fetch` servers defined in [`server_config.json`](/home/mopolaria/dev/mcp-project/mcp_project/server_config.json) or remove them if not needed.
*   Add more tools, resources, and prompts to the `research_server.py` or create new specialized MCP servers.
*   Enhance error handling and user feedback in the chatbot.
*   Explore other MCP transport mechanisms (e.g., HTTP/SSE) if remote server access is required.