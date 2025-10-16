# Multi-Repo Agent Architecture Migration Guide

## Part 0: Architecture Overview
```
┌─────────────────────────────────────────────────────────────┐
│                     FRAMEWORK REPOSITORY                    │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │            LangGraph Orchestration Engine          │     │
│  │                                                    │     │
│  │  ┌──────────┐   ┌──────────────┐   ┌──────────┐    │     │
│  │  │Classifier│──►│Orchestrator  │──►│ Router   │    │     │
│  │  │  Node    │   │    Node      │   │  Node    │    │     │
│  │  └──────────┘   └──────────────┘   └────┬─────┘    │     │
│  │                                         │          │     │
│  └─────────────────────────────────────────┼──────────┘     │
│                                            │                │
│  ┌─────────────────────────────────────────┼──────────┐     │
│  │           Agent Registry Service        │          │     │
│  │                                         ▼          │     │
│  │  ┌──────────────────────────────────────────────┐  │     │
│  │  │ agent_id → AgentInfo(url, capabilities)      │  │     │
│  │  │ "BL5.3.1" → ["motor_control", ...]           │  │     │
│  │  │ "BL7.0.1" → ["sample_stage", ...]            │  │     │
│  │  └──────────────────────────────────────────────┘  │     │
│  └────────────────────────────────────────────────────┘     │
│                                                             │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │  HTTP API (Port 8100)
                         │  POST /register
                         │  GET  /agents/{agent_id}
                         │
    ┌────────────────────┼────────────────────────────┐
    │                    │                            │
┌───▼──────────┐   ┌─────▼────────┐   ┌───────────────▼ ──┐
│   AGENT 1    │   │   AGENT 2    │   │     AGENT N       │
│ BL5.3.1      │   │ BL7.0.1      │   │ Custom Agent      │
│ Capabilities:│   │ Capabilities:│   │ Capabilities:     │
│ - motor_ctrl │   │ - sample_move│   │ - data_analysis   │
│ - beam_align │   │ - temp_ctrl  │   │ - visualization   │
│ Port: 8053   │   │ Port: 8071   │   │ Port: 8090        │
└──────────────┘   └──────────────┘   └───────────────────┘
     ▲                    ▲                     ▲
     │                    │                     │
     └────────────────────┴─────────────────────┘
              Docker Network: agent-network

```
## Part 1: Framework Repository Structure

```
alpha-berkeley-framework/
│
├── src/
│   ├── framework/
│   │   ├── __init__.py                           # No change
│   │   ├── base/                                 # No change - Base classes
│   │   │   ├── __init__.py
│   │   │   ├── capability.py
│   │   │   ├── node.py
│   │   │   └── ...
│   │   ├── capabilities/                         # No change - Framework capabilities
│   │   │   ├── __init__.py
│   │   │   ├── memory.py
│   │   │   ├── time_parser.py
│   │   │   └── ...
│   │   ├── context/                              # No change - Context management
│   │   │   ├── __init__.py
│   │   │   ├── manager.py
│   │   │   └── ...
│   │   ├── data_sources/                         # No change - Data sources
│   │   │   ├── __init__.py
│   │   │   └── ...
│   │   ├── graph/                                # No change - LangGraph orchestration
│   │   │   ├── __init__.py
│   │   │   ├── builder.py
│   │   │   └── ...
│   │   ├── infrastructure/                       # MODIFIED - Added agent support
│   │   │   ├── __init__.py                       # No change
│   │   │   ├── gateway.py                        # No change
│   │   │   ├── agent_registry.py                 # NEW - Remote agent registry
│   │   │   ├── agent_registry_server.py          # NEW - FastAPI server for registry
│   │   │   └── agent_client.py                   # NEW - HTTP client for agents
│   │   ├── nodes/                                # No change - Core nodes
│   │   │   ├── __init__.py
│   │   │   ├── router.py
│   │   │   ├── orchestrator.py
│   │   │   └── ...
│   │   ├── prompts/                              # No change - Prompt templates
│   │   │   ├── __init__.py
│   │   │   └── ...
│   │   ├── registry/                             # No change - Registry system
│   │   │   ├── __init__.py
│   │   │   ├── manager.py
│   │   │   ├── base.py
│   │   │   └── registry.py
│   │   ├── services/                             # No change - Internal services
│   │   │   ├── __init__.py
│   │   │   └── python_executor/                  # No change
│   │   │       ├── __init__.py
│   │   │       └── ...
│   │   └── config.yml                            # No change - Framework config
│   │
│   ├── applications/                             # REMOVED - Move to separate repos
│   │
│   └── configs/                                  # No change - Global config utilities
│       ├── __init__.py                           # No change
│       ├── config.py                             # No change
│       └── logger.py                             # No change
│
├── interfaces/                                   # No change - UI interfaces
│   ├── openwebui/                                # No change
│   └── ...
│
├── services/                                     # No change - Optional services
│   └── framework/
│       └── jupyter/                              # No change - Jupyter service
│
├── tests/                                        # No change
├── docs/                                         # No change
│   └── agent-development-guide.md                # NEW - Guide for building agents
│
├── docker-compose.yml                            # NEW - At root (no deployment/ folder)
├── Dockerfile                                    # NEW - At root (no deployment/ folder)
├── Dockerfile.registry                           # NEW - For registry service
├── .gitignore                                    # No change
├── .env.example                                  # NEW - At root (ONLY ONE)
├── config.yml                                    # MODIFIED - Remove applications list
├── pyproject.toml                                # MODIFIED - Add framework dependencies
└── README.md                                     # MODIFIED - Update with new architecture
```

