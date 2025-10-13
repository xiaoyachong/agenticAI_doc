# Multi-Agent Framework Architecture

This document outlines the architecture for a scalable multi-agent framework that allows for dynamic addition of new agents and capabilities without modifying the core framework code. Each agent exists in its own separate GitHub repository.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   FRAMEWORK CORE                        │
├────────────────┬───────────────┬────────────────────────┤
│ Registry       │ Classifier    │ Message                │
│ Service        │ Service       │ Broker                 │
├────────────────┴───────────────┴────────────────────────┤
│                 Communication Layer                     │
└─────────┬─────────────┬───────────────┬─────────────────┘
          │             │               │        
 ┌────────▼─────┐ ┌─────▼───────┐ ┌─────▼──────┐  
 │              │ │             │ │            │  
 │ Beamline     │ │ Weather     │ │ Your New   │  
 │ Agent Repo   │ │ Agent Repo  │ │ Agent Repo │  
 │              │ │             │ │            │  
 └──────────────┘ └─────────────┘ └────────────┘  
```

## Repository Structure

### Framework Core Repository

```
agent-framework/
├── src/
│   ├── framework/
│   │   ├── registry/               # Service registry 
│   │   ├── classifier/             # NL classifier for capabilities
│   │   ├── messaging/              # Message broker interface
│   │   └── base/                   # Base classes and interfaces
│   ├── api/                        # API gateway
│   │   ├── gateway.py              # Main API entry point
│   │   └── schemas.py              # API schemas
│   └── sdk/                        # Agent SDK (essential for agents)
│       └── base_agent.py           # Base agent class
├── docker/
│   └── docker-compose.yml          # Main docker-compose file
├── requirements.txt                # Python dependencies
└── README.md                       # Documentation
```

## Core Components

### 1. Service Registry

The Service Registry provides service discovery and registration for agents and their capabilities. It maintains information about:
- Available agents and their health status
- Capabilities provided by each agent
- Metadata about capabilities (parameters, schemas, etc.)

### 2. Attention Router

The Attention Router determines which agent and capability should handle a given request using:
- Embedding-based similarity matching
- Contextual relevance scoring
- Load and availability information
- Capability metadata matching

### 3. Message Broker

The Message Broker handles communication between agents and services:
- Asynchronous message passing
- Publish/subscribe patterns
- Request/response patterns
- Event streaming

### 4. Capability Store

The Capability Store maintains detailed information about capabilities:
- Capability schemas (parameters, return types)
- Capability metadata (description, examples)
- Versioning information
- Access control policies

## Deployment

### Docker Compose

```yaml
version: '3.8'

services:
  # Infrastructure
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  # Framework core
  registry:
    build:
      context: .
      dockerfile: ./deploy/docker/registry.Dockerfile
    ports:
      - "8001:8001"
    environment:
      - REDIS_URI=redis://redis:6379
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  router:
    build:
      context: .
      dockerfile: ./deploy/docker/router.Dockerfile
    ports:
      - "8002:8002"
    environment:
      - REGISTRY_URI=http://registry:8001
      - RABBITMQ_URI=amqp://rabbitmq:5672
    depends_on:
      registry:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  api-gateway:
    build:
      context: .
      dockerfile: ./deploy/docker/api-gateway.Dockerfile
    ports:
      - "8000:8000"
    environment:
      - REGISTRY_URI=http://registry:8001
      - ROUTER_URI=http://router:8002
      - RABBITMQ_URI=amqp://rabbitmq:5672
    depends_on:
      registry:
        condition: service_healthy
      router:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  # Agents
  weather-agent:
    build:
      context: ./agents/weather
      dockerfile: Dockerfile
    environment:
      - REGISTRY_URI=http://registry:8001
      - RABBITMQ_URI=amqp://rabbitmq:5672
    depends_on:
      registry:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    restart: unless-stopped

  beamline-agent:
    build:
      context: ./agents/beamline
      dockerfile: Dockerfile
    environment:
      - REGISTRY_URI=http://registry:8001
      - RABBITMQ_URI=amqp://rabbitmq:5672
    depends_on:
      registry:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    restart: unless-stopped

volumes:
  rabbitmq_data:
  redis_data:
```

### Kubernetes Deployment

For production deployments, Kubernetes manifests are provided in the `deploy/kubernetes` directory.

## Adding a New Agent

### 1. Create Agent Structure

Create a new directory for your agent:

```bash
mkdir -p agents/my-new-agent/src/capabilities
touch agents/my-new-agent/Dockerfile
touch agents/my-new-agent/requirements.txt
touch agents/my-new-agent/config.yaml
touch agents/my-new-agent/src/__init__.py
touch agents/my-new-agent/src/agent.py
touch agents/my-new-agent/src/capabilities/__init__.py
```

### 2. Implement Agent

Implement the agent using the SDK. Here's a simplified example structure:

**agent.py**:
```python
# Import the agent SDK
from framework_sdk import BaseAgent, Capability

