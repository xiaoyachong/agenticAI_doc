# Multi-Repo Agent Architecture Migration Guide

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
├── .gitignore                                    # No change
├── .env.example                                  # NEW - At root (ONLY ONE)
├── config.yml                                    # MODIFIED - Remove applications list
├── pyproject.toml                                # MODIFIED - Add framework dependencies
└── README.md                                     # MODIFIED - Update with new architecture
```

---

## Part 2: Agent Repository Structure (Example: beamline-531-agent)

```
beamline-531-agent/
│
├── src/
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── server.py                             # NEW - FastAPI HTTP server
│   │   ├── capabilities.py                       # Moved from framework
│   │   ├── context.py                            # Moved from framework
│   │   └── registry.py                           # Moved from framework
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
├── requirements.txt                              # NEW - Agent requirements
├── .gitignore
└── README.md                                     # NEW - Agent documentation
```

---

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
    capabilities: List[str]
    status: str = "active"
    registered_at: datetime
    last_heartbeat: Optional[datetime] = None


class AgentRegistry:
    """Registry for tracking remote agents and their capabilities."""
    
    def __init__(self):
        self._agents: Dict[str, AgentInfo] = {}
        self._capability_map: Dict[str, List[str]] = {}
    
    def register_agent(
        self,
        name: str,
        url: str,
        capabilities: List[str]
    ) -> None:
        """Register a new agent with its capabilities."""
        agent_info = AgentInfo(
            name=name,
            url=url,
            capabilities=capabilities,
            registered_at=datetime.now(),
            last_heartbeat=datetime.now()
        )
        
        self._agents[name] = agent_info
        
        # Update capability map
        for capability in capabilities:
            if capability not in self._capability_map:
                self._capability_map[capability] = []
            if name not in self._capability_map[capability]:
                self._capability_map[capability].append(name)
        
        logger.info(f"Registered agent '{name}' with capabilities: {capabilities}")
    
    def unregister_agent(self, name: str) -> None:
        """Remove an agent from the registry."""
        if name in self._agents:
            agent_info = self._agents[name]
            
            # Remove from capability map
            for capability in agent_info.capabilities:
                if capability in self._capability_map:
                    self._capability_map[capability].remove(name)
                    if not self._capability_map[capability]:
                        del self._capability_map[capability]
            
            del self._agents[name]
            logger.info(f"Unregistered agent '{name}'")
    
    def get_agent(self, name: str) -> Optional[AgentInfo]:
        """Get information about a specific agent."""
        return self._agents.get(name)
    
    def get_agents_for_capability(self, capability: str) -> List[AgentInfo]:
        """Get all agents that provide a specific capability."""
        agent_names = self._capability_map.get(capability, [])
        return [self._agents[name] for name in agent_names if name in self._agents]
    
    def list_all_agents(self) -> List[AgentInfo]:
        """List all registered agents."""
        return list(self._agents.values())
    
    def update_heartbeat(self, name: str) -> None:
        """Update the last heartbeat time for an agent."""
        if name in self._agents:
            self._agents[name].last_heartbeat = datetime.now()


# Global registry instance
_registry = AgentRegistry()


def get_agent_registry() -> AgentRegistry:
    """Get the global agent registry instance."""
    return _registry
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
"""FastAPI HTTP Server for Agent."""

import logging
from typing import Any, Dict, List, Optional
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import uvicorn

from agent.registry import get_agent_registry
from agent.capabilities import get_capability_handler

logger = logging.getLogger(__name__)

app = FastAPI(title="Agent Server")


class CapabilityRequest(BaseModel):
    """Request model for capability invocation."""
    parameters: Dict[str, Any]
    context: Dict[str, Any] = {}


class CapabilityResponse(BaseModel):
    """Response model for capability results."""
    success: bool
    result: Any
    error: Optional[str] = None


@app.get("/health")
async def health_check():
    """Health check endpoint."""
    return {"status": "healthy"}


@app.get("/capabilities")
async def list_capabilities():
    """List all available capabilities."""
    registry = get_agent_registry()
    capabilities = registry.list_capabilities()
    return {"capabilities": [cap.name for cap in capabilities]}


@app.post("/capability/{capability_name}")
async def invoke_capability(
    capability_name: str,
    request: CapabilityRequest
) -> CapabilityResponse:
    """
    Invoke a capability on this agent.
    
    Args:
        capability_name: Name of the capability to invoke
        request: Request containing parameters and context
        
    Returns:
        Result of the capability execution
    """
    try:
        # Get capability handler
        handler = get_capability_handler(capability_name)
        if not handler:
            raise HTTPException(
                status_code=404,
                detail=f"Capability '{capability_name}' not found"
            )
        
        # Execute capability
        result = await handler.execute(
            parameters=request.parameters,
            context=request.context
        )
        
        return CapabilityResponse(
            success=True,
            result=result
        )
        
    except Exception as e:
        logger.error(f"Capability execution failed: {e}")
        return CapabilityResponse(
            success=False,
            result=None,
            error=str(e)
        )


def start_server(host: str = "0.0.0.0", port: int = 8051):
    """Start the agent server."""
    uvicorn.run(app, host=host, port=port)


if __name__ == "__main__":
    import os
    host = os.getenv("AGENT_HOST", "0.0.0.0")
    port = int(os.getenv("AGENT_PORT", "8051"))
    start_server(host=host, port=port)
```