## Part 2: Agent Repository Structure (Example: beamline-531-agent)

```
beamline-531-agent/
│
├── src/
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── server.py                             # NEW - FastAPI HTTP server (including auto-registration of capabilities when start up)
│   │   ├── capabilities.py                       # Moved from framework
│   │   ├── context.py                            # Moved from framework (if agent-specific)
│   │   └── handlers.py                           # NEW - handle capability call requested from framework
│   │
│   └── lib/                                      # Agent-specific libraries
│       ├── __init__.py
│       ├── motor_control.py                      # Motor control logic
│       ├── beam_alignment.py                     # Beam alignment logic
│       └── shutter_control.py                    # Shutter control logic
│
├── tests/
│   ├── __init__.py
│   └── test_agent.py
│
├── docker-compose.yml                            # NEW - Agent deployment
├── Dockerfile                                    # NEW - Agent image
├── .env.example                                  # NEW - Agent env vars
├── config.yml                                    # NEW - Agent config
├── pyproject.toml                                # NEW - Agent dependencies
├── .gitignore
└── README.md                                     # NEW - Agent documentation
```

## Ensures Scoped Capability Lookup

**Problem**: 10 Agents × 10 Capabilities = 100 Total

When a user with Agent BL5.3.1 asks "Move motor M1 to 45 degrees", we want to:
- ✅ Check only Agent BL5.3.1's 10 capabilities
- ❌ NOT check all 100 capabilities across all agents

**Solution: Direct Agent ID-to-Agent Mapping**

Registry Structure:
```python
{
  "BL1": AgentInfo(url="...", capabilities=[10 BL1 capabilities]),
  "BL5.3.1": AgentInfo(url="...", capabilities=[10 BL5.3.1 capabilities]),
  "BL7.0.1": AgentInfo(url="...", capabilities=[10 BL7.0.1 capabilities]),
  ...
}
```

Lookup Process:
```python
# User with Agent BL5.3.1
agent_id = "BL5.3.1"

# O(1) lookup - get only this agent
agent = registry.get_agent_by_id(agent_id)

# Get capabilities - only 10!
capabilities = agent.capabilities  

# LLM classification sees only these 10, never the other 90!
```

**Result**: Framework only checks 10 relevant capabilities! ✅

LangGraph Usage (with agent context):
```python
# Framework classifier node - only checks agent's capabilities
async def classifier_node(state: AgentState) -> AgentState:
    """Classify intent - only look at current agent's capabilities"""
    
    # Get agent context from state
    agent_id = state.get("agent_id")  # e.g., "BL5.3.1"
    
    # Get agent by ID (O(1) lookup)
    registry = get_agent_registry()
    agent = registry.get_agent_by_id(agent_id)
    
    if not agent:
        return {**state, "error": f"No agent found with ID: {agent_id}"}
    
    # Get capabilities - only 10 for this agent!
    capabilities = agent.capabilities
    
    # Classify with LLM (sees only 10 capabilities, not 100!)
    prompt = f"""
    You are helping a user with Agent {agent_id}.
    
    Available capabilities: {', '.join(capabilities)}
    
    User request: {state['input']}
    
    Which capability should be used?
    """
    
    llm_response = await llm.ainvoke(prompt)
    
    return {
        **state,
        "agent_url": agent.url,
        "capability": llm_response.capability,
        "parameters": llm_response.parameters
    }
```

## Part 3: New Code Implementation

### Framework: agent_registry.py

