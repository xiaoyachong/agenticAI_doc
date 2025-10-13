# Agent Registration and Request Processing

This document explains two critical processes in multi-agent architectures:
1. How agent registration happens automatically when the system starts
2. How a user request flows through the system in both architectures

## Agent Registration Process

### Message-Based Architecture

#### Registration Process

1. **Framework Startup**:
   - The framework services (Registry, Broker, API Gateway, Classifier) start
   - The Registry initializes empty capability and agent registries
   - The Message Broker starts listening for connections

2. **Agent Startup**:
   - Each agent container starts
   - In the agent's initialization code, it calls the registration logic

3. **Automatic Registration**:
   ```
   ┌──────────────┐                      ┌───────────────┐
   │              │                      │               │
   │ Agent        │                      │ Registry      │
   │              │                      │               │
   └──────┬───────┘                      └───────┬───────┘
          │                                      │
          │    POST /agents/register             │
          │ ────────────────────────────────►    │
          │  {                                   │
          │    "agent_id": "bolt_agent",         │
          │    "capabilities": [                 │
          │      {                               │
          │        "name": "move_motor",         │
          │        "description": "...",         │
          │        "parameters": {...}           │
          │      }                               │
          │    ]                                 │
          │  }                                   │
          │                                      │
          │    Response 200 OK                   │
          │ ◄────────────────────────────────    │
          │                                      │
   ```

#### Implementation

In the `bolt_agent` code, the registration happens in the initialization:

```python
def __init__(self):
    self.agent_id = "bolt_agent"
    self.registry_url = os.environ.get("REGISTRY_URL", "http://registry:8001")
    self.broker_url = os.environ.get("BROKER_URL", "amqp://rabbitmq:5672")
    
    # Register with registry
    self.register()
    
    # Connect to message broker
    self.connect_to_broker()

def register(self):
    """Register agent with the registry service"""
    capabilities = self.get_capabilities()
    
    registration_data = {
        "agent_id": self.agent_id,
        "capabilities": capabilities
    }
    
    response = requests.post(
        f"{self.registry_url}/agents/register", 
        json=registration_data
    )
    
    if response.status_code == 200:
        print(f"Agent {self.agent_id} registered successfully")
    else:
        print(f"Failed to register agent: {response.text}")
```

#### Retry and Health Checks

In a robust implementation, the agent would:
1. Retry registration if the registry service isn't available immediately
2. Implement health checks to re-register if disconnected
3. Use a startup delay to ensure the registry is ready before attempting registration

```python
def register_with_retry(self, max_retries=5, delay_seconds=5):
    """Register with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.post(...)
            if response.status_code == 200:
                return True
        except:
            pass
            
        print(f"Registration attempt {attempt+1} failed, retrying in {delay_seconds}s...")
        time.sleep(delay_seconds)
        
    return False
```

### LangGraph-Based Architecture

#### Registration Process

1. **Framework Startup**:
   - The LangGraph framework starts
   - It initializes an empty agent registry
   - It starts the HTTP API server

2. **Agent Startup**:
   - Each agent container starts
   - The agent initializes and waits briefly for the framework to be ready
   - The agent calls the framework's registration endpoint

3. **Automatic Registration**:
   ```
   ┌──────────────┐                      ┌───────────────┐
   │              │                      │               │
   │ Agent        │                      │ Framework     │
   │              │                      │               │
   └──────┬───────┘                      └───────┬───────┘
          │                                      │
          │    POST /register_agent              │
          │ ────────────────────────────────►    │
          │  {                                   │
          │    "agent_name": "bolt_agent",       │
          │    "agent_url": "http://bolt_agent:8000" │
          │  }                                   │
          │                                      │
          │    Response 200 OK                   │
          │ ◄────────────────────────────────    │
          │                                      │
   ```

4. **Graph Rebuilding**:
   - After registration, the framework rebuilds its workflow graph
   - It adds a node for the new agent
   - It connects the node to the workflow based on predefined patterns