# Define your agent class
class MyNewAgent(BaseAgent):
    def __init__(self, config_path=None):
        super().__init__("my-new-agent", config_path)
    
    def register_capabilities(self):
        # Register all capabilities with the framework
        return [
            {
                "name": "my_capability",
                "description": "Performs specific task",
                "parameters": {
                    "param1": {"type": "string", "description": "Parameter description"}
                },
                "returns": {
                    "type": "object",
                    "properties": {
                        "result": {"type": "string", "description": "Result description"}
                    }
                }
            }
        ]
    
    async def handle_capability(self, capability_name, params):
        # Route to appropriate handler based on capability name
        if capability_name == "my_capability":
            return await self.handle_my_capability(params)
        return {"error": "Capability not found"}
    
    async def handle_my_capability(self, params):
        # Implement capability logic
        # ...
        return {"result": "Processed result"}
```

### 3. Create Dockerfile

**Dockerfile**:
```Dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install the framework SDK
RUN pip install --no-cache-dir git+https://github.com/your-org/agent-framework.git#subdirectory=sdk/python

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "-m", "src.agent"]
```

### 4. Add to Docker Compose

Add your agent to the docker-compose.yml file:

```yaml
my-new-agent:
  build:
    context: ./agents/my-new-agent
    dockerfile: Dockerfile
  environment:
    - REGISTRY_URI=http://registry:8001
    - RABBITMQ_URI=amqp://rabbitmq:5672
  depends_on:
    registry:
      condition: service_healthy
    rabbitmq:
      condition: service_healthy
  restart: unless-stopped
```

### 5. Deploy the Agent

```bash
docker-compose up -d my-new-agent
```

The agent will automatically:
1. Register itself with the service registry
2. Register its capabilities
3. Connect to the message broker
4. Start processing requests for its capabilities

## Key Features

### 1. Attention-Based Routing

The Attention Router uses embeddings to match requests to the most appropriate capability, considering:
- Semantic similarity between request and capability descriptions
- Historical performance of capabilities for similar requests
- Current load and availability of agents

### 2. Event-Driven Architecture

The framework uses an event-driven architecture to enable:
- Asynchronous processing
- Loose coupling between components
- High scalability and resilience
- Real-time updates and notifications

### 3. Automatic Capability Discovery

The Service Registry automatically discovers and indexes capabilities:
- New agents register themselves on startup
- Capability metadata is automatically extracted
- API documentation is generated dynamically
- Health checks monitor capability availability

### 4. Versioning and Backward Compatibility

The framework supports capability versioning:
- Semantic versioning for capabilities
- Multiple versions of a capability can coexist
- Capability deprecation and migration paths
- Version negotiation between clients and agents

### 5. Observability and Monitoring

Comprehensive observability is built-in:
- Distributed tracing across agent boundaries
- Metrics for performance and health
- Structured logging
- Error tracking and alerting

## API Interfaces

The framework provides multiple API interfaces:

### 1. REST API

```
POST /api/v1/capabilities
{
  "capability": "weather_forecast",
  "parameters": {
    "location": "San Francisco",
    "days": 5
  }
}
```

### 2. gRPC Interface

```protobuf
service CapabilityService {
  rpc ExecuteCapability(CapabilityRequest) returns (CapabilityResponse);
}

message CapabilityRequest {
  string capability_name = 1;
  string parameters_json = 2;
}

message CapabilityResponse {
  string result_json = 1;
}
```

### 3. WebSocket Interface

```javascript
// Connect to WebSocket
const socket = new WebSocket('ws://api-gateway:8000/ws');

// Execute capability
socket.send(JSON.stringify({
  type: 'execute_capability',
  capability: 'weather_forecast',
  parameters: {
    location: 'San Francisco',
    days: 5
  }
}));

// Receive result
socket.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Result:', data.result);
};
```

### 4. Natural Language Interface

```
POST /api/v1/nl
{
  "query": "What's the weather forecast for San Francisco for the next 5 days?"
}
```

## Advanced Configuration

The framework supports advanced configuration for high-availability and scaling:

### 1. Horizontal Scaling

Multiple instances of agents can be deployed for horizontal scaling:
- Load balancing across agent instances
- Automatic failover
- State synchronization (when needed)

### 2. Geographic Distribution

Agents can be deployed across multiple regions:
- Latency-based routing
- Regional capability preferences
- Data residency compliance

### 3. Custom Middleware

Custom middleware can be added for:
- Authentication and authorization
- Rate limiting
- Request transformation
- Response caching
- Custom logging and metrics

## Conclusion

This multi-agent framework provides a robust, scalable architecture for building complex AI systems with multiple specialized agents. It combines modern architectural patterns with practical deployment considerations to create a system that is both powerful and maintainable.

By following the patterns and practices outlined in this document, you can build a framework that:
- Scales to handle large numbers of agents and requests
- Adapts to changing requirements with minimal disruption
- Provides comprehensive observability and monitoring
- Supports multiple interfaces for different client needs
- Enables rapid development and deployment of new agents

For more detailed implementation guidelines, refer to the documentation in the `docs/` directory.