```python
"""Agent Registry Service - Track and discover remote agents."""

import asyncio
import logging
from typing import Dict, List, Optional
from datetime import datetime
from pydantic import BaseModel

logger = logging.getLogger(__name__)


class AgentInfo(BaseModel):
    """Information about a registered agent."""
    name: str
    url: str
    agent_id: str  # Unique agent identifier (e.g., "BL5.3.1")
    capabilities: List[str]
    status: str = "active"
    registered_at: datetime
    last_heartbeat: Optional[datetime] = None


class AgentRegistry:
    """Registry for tracking remote agents and their capabilities.
    
    Uses direct agent_id-to-agent mapping for O(1) lookups.
    This ensures that when querying capabilities for a specific agent,
    only that agent's capabilities are checked (e.g., 10 instead of 100).
    """
    
    def __init__(self):
        # PRIMARY INDEX: Direct agent_id → agent mapping for O(1) lookup
        self._agents_by_id: Dict[str, AgentInfo] = {}
        
        # SECONDARY INDEX: Capability → agent names (for cross-agent queries)
        self._capability_map: Dict[str, List[str]] = {}
    
    def register_agent(
        self,
        name: str,
        url: str,
        agent_id: str,  # REQUIRED - unique agent identifier
        capabilities: List[str]
    ) -> None:
        """Register a new agent with its capabilities.
        
        Args:
            name: Agent name
            url: Agent URL
            agent_id: Unique agent identifier (e.g., "BL5.3.1", "BL1", "agent_foo")
            capabilities: List of capability names
        """
        agent_info = AgentInfo(
            name=name,
            url=url,
            agent_id=agent_id,
            capabilities=capabilities,
            registered_at=datetime.now(),
            last_heartbeat=datetime.now()
        )
        
        # Store by agent_id for O(1) lookup
        self._agents_by_id[agent_id] = agent_info
        
        # Update capability map (for cross-agent capability search)
        for capability in capabilities:
            if capability not in self._capability_map:
                self._capability_map[capability] = []
            if name not in self._capability_map[capability]:
                self._capability_map[capability].append(name)
        
        logger.info(
            f"Registered agent '{name}' (ID: {agent_id}) "
            f"with {len(capabilities)} capabilities"
        )
    
    def unregister_agent(self, name: str) -> None:
        """Remove an agent from the registry."""
        # Find agent by name (need to search through agents_by_id)
        agent_info = None
        agent_id_to_remove = None
        
        for ag_id, agent in self._agents_by_id.items():
            if agent.name == name:
                agent_info = agent
                agent_id_to_remove = ag_id
                break
        
        if agent_info and agent_id_to_remove:
            # Remove from capability map
            for capability in agent_info.capabilities:
                if capability in self._capability_map:
                    self._capability_map[capability].remove(name)
                    if not self._capability_map[capability]:
                        del self._capability_map[capability]
            
            # Remove from agents
            del self._agents_by_id[agent_id_to_remove]
            logger.info(f"Unregistered agent '{name}' (ID: {agent_id_to_remove})")
    
    def get_agent(self, name: str) -> Optional[AgentInfo]:
        """Get information about a specific agent by name."""
        for agent in self._agents_by_id.values():
            if agent.name == name:
                return agent
        return None
    
    def get_agent_by_id(self, agent_id: str) -> Optional[AgentInfo]:
        """Get agent by its unique ID - O(1) lookup.
        
        This is the PRIMARY method for agent-scoped queries.
        
        Args:
            agent_id: Unique agent identifier (e.g., "BL5.3.1")
            
        Returns:
            Agent info for this ID, or None if not found
        """
        return self._agents_by_id.get(agent_id)
    
    def get_agent_capabilities(self, agent_id: str) -> List[str]:
        """Get all capabilities for a specific agent.
        
        This ensures only the agent's capabilities are returned
        (e.g., 10 capabilities for one agent, not all 100).
        
        Args:
            agent_id: Unique agent identifier
            
        Returns:
            List of capability names for this agent
        """
        agent = self._agents_by_id.get(agent_id)
        return agent.capabilities if agent else []
    
    def get_agents_for_capability(self, capability: str) -> List[AgentInfo]:
        """Get all agents that provide a specific capability.
        
        This is for cross-agent queries (less common).
        """
        agent_names = self._capability_map.get(capability, [])
        return [
            agent for agent in self._agents_by_id.values() 
            if agent.name in agent_names
        ]
    
    def list_all_agents(self) -> List[AgentInfo]:
        """List all registered agents."""
        return list(self._agents_by_id.values())
    
    def update_heartbeat(self, name: str) -> None:
        """Update the last heartbeat time for an agent."""
        agent = self.get_agent(name)
        if agent:
            agent.last_heartbeat = datetime.now()


# Global registry instance
_registry = AgentRegistry()


def get_agent_registry() -> AgentRegistry:
    """Get the global agent registry instance."""
    return _registry
```

### Framework: agent_registry_server.py