### Agent: capabilities.py

```python
"""Agent Capabilities - Moved from framework/applications."""

from typing import Any, Dict, Optional
from pydantic import BaseModel


class CapabilityHandler:
    """Base class for capability handlers."""
    
    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description
    
    async def execute(
        self,
        parameters: Dict[str, Any],
        context: Dict[str, Any]
    ) -> Any:
        """Execute the capability logic."""
        raise NotImplementedError


# Motor Control Capability
class MotorControlCapability(CapabilityHandler):
    """Control motors on Beamline 5.3.1."""
    
    def __init__(self):
        super().__init__(
            name="motor_control",
            description="Control beamline motors (position, velocity, etc.)"
        )
    
    async def execute(
        self,
        parameters: Dict[str, Any],
        context: Dict[str, Any]
    ) -> Any:
        """Execute motor control logic."""
        motor_name = parameters.get("motor_name", "")
        target_position = parameters.get("target_position", 0)
        
        # Your existing motor control logic here
        # This is the same code from src/applications/beamline_531/capabilities/
        
        # Example: EPICS motor control
        pv = f"BL531:{motor_name}:POSITION"
        # caput(pv, target_position)
        
        return {
            "success": True,
            "motor": motor_name,
            "position": target_position,
            "message": f"Motor {motor_name} moved to {target_position}"
        }


# Beam Alignment Capability
class BeamAlignmentCapability(CapabilityHandler):
    """Align the beam on Beamline 5.3.1."""
    
    def __init__(self):
        super().__init__(
            name="beam_alignment",
            description="Perform beam alignment procedures"
        )
    
    async def execute(
        self,
        parameters: Dict[str, Any],
        context: Dict[str, Any]
    ) -> Any:
        """Execute beam alignment logic."""
        alignment_type = parameters.get("type", "auto")
        
        # Your existing beam alignment logic here
        
        return {
            "success": True,
            "alignment_type": alignment_type,
            "beam_centered": True,
            "message": "Beam alignment completed successfully"
        }


# Shutter Control Capability
class ShutterControlCapability(CapabilityHandler):
    """Control shutters on Beamline 5.3.1."""
    
    def __init__(self):
        super().__init__(
            name="shutter_control",
            description="Open/close beamline shutters"
        )
    
    async def execute(
        self,
        parameters: Dict[str, Any],
        context: Dict[str, Any]
    ) -> Any:
        """Execute shutter control logic."""
        shutter_id = parameters.get("shutter_id", "")
        action = parameters.get("action", "open")  # "open" or "close"
        
        # Your existing shutter control logic here
        pv = f"BL531:SHUTTER:{shutter_id}:STATE"
        # caput(pv, 1 if action == "open" else 0)
        
        return {
            "success": True,
            "shutter": shutter_id,
            "state": action,
            "message": f"Shutter {shutter_id} {action}ed"
        }


# Capability registry for this agent
_capabilities = {
    "motor_control": MotorControlCapability(),
    "beam_alignment": BeamAlignmentCapability(),
    "shutter_control": ShutterControlCapability()
}


def get_capability_handler(name: str) -> Optional[CapabilityHandler]:
    """Get a capability handler by name."""
    return _capabilities.get(name)
```

### Agent: registry.py