#### Implementation

In the agent server code, the registration happens in a startup event:

```python
class AgentServer:
    def __init__(self, agent, framework_url=None):
        self.agent = agent
        self.framework_url = framework_url or os.environ.get("FRAMEWORK_URL")
        self.app = FastAPI()
        self.register_routes()
    
    async def startup(self):
        """Startup event handler"""
        # Give the framework time to start
        await asyncio.sleep(10)
        self.register_with_framework()
    
    def register_with_framework(self):
        """Register agent with the framework"""
        agent_url = os.environ.get("AGENT_URL", f"http://{self.agent.name}:8000")
        
        try:
            response = requests.post(
                f"{self.framework_url}/register_agent",
                json={
                    "agent_name": self.agent.name,
                    "agent_url": agent_url
                }
            )
            
            if response.status_code == 200:
                print(f"Agent {self.agent.name} registered successfully")
                return True
            else:
                print(f"Failed to register agent: {response.text}")
                return False
        except Exception as e:
            print(f"Error registering agent: {e}")
            return False
```

In the FastAPI application:

```python
# Add startup event
@app.on_event("startup")
async def on_startup():
    await server.startup()
```

#### Docker Configuration for Automatic Registration

For both architectures, the Docker Compose configuration ensures that the agents can find and connect to the framework:

##### Message-Based Architecture

```yaml
services:
  registry:
    # ... registry configuration ...

  bolt-agent:
    build: ./bolt-agent
    environment:
      - REGISTRY_URL=http://registry:8001  # Points to the registry service
      - BROKER_URL=amqp://rabbitmq:5672    # Points to the message broker
    depends_on:
      - registry
      - rabbitmq
```

##### LangGraph-Based Architecture

```yaml
services:
  framework:
    # ... framework configuration ...

  bolt-agent:
    build: ./bolt-agent
    environment:
      - FRAMEWORK_URL=http://framework:8000  # Points to the framework service
      - AGENT_URL=http://bolt-agent:8000     # URL where the agent can be reached
    depends_on:
      - framework
```

## Request Processing: "Move the Motor to 45 Degrees"

If a user types "move the motor to 45 degrees" and there's a `move_motor` capability defined in the `bolt_agent`, here's what would happen in each of the architectures:

### Message-Based Architecture

1. **Request Handling**:
   - User sends the text "move the motor to 45 degrees" to the API Gateway

2. **Classifier Processing**:
   - The API Gateway forwards the text to the Classifier service
   - The Classifier analyzes the text using an LLM
   - It identifies that the request is asking to move a motor to 45 degrees
   - It determines this matches the `move_motor` capability
   - It extracts the parameter value: angle = 45

3. **Capability Lookup**:
   - The API Gateway asks the Registry service which agent provides the `move_motor` capability
   - The Registry responds that `bolt_agent` provides this capability

4. **Message Routing**:
   - The API Gateway creates a message with:
     ```json
     {
       "capability": "move_motor",
       "params": {
         "angle": 45
       }
     }
     ```
   - It publishes this message to the Message Broker with routing key `agent.bolt_agent`
   - It sets up a response queue to receive the result

5. **Agent Processing**:
   - The `bolt_agent` receives the message from its queue
   - It recognizes the `move_motor` capability
   - It calls its internal function to move the motor to 45 degrees
   - It constructs a response with the results

6. **Response Handling**:
   - The `bolt_agent` sends the response back through the Message Broker to the response queue
   - The API Gateway receives the response and returns it to the user

### LangGraph-Based Architecture

1. **Request Handling**:
   - User sends the text "move the motor to 45 degrees" to the Framework API

2. **Workflow Initialization**:
   - The Framework API initializes a new workflow with this input
   - It creates an initial state containing the user's query

3. **Entry Node Processing**:
   - The workflow begins at the entry node
   - The entry node might identify this as a motor control request
   - It adds routing information to the state to direct it to the appropriate agent node