```python
"""Agent Registry HTTP Server."""

from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List
from datetime import datetime
import uvicorn

from framework.infrastructure.agent_registry import get_agent_registry

app = FastAPI(title="Agent Registry Service")

class AgentRegistration(BaseModel):
    name: str
    url: str
    agent_id: str
    capabilities: List[str]

@app.post("/register")
async def register_agent(registration: AgentRegistration):
    """Register a new agent with the framework."""
    registry = get_agent_registry()
    
    try:
        registry.register_agent(
            name=registration.name,
            url=registration.url,
            agent_id=registration.agent_id,
            capabilities=registration.capabilities
        )
        return {
            "status": "registered",
            "agent_id": registration.agent_id,
            "message": f"Agent {registration.name} registered successfully"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.delete("/unregister/{agent_name}")
async def unregister_agent(agent_name: str):
    """Unregister an agent."""
    registry = get_agent_registry()
    registry.unregister_agent(agent_name)
    return {"status": "unregistered", "agent": agent_name}

@app.get("/agents")
async def list_agents():
    """List all registered agents."""
    registry = get_agent_registry()
    agents = registry.list_all_agents()
    return {"agents": [agent.dict() for agent in agents]}

@app.get("/agents/{agent_id}")
async def get_agent(agent_id: str):
    """Get a specific agent by ID."""
    registry = get_agent_registry()
    agent = registry.get_agent_by_id(agent_id)
    if not agent:
        raise HTTPException(status_code=404, detail=f"Agent {agent_id} not found")
    return agent.dict()

@app.post("/agents/{agent_name}/heartbeat")
async def agent_heartbeat(agent_name: str):
    """Update heartbeat for an agent."""
    registry = get_agent_registry()
    registry.update_heartbeat(agent_name)
    return {"status": "ok", "timestamp": datetime.now().isoformat()}

@app.get("/health")
async def health_check():
    """Health check endpoint."""
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8100)
```

### Framework: agent_client.py

```python
"""HTTP Client for Remote Agent Communication."""

import httpx
import logging
from typing import Any, Dict, Optional

logger = logging.getLogger(__name__)


class AgentClient:
    """HTTP client for communicating with remote agents."""
    
    def __init__(self, timeout: int = 30):
        self.timeout = timeout
        self._client = httpx.AsyncClient(timeout=timeout)
    
    async def call_capability(
        self,
        agent_url: str,
        capability_name: str,
        parameters: Dict[str, Any],
        context: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """
        Call a capability on a remote agent via HTTP.
        
        Args:
            agent_url: Base URL of the agent
            capability_name: Name of the capability to invoke
            parameters: Parameters for the capability
            context: Optional context data
            
        Returns:
            Response from the agent
            
        Raises:
            httpx.HTTPError: If the request fails
        """
        endpoint = f"{agent_url}/capability/{capability_name}"
        
        payload = {
            "parameters": parameters,
            "context": context or {}
        }
        
        logger.info(f"Calling agent capability: {endpoint}")
        
        try:
            response = await self._client.post(endpoint, json=payload)
            response.raise_for_status()
            return response.json()
        except httpx.HTTPError as e:
            logger.error(f"Agent call failed: {e}")
            raise
    
    async def health_check(self, agent_url: str) -> bool:
        """Check if an agent is healthy and responsive."""
        try:
            response = await self._client.get(f"{agent_url}/health")
            return response.status_code == 200
        except Exception as e:
            logger.warning(f"Health check failed for {agent_url}: {e}")
            return False
    
    async def close(self):
        """Close the HTTP client."""
        await self._client.aclose()
```

### Agent: server.py (FastAPI HTTP Server)