```python
"""Agent Registry - List capabilities for this agent."""

from typing import List
from pydantic import BaseModel


class CapabilityInfo(BaseModel):
    """Information about a capability."""
    name: str
    description: str


class AgentRegistry:
    """Registry of capabilities provided by this agent."""
    
    def __init__(self):
        self._capabilities: List[CapabilityInfo] = []
    
    def register_capability(self, name: str, description: str):
        """Register a capability."""
        self._capabilities.append(
            CapabilityInfo(name=name, description=description)
        )
    
    def list_capabilities(self) -> List[CapabilityInfo]:
        """List all registered capabilities."""
        return self._capabilities


# Global registry
_agent_registry = AgentRegistry()

# Register capabilities for Beamline 5.3.1
_agent_registry.register_capability(
    "motor_control",
    "Control beamline motors (position, velocity, etc.)"
)
_agent_registry.register_capability(
    "beam_alignment",
    "Perform beam alignment procedures"
)
_agent_registry.register_capability(
    "shutter_control",
    "Open/close beamline shutters"
)


def get_agent_registry() -> AgentRegistry:
    """Get the agent registry."""
    return _agent_registry
```

---

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

---

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
      - AGENT_HOST=0.0.0.0
      # Optional: Auto-register with framework
      - FRAMEWORK_REGISTRY_URL=${FRAMEWORK_REGISTRY_URL}
      - AUTO_REGISTER=${AUTO_REGISTER:-false}
      # Beamline-specific env vars
      - BEAMLINE_ID=5.3.1
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

# Copy requirements and install
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

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
AGENT_HOST=0.0.0.0

# Framework Integration (Optional)
FRAMEWORK_REGISTRY_URL=http://agent-registry:8100
AUTO_REGISTER=false

# Beamline-Specific Configuration
BEAMLINE_ID=5.3.1
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

### Agent: requirements.txt

```
fastapi>=0.104.0
uvicorn>=0.24.0
pydantic>=2.5.0
httpx>=0.25.0
python-dotenv>=1.0.0
pyepics>=3.5.0
```

### Agent: config.yml

```yaml
# Agent Configuration
agent:
  name: beamline_531_control
  version: "0.1.0"
  description: "Control agent for ALS Beamline 5.3.1"
  beamline_id: "5.3.1"

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

---

## Usage Instructions

### Starting the Framework

```bash
# Copy environment file
cp .env.example .env
# Edit .env with your values

# Start core services only
docker compose --profile core up -d

# Start core + UI
docker compose --profile core --profile ui up -d

# Start everything (including dev tools)
docker compose --profile core --profile ui --profile dev up -d

# Stop all services
docker compose --profile core --profile ui --profile dev down
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

1. **Clone framework repo** - Set up core infrastructure
2. **Clone agent repos** - One for each agent you want to run
3. **Start framework** - `docker compose --profile core up -d`
4. **Start agents** - In each agent repo: `docker compose up -d`
5. **Agents connect** - They automatically join the `agent-network`

---

## Migration Checklist

- [ ] Create new framework repo without `src/applications/`
- [ ] Add `agent_registry.py` and `agent_client.py` to framework
- [ ] Create agent repo for each existing application
- [ ] Move capabilities/context/registry from framework to agent
- [ ] Add FastAPI server to each agent
- [ ] Test framework deployment
- [ ] Test agent deployment
- [ ] Test framework-agent communication
- [ ] Update documentation
- [ ] Archive old monorepo (optional)

---

## Key Changes Summary

**Framework Repository:**
- ✅ Keep all existing framework code
- ➕ Add 2 new files: `agent_registry.py`, `agent_client.py`
- ➖ Remove `src/applications/` directory
- ➖ Remove `deployment/` directory
- 🔄 Move `docker-compose.yml` and `Dockerfile` to root
- 🔄 Simplify `config.yml` (remove applications list)
- ➕ Use only `pyproject.toml` (no requirements.txt)
- ➕ Single `.env.example` at root

**Agent Repositories:**
- ➕ Create new repo for each agent
- ➕ Add FastAPI HTTP server (`server.py`)
- 📦 Move capability code from framework
- ➕ Add Docker Compose + Dockerfile at root
- ➕ Add dedicated configuration

**No Logic Changes:**
- All your existing capability logic stays the same
- LangGraph orchestration unchanged
- Registry system unchanged
- Just communication layer changes (local imports → HTTP)