4. **Routing Node Processing**:
   - The router node examines the state and determines this should go to the `bolt_agent`

5. **Agent Node Processing**:
   - The framework calls the `bolt_agent` via HTTP POST to its `/invoke` endpoint
   - It sends the current workflow state containing the original query
   - The `bolt_agent` parses the query, identifies the intent to move the motor
   - It extracts the parameter (angle = 45)
   - It executes the motor movement operation
   - It updates the state with the operation results
   - It returns the updated state to the framework

6. **Workflow Continuation**:
   - The framework receives the updated state from the `bolt_agent`
   - It continues workflow processing (which might involve additional agent calls)
   - When the workflow completes, the final state is returned to the user

### Flow Diagrams

#### Message-Based Request Flow:

```
┌────────┐      ┌──────────┐      ┌──────────┐      ┌────────┐      ┌───────────┐
│        │      │          │      │          │      │        │      │           │
│ User   ├─────►│ API      ├─────►│ Classifier├─────►│Registry│      │ Bolt Agent│
│        │      │ Gateway  │      │          │      │        │      │           │
└────────┘      └────┬─────┘      └──────────┘      └────────┘      └─────┬─────┘
                     │                                                     │
                     │                                                     │
                     ▼                                                     │
                ┌────────┐           capability: move_motor                │
                │        │           params: {angle: 45}                   │
                │Message ├────────────────────────────────────────────────►│
                │Broker  │                                                 │
                │        │           result: {status: success}             │
                └───┬────┘◄────────────────────────────────────────────────┘
                    │
                    ▼
          ┌─────────────────┐
          │                 │
          │ Response to User│
          │                 │
          └─────────────────┘
```

#### LangGraph-Based Request Flow:

```
┌────────┐      ┌─────────────────────────────────────────┐      ┌───────────┐
│        │      │ Framework                               │      │           │
│ User   ├─────►│ ┌──────┐     ┌───────┐     ┌────────┐  │      │ Bolt Agent│
│        │      │ │Entry ├────►│Router ├────►│Agent   ├──┼─────►│           │
└────────┘      │ │Node  │     │Node   │     │Node    │  │      │           │
                │ └──────┘     └───────┘     └────────┘  │      └─────┬─────┘
                │                                        │            │
                └─────────────────────┬─────────────────┘            │
                                      │                               │
                                      │  state with results          │
                                      ◄───────────────────────────────┘
                                      │
                                      ▼
                            ┌─────────────────┐
                            │                 │
                            │ Response to User│
                            │                 │
                            └─────────────────┘
```

### Key Differences in How the Request is Handled

#### Message-Based Approach
- The classifier explicitly extracts the capability and parameters
- The registry explicitly maps the capability to an agent
- Communication is via message queues
- The agent receives a specific capability request with structured parameters
- The processing is capability-centric

#### LangGraph-Based Approach
- The workflow structure determines which agent handles the request
- The request might pass through multiple processing steps
- Communication is via direct HTTP calls
- The agent receives the entire workflow state and must parse the intent itself
- The processing is workflow-centric

In both cases, the end result is the motor being moved to 45 degrees, but the path to get there differs significantly between the two architectures.

## Summary

Both architectures provide automated registration of agents at startup, allowing for dynamic composition of the system without code changes. The key differences are:

1. **Registration Content**:
   - Message-Based: Agents register capabilities with descriptions and parameters
   - LangGraph: Agents register their endpoint URLs

2. **Request Flow**:
   - Message-Based: Capability-centered with explicit matching and routing
   - LangGraph: Workflow-centered with state passing between nodes

3. **Communication Style**:
   - Message-Based: Asynchronous messaging with explicit message formats
   - LangGraph: Synchronous HTTP calls passing state objects

These differences make each architecture better suited to different use cases, but both support the core requirement of having agents in separate repositories that can be added to the system without modifying the framework code.