```python
"""Agent Server with Auto-Registration."""

import os
import logging
from fastapi import FastAPI
from contextlib import asynccontextmanager
import httpx

from agent.capabilities import get_capability_registry
from agent.handlers import capability_router

logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Manage agent lifecycle - register on startup, cleanup on shutdown."""
    # Startup
    await register_with_framework()
    yield
    # Shutdown (optional cleanup)
    await unregister_from_framework()


app = FastAPI(
    title="Agent Server",
    lifespan=lifespan  # Clean lifecycle management
)

# Include capability routes
app.include_router(capability_router)


async def register_with_framework():
    """Register this agent with the framework."""
    # Skip if not configured
    if not os.getenv("FRAMEWORK_REGISTRY_URL"):
        logger.info("No FRAMEWORK_REGISTRY_URL configured, skipping registration")
        return
    
    # Get configuration from environment
    config = {
        "name": os.getenv("AGENT_NAME", "unnamed_agent"),
        "url": f"http://{os.getenv('HOSTNAME', 'localhost')}:{os.getenv('AGENT_PORT', '8000')}",
        "agent_id": os.getenv("AGENT_ID"),  # Required for scoped lookup
        "capabilities": get_capability_registry().list_capability_names()
    }
    
    # Validate required fields
    if not config["agent_id"]:
        raise ValueError("AGENT_ID environment variable is required")
    
    # Register
    async with httpx.AsyncClient(timeout=10.0) as client:
        try:
            response = await client.post(
                f"{os.getenv('FRAMEWORK_REGISTRY_URL')}/register",
                json=config
            )
            response.raise_for_status()
            logger.info(f"✅ Registered agent '{config['name']}' (ID: {config['agent_id']})")
        except Exception as e:
            logger.error(f"❌ Registration failed: {e}")
            # Decide: fail fast or continue without registration
            if os.getenv("REQUIRE_REGISTRATION", "false").lower() == "true":
                raise


async def unregister_from_framework():
    """Unregister from framework on shutdown (optional)."""
    if not os.getenv("FRAMEWORK_REGISTRY_URL"):
        return
    
    agent_name = os.getenv("AGENT_NAME")
    async with httpx.AsyncClient(timeout=5.0) as client:
        try:
            await client.delete(
                f"{os.getenv('FRAMEWORK_REGISTRY_URL')}/unregister/{agent_name}"
            )
            logger.info(f"Unregistered agent '{agent_name}'")
        except Exception:
            pass  # Best effort on shutdown


@app.get("/health")
async def health_check():
    """Simple health check endpoint."""
    return {"status": "healthy"}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        app, 
        host="0.0.0.0", 
        port=int(os.getenv("AGENT_PORT", "8000"))
    )
```

### Agent: capabilities.py

```python
"""Agent Capabilities Registry - Self-contained and simple."""

from typing import List, Dict, Any
from abc import ABC, abstractmethod


class Capability(ABC):
    """Base capability class - simple and clean."""
    
    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description
    
    @abstractmethod
    async def execute(self, parameters: Dict[str, Any]) -> Dict[str, Any]:
        """Execute the capability."""
        pass


class MotorControlCapability(Capability):
    """Control motors on the beamline."""
    
    def __init__(self):
        super().__init__(
            name="motor_control",
            description="Control beamline motors"
        )
    
    async def execute(self, parameters: Dict[str, Any]) -> Dict[str, Any]:
        motor = parameters.get("motor_name")
        position = parameters.get("position")
        
        # Your actual motor control logic here
        # await self.move_motor(motor, position)
        
        return {
            "success": True,
            "motor": motor,
            "position": position
        }


class CapabilityRegistry:
    """Simple registry for agent capabilities."""
    
    def __init__(self):
        self.capabilities = {}
        self._register_capabilities()
    
    def _register_capabilities(self):
        """Register all capabilities for this agent."""
        # Add all capabilities here
        capabilities = [
            MotorControlCapability(),
            # BeamAlignmentCapability(),
            # ShutterControlCapability(),
        ]
        
        for cap in capabilities:
            self.capabilities[cap.name] = cap
    
    def get(self, name: str) -> Capability:
        """Get capability by name."""
        return self.capabilities.get(name)
    
    def list_capability_names(self) -> List[str]:
        """Get list of capability names for registration."""
        return list(self.capabilities.keys())


# Global instance
_registry = CapabilityRegistry()

def get_capability_registry() -> CapabilityRegistry:
    return _registry
```

### Agent: handlers.py

```python
"""API route handlers for capabilities."""

from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from typing import Dict, Any

from agent.capabilities import get_capability_registry

router = APIRouter(prefix="/capability", tags=["capabilities"])


class CapabilityRequest(BaseModel):
    parameters: Dict[str, Any]
    context: Dict[str, Any] = {}


@router.post("/{capability_name}")
async def execute_capability(capability_name: str, request: CapabilityRequest):
    """Execute a capability."""
    registry = get_capability_registry()
    capability = registry.get(capability_name)
    
    if not capability:
        raise HTTPException(
            status_code=404,
            detail=f"Capability '{capability_name}' not found"
        )
    
    try:
        result = await capability.execute(request.parameters)
        return {"success": True, "result": result}
    except Exception as e:
        return {"success": False, "error": str(e)}


@router.get("/")
async def list_capabilities():
    """List available capabilities."""
    registry = get_capability_registry()
    return {
        "capabilities": [
            {"name": name, "description": cap.description}
            for name, cap in registry.capabilities.items()
        ]
    }


# Export router for inclusion in main app
capability_router = router
```

## Part 4: Docker Configuration Files

### Framework: docker-compose.yml (AT ROOT)

