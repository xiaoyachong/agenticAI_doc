# LangGraph Multi-Agent Architecture Overview

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                 LANGGRAPH FRAMEWORK CORE                 │
├──────────────────┬─────────────────────┬────────────────┤
│  Workflow Engine │ Agent Interface     │ State Manager  │
├──────────────────┴─────────────────────┴────────────────┤
│                  HTTP API Layer                          │
└─────────┬─────────────────┬─────────────┬───────────────┘
          │                 │             │
 ┌────────▼────────┐ ┌─────▼───────┐ ┌───▼─────────┐
 │                 │ │             │ │             │
 │ Researcher      │ │ Planner     │ │ Executor    │
 │ Agent Repo      │ │ Agent Repo  │ │ Agent Repo  │
 │                 │ │             │ │             │
 └─────────────────┘ └─────────────┘ └─────────────┘
```

## Repository Structure

### Framework Core Repository

```
langgraph-framework/
├── src/
│   ├── framework/
│   │   ├── __init__.py
│   │   ├── workflow.py            # Graph definition and orchestration
│   │   ├── agent_interface.py     # Interface for remote agents
│   │   └── state_manager.py       # State management
│   ├── api/
│   │   ├── __init__.py
│   │   └── app.py                 # FastAPI application
│   └── utils/
│       ├── __init__.py
│       └── logging.py             # Logging utilities
├── docker/
│   └── docker-compose.yml
├── requirements.txt
└── README.md
```

### Agent Repository Template

```
agent-template/
├── src/
│   ├── agent.py                   # Agent implementation
│   ├── agent_server.py            # FastAPI server for the agent
│   └── utils.py                   # Utilities
├── Dockerfile
├── requirements.txt
└── README.md
```

## Core Components

### 1. Workflow Engine

The Workflow Engine is responsible for:
- Defining the graph structure of agent interactions
- Orchestrating the flow of state between agents
- Managing conditional branches in the workflow
- Handling agent responses and errors

The Workflow Engine uses LangGraph to create a directed graph where:
- Nodes are either local functions or remote agent calls
- Edges define the flow of state between nodes
- Conditions determine which paths to follow based on state

### 2. Agent Interface

The Agent Interface provides:
- A standardized way to communicate with remote agents
- HTTP client for making API calls to agent endpoints
- Error handling and retry logic for agent communication
- Timeout handling for unresponsive agents

### 3. State Manager

The State Manager is responsible for:
- Maintaining the workflow state throughout execution
- Merging agent responses into the current state
- Tracking the progress of the workflow
- Handling state persistence (optional)

## Deployment with Docker Compose

```yaml
version: '3'

services:
  framework:
    build:
      context: .
      dockerfile: docker/Dockerfile
    ports:
      - "8000:8000"
    environment:
      - LOG_LEVEL=INFO
    networks:
      - agent-network
    volumes:
      - ./config:/app/config

networks:
  agent-network:
    driver: bridge
```

## Adding a New Agent

To add a new agent to the system:

1. **Create a New Repository** using the agent template structure

2. **Implement the Agent Logic**:
   - Define the agent class that handles requests
   - Implement the invoke method to process state
   - Set up any necessary LLM, tools, or other components

3. **Create Docker Configuration**:
   - Define Dockerfile
   - Create docker-compose.yml that connects to the framework network
   - Define environment variables for framework connection

4. **Define Agent Registration**:
   - Implement startup logic to register with framework
   - Define agent name and endpoint URL

5. **Deploy the Agent**:
   - Build and start the agent container
   - Agent automatically registers with framework
   - Framework dynamically updates its workflow graph

Example Agent docker-compose.yml:
```yaml
version: '3'

services:
  my-agent:
    build: .
    environment:
      - FRAMEWORK_URL=http://framework:8000
      - AGENT_URL=http://my-agent:8000
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    networks:
      - langgraph-framework_agent-network

networks:
  langgraph-framework_agent-network:
    external: true
```

## Communication Flow

When the system runs, the communication flows as follows:

1. **Workflow Invocation**: Client sends request to framework API
2. **Workflow Execution**: Framework begins executing the graph
3. **Agent Invocation**: For agent nodes, framework calls the agent API
4. **State Passing**: Current workflow state is passed to the agent
5. **Agent Processing**: Agent processes the state and returns updated state
6. **State Update**: Framework updates workflow state with agent response
7. **Next Node**: Framework continues to next node in the graph
8. **Completion**: When graph reaches an end node, final state is returned

## Key Benefits

1. **Explicit Workflow**: Clear visualization of agent interactions
2. **Dynamic Agent Registration**: Agents can register at runtime
3. **State Management**: Consistent state handling throughout workflow
4. **Independent Development**: Separate repositories for each agent
5. **Scalability**: Can deploy multiple instances of framework or agents
6. **Flexibility**: Can add, remove, or update agents without changing core

## Implementation Considerations

### Agent Implementation Pattern

Each agent should follow this general pattern:

```python
class Agent:
    def __init__(self, name):
        self.name = name
        # Initialize any tools, LLMs, or other resources
    
    async def invoke(self, state):
        # Process the state
        # Update the state with new information
        return updated_state
```

### Framework Registration

Agents register with the framework at startup:

```python
def register_with_framework():
    registration_data = {
        "agent_name": AGENT_NAME,
        "agent_url": AGENT_URL
    }
    
    response = requests.post(
        f"{FRAMEWORK_URL}/register_agent",
        json=registration_data
    )
```

### Agent HTTP Interface

Each agent exposes an HTTP endpoint for the framework to call:

```python
@app.post("/invoke")
async def invoke_agent(request: Request):
    data = await request.json()
    state = data.get("state", {})
    result = await agent.invoke(state)
    return result
```

### Framework Workflow Configuration

The framework defines the workflow as a directed graph:

```python
def build_graph():
    graph = Graph()
    
    # Add nodes for each agent
    for agent_name in agent_registry:
        graph.add_node(agent_name, create_agent_node(agent_name))
    
    # Define workflow edges
    graph.add_edge("researcher", "planner")
    graph.add_edge("planner", "executor")
    
    return graph
```

This architecture provides a structured yet flexible approach to building multi-agent systems with LangGraph, allowing for clear workflow definition while maintaining the independence of each agent.