```yaml
version: '3.8'

name: alpha-framework

services:
  # LangGraph API Server
  framework-api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: framework-api
    ports:
      - "${FRAMEWORK_PORT:-8000}:8000"
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DB=${POSTGRES_DB:-framework}
      - POSTGRES_USER=${POSTGRES_USER:-framework}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - GEMINI_API_KEY=${GEMINI_API_KEY}
      - LANGFUSE_SECRET_KEY=${LANGFUSE_SECRET_KEY}
      - LANGFUSE_PUBLIC_KEY=${LANGFUSE_PUBLIC_KEY}
      - LANGFUSE_HOST=${LANGFUSE_HOST}
      - PYTHONPATH=/app/src
    volumes:
      - ./src:/app/src:ro
      - ./config.yml:/app/config.yml:ro
      - agent-data:/app/_agent_data
    networks:
      - agent-network
    depends_on:
      - postgres
    restart: unless-stopped
    profiles: ["core"]

  # PostgreSQL for LangGraph Checkpointing
  postgres:
    image: postgres:15
    container_name: framework-postgres
    environment:
      - POSTGRES_DB=${POSTGRES_DB:-framework}
      - POSTGRES_USER=${POSTGRES_USER:-framework}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    networks:
      - agent-network
    restart: unless-stopped
    profiles: ["core"]

  # Agent Registry Service (Optional)
  agent-registry:
    build:
      context: .
      dockerfile: Dockerfile.registry
    container_name: agent-registry
    ports:
      - "${REGISTRY_PORT:-8100}:8100"
    environment:
      - REGISTRY_PORT=8100
    volumes:
      - registry-data:/data
    networks:
      - agent-network
    restart: unless-stopped
    profiles: ["registry"]

  # OpenWebUI Interface (Optional)
  openwebui:
    build:
      context: ./interfaces/openwebui
      dockerfile: Dockerfile
    container_name: framework-openwebui
    ports:
      - "${OPENWEBUI_PORT:-3000}:8080"
    environment:
      - FRAMEWORK_API_URL=http://framework-api:8000
    networks:
      - agent-network
    depends_on:
      - framework-api
    restart: unless-stopped
    profiles: ["ui"]

  # Jupyter Notebook (Development)
  jupyter:
    build:
      context: ./services/framework/jupyter
      dockerfile: Dockerfile
    container_name: framework-jupyter
    ports:
      - "${JUPYTER_PORT:-8888}:8888"
    environment:
      - JUPYTER_ENABLE_LAB=yes
      - PYTHONPATH=/jupyter/repo_src
    volumes:
      - ./src:/jupyter/repo_src:ro
      - jupyter-data:/home/jovyan/work
    networks:
      - agent-network
    restart: unless-stopped
    profiles: ["dev"]

networks:
  agent-network:
    name: agent-network
    driver: bridge

volumes:
  postgres-data:
  registry-data:
  agent-data:
  jupyter-data:
```

### Framework: Dockerfile (AT ROOT)

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy pyproject.toml and install dependencies
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Copy source code
COPY src/ /app/src/
COPY config.yml /app/

# Set Python path
ENV PYTHONPATH=/app/src

# Expose port
EXPOSE 8000

# Run the framework
CMD ["python", "-m", "uvicorn", "framework.api:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Framework: Dockerfile.registry (AT ROOT)

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install minimal dependencies for registry service
RUN pip install --no-cache-dir fastapi uvicorn httpx pydantic

# Copy registry code
COPY src/framework/infrastructure/agent_registry.py /app/
COPY src/framework/infrastructure/agent_registry_server.py /app/

# Run registry server
CMD ["python", "agent_registry_server.py"]
```

### Framework: .env.example (AT ROOT - ONLY ONE)

```bash
# PostgreSQL Configuration
POSTGRES_DB=framework
POSTGRES_USER=framework
POSTGRES_PASSWORD=your_secure_password_here

# Service Ports
FRAMEWORK_PORT=8000
POSTGRES_PORT=5432
REGISTRY_PORT=8100
OPENWEBUI_PORT=3000
JUPYTER_PORT=8888

# API Keys
OPENAI_API_KEY=your_openai_key_here
GEMINI_API_KEY=your_gemini_key_here

# Langfuse (Optional)
LANGFUSE_SECRET_KEY=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com
LANGFUSE_ENABLED=false

# Project Configuration
PROJECT_ROOT=/app
TZ=America/Los_Angeles
```

### Framework: pyproject.toml

```toml
[build-system]
requires = ["setuptools>=65.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "alpha-berkeley-framework"
version = "0.5.0"
description = "An open-source, domain-agnostic, capability-based architecture for building intelligent agents"
readme = "README.md"
requires-python = ">=3.11"
license = {text = "MIT"}
authors = [
    {name = "Thorsten Hellert", email = "thellert@lbl.gov"},
]

keywords = [
    "ai", "agents", "framework", "scientific-computing", "langgraph", 
    "epics", "accelerator-physics", "als", "berkeley", "agent-framework",
    "capability-based", "human-in-the-loop", "container-orchestration"
]

# Core runtime dependencies
dependencies = [
    # LangGraph dependencies - CRITICAL VERSION REQUIREMENTS
    "langgraph>=0.5.2",
    "langchain-core>=0.3.68",
    "langgraph-checkpoint-postgres>=2.0.22,<3.0.0",
    "langgraph-sdk>=0.1.70,<0.2.0",
    "psycopg[pool]>=3.1.0,<4.0.0",
    "psycopg-pool>=3.1.0,<4.0.0",
    "langchain-postgres>=0.0.12,<0.1.0",
    
    # Core framework dependencies
    "rich>=14.0.0",
    "pydantic-ai>=0.2.11",
    "python-dotenv>=1.1.0",
    "PyYAML>=6.0.2",
    "Jinja2>=3.1.6",
    "requests>=2.32.3",
    
    # API server
    "fastapi>=0.104.0",
    "uvicorn>=0.24.0",
    
    # HTTP client for agents
    "httpx>=0.25.0",
    
    # Additional dependencies from your project
    "langchain-openai>=0.3.0",
    "langchain-google-genai>=2.0.11",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "black>=23.0.0",
    "ruff>=0.1.0",
    "ipython>=8.12.0",
]

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]
```

### Framework: config.yml

```yaml
# Import framework configuration
import: src/framework/config.yml

build_dir: ./build

# Root directory (absolute path)
project_root: ${PROJECT_ROOT}

# Applications - REMOVED (agents now in separate repos)
# applications:
#   - als_assistant

# ============================================================
# OPERATOR CONTROLS
# ============================================================

approval:
  global_mode: "selective"
  capabilities:
    python_execution:
      enabled: true
      mode: "epics_writes"
    memory: 
      enabled: true

execution_control:
  epics:
    writes_enabled: false
  limits:
    max_reclassifications: 1
    max_planning_attempts: 2
    max_step_retries: 3
    max_execution_time_seconds: 3000
    graph_recursion_limit: 100

system:
  timezone: ${TZ:-America/Los_Angeles}

file_paths:
  agent_data_dir: _agent_data
  executed_python_scripts_dir: executed_scripts
  execution_plans_dir: execution_plans
  user_memory_dir: user_memory
  registry_exports_dir: registry_exports
  prompts_dir: prompts
  checkpoints: checkpoints
```

## Agent Files

### Agent: docker-compose.yml

```yaml
version: '3.8'

name: beamline-531-agent

services:
  beamline-531:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: beamline-531-agent
    ports:
      - "${AGENT_PORT:-8053}:8053"
    environment:
      - AGENT_PORT=8053
      - AGENT_NAME=beamline_531_control
      - HOSTNAME=beamline-531-agent  # Container name for network discovery
      - AGENT_ID=BL5.3.1  # Agent ID for scoped capability lookup
      # Optional: Auto-register with framework
      - FRAMEWORK_REGISTRY_URL=${FRAMEWORK_REGISTRY_URL}
      - AUTO_REGISTER=${AUTO_REGISTER:-false}
      # Beamline-specific env vars
      - EPICS_CA_ADDR_LIST=${EPICS_CA_ADDR_LIST}
      - EPICS_CA_SERVER_PORT=${EPICS_CA_SERVER_PORT}
      - EPICS_CA_MAX_ARRAY_BYTES=${EPICS_CA_MAX_ARRAY_BYTES:-16384}
      # API Keys
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - GEMINI_API_KEY=${GEMINI_API_KEY}
      - PYTHONPATH=/app/src
    volumes:
      - ./src:/app/src:ro
      - ./config.yml:/app/config.yml:ro
    networks:
      - agent-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8053/health"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  agent-network:
    external: true  # Connect to framework network
```

### Agent: Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies including EPICS libraries
RUN apt-get update && apt-get install -y \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy and install from pyproject.toml
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Copy source code
COPY src/ /app/src/
COPY config.yml /app/

# Set Python path
ENV PYTHONPATH=/app/src

# Expose agent port
EXPOSE 8053

# Run the agent server
CMD ["python", "src/agent/server.py"]
```

### Agent: .env.example

```bash
# Agent Configuration
AGENT_PORT=8053
AGENT_NAME=beamline_531_control
AGENT_ID=BL5.3.1

# Framework Integration (Optional)
FRAMEWORK_REGISTRY_URL=http://agent-registry:8100
AUTO_REGISTER=false

# Beamline-Specific Configuration
EPICS_CA_ADDR_LIST=10.0.0.1
EPICS_CA_SERVER_PORT=5064
EPICS_CA_MAX_ARRAY_BYTES=16384

# API Keys
OPENAI_API_KEY=your_openai_key_here
GEMINI_API_KEY=your_gemini_key_here

# Safety Settings
ENABLE_MOTOR_WRITES=false
ENABLE_SHUTTER_CONTROL=false
MAX_MOTOR_SPEED=10.0
```

### Agent: pyproject.toml

```toml
[build-system]
requires = ["setuptools>=65.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "beamline-531-agent"
version = "0.1.0"
description = "Beamline 5.3.1 Control Agent for ALS"
readme = "README.md"
requires-python = ">=3.11"
license = {text = "MIT"}

dependencies = [
    "fastapi>=0.104.0",
    "uvicorn>=0.24.0",
    "pydantic>=2.5.0",
    "httpx>=0.25.0",
    "python-dotenv>=1.0.0",
    # EPICS dependencies
    "pyepics>=3.5.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "black>=23.0.0",
    "ruff>=0.1.0",
]
```

### Agent: config.yml

```yaml
# Agent Configuration
agent:
  name: beamline_531_control
  version: "0.1.0"
  description: "Control agent for ALS Beamline 5.3.1"
  agent_id: "BL5.3.1"  # Unique agent identifier

# Capabilities provided by this agent
capabilities:
  - name: motor_control
    description: "Control beamline motors (position, velocity, etc.)"
    always_active: true
  
  - name: beam_alignment
    description: "Perform beam alignment procedures"
    always_active: true
  
  - name: shutter_control
    description: "Open/close beamline shutters"
    always_active: true

# Agent-specific settings
settings:
  timeout: 30
  max_retries: 3

# EPICS configuration
epics:
  ca_addr_list: ${EPICS_CA_ADDR_LIST}
  ca_server_port: ${EPICS_CA_SERVER_PORT:-5064}
  ca_max_array_bytes: ${EPICS_CA_MAX_ARRAY_BYTES:-16384}

# Safety limits
safety:
  enable_motor_writes: ${ENABLE_MOTOR_WRITES:-false}
  enable_shutter_control: ${ENABLE_SHUTTER_CONTROL:-false}
  max_motor_speed: ${MAX_MOTOR_SPEED:-10.0}
```

## Usage Instructions

### Starting the Framework

```bash
# Copy environment file
cp .env.example .env
# Edit .env with your values

# Start core services only
docker compose --profile core up -d

# Start core + registry
docker compose --profile core --profile registry up -d

# Start core + UI
docker compose --profile core --profile ui up -d

# Start everything (including dev tools)
docker compose --profile core --profile ui --profile dev --profile registry up -d

# Stop all services
docker compose --profile core --profile ui --profile dev --profile registry down
```

### Starting an Agent

```bash
# In the agent repository
cp .env.example .env
# Edit .env with your values

# Create the shared network (if not exists)
docker network create agent-network

# Start the agent
docker compose up -d

# Check logs
docker compose logs -f

# Stop the agent
docker compose down
```

### Development Workflow

1. Clone framework repo - Set up core infrastructure
2. Clone agent repos - One for each agent you want to run
3. Start framework - `docker compose --profile core --profile registry up -d`
4. Start agents - In each agent repo: `docker compose up -d`
5. Agents connect - They automatically join the agent-network and register

## Migration Checklist

- [ ] Create new framework repo without `src/applications/`
- [ ] Add `agent_registry.py`, `agent_registry_server.py`, and `agent_client.py` to `framework/infrastructure/`
- [ ] Add `Dockerfile.registry` to framework root
- [ ] Create agent repo for each existing application
- [ ] Move capabilities from framework to agent
- [ ] Move agent-specific context classes to agent (if any)
- [ ] Add FastAPI server (`server.py`) to each agent
- [ ] Add `handlers.py` for routing to each agent
- [ ] Test framework deployment with registry service
- [ ] Test agent deployment and auto-registration
- [ ] Test framework-agent communication via HTTP
- [ ] Update documentation
- [ ] Archive old monorepo (optional)

## Key Changes Summary

### Framework Repository:
- ✅ Keep all existing framework code
- ➕ Add 3 new files: `agent_registry.py`, `agent_registry_server.py`, `agent_client.py`
- ➕ Add `Dockerfile.registry` for registry service
- ➖ Remove `src/applications/` directory
- ➖ Remove `deployment/` directory
- 📄 Move `docker-compose.yml` and `Dockerfile` to root
- 📄 Simplify `config.yml` (remove applications list)
- ➕ Use only `pyproject.toml` (no requirements.txt)
- ➕ Single `.env.example` at root

### Agent Repositories:
- ➕ Create new repo for each agent
- ➕ Add FastAPI HTTP server (`server.py`)
- 📦 Move capability code from framework
- ➕ Add Docker Compose + Dockerfile at root
- ➕ Add dedicated configuration

### No Logic Changes:
- All your existing capability logic stays the same
- LangGraph orchestration unchanged
- Registry system unchanged
- Just communication layer changes (local imports → HTTP)
