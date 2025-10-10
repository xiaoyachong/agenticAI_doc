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
├── deployment/                                   # MODIFIED - Simplified deployment
│   ├── docker-compose.yml                        # NEW - Main compose file
│   ├── Dockerfile                                # NEW - Framework image
│   └── .env.example                              # NEW - Environment template
│
├── tests/                                        # No change
├── docs/                                         # No change
│   └── agent-development-guide.md                # NEW - Guide for building agents
│
├── .gitignore                                    # No change
├── .env.example                                  # NEW - Root env template
├── config.yml                                    # MODIFIED - Remove applications list
├── pyproject.toml                                # No change
├── requirements.txt                              # No change
└── README.md                                     # MODIFIED - Update with new architecture
```

---

## Part 2: Agent Repository Structure (Example: pv-finder-agent)

```
pv-finder-agent/
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
│       └── pv_search.py                          # Domain logic
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
from typing import Any, Dict, List
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


# Example: PV Finder Capability
class PVFinderCapability(CapabilityHandler):
    """Find PVs in the ALS control system."""
    
    def __init__(self):
        super().__init__(
            name="find_pv",
            description="Find PVs based on natural language query"
        )
    
    async def execute(
        self,
        parameters: Dict[str, Any],
        context: Dict[str, Any]
    ) -> Any:
        """Execute PV finding logic."""
        query = parameters.get("query", "")
        
        # Your existing PV finding logic here
        # This is the same code from src/applications/als_assistant/capabilities/
        
        return {
            "pvs": ["SR:C01:BI:CURRENT", "SR:C02:BI:CURRENT"],
            "query": query
        }


# Capability registry for this agent
_capabilities = {
    "find_pv": PVFinderCapability()
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

# Register capabilities
_agent_registry.register_capability(
    "find_pv",
    "Find PVs in the ALS control system"
)


def get_agent_registry() -> AgentRegistry:
    """Get the agent registry."""
    return _agent_registry
```

---

## Part 4: Docker Configuration Files

### Framework: docker-compose.yml

```yaml
version: '3.8'

name: alpha-framework

services:
  # LangGraph API Server
  framework-api:
    build:
      context: .
      dockerfile: deployment/Dockerfile
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
      dockerfile: deployment/Dockerfile.registry
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

### Framework: deployment/Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

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

### Framework: .env.example

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

### Framework: Modified config.yml

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

name: pv-finder-agent

services:
  pv-finder:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: pv-finder-agent
    ports:
      - "${AGENT_PORT:-8051}:8051"
    environment:
      - AGENT_PORT=8051
      - AGENT_NAME=pv_finder
      - AGENT_HOST=0.0.0.0
      # Optional: Auto-register with framework
      - FRAMEWORK_REGISTRY_URL=${FRAMEWORK_REGISTRY_URL}
      - AUTO_REGISTER=${AUTO_REGISTER:-false}
      # Domain-specific env vars
      - CBORG_API_KEY=${CBORG_API_KEY}
      - CBORG_API_URL=${CBORG_API_URL}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - GEMINI_API_KEY=${GEMINI_API_KEY}
      - HTTPS_PROXY=${HTTPS_PROXY}
      - NO_PROXY=${NO_PROXY}
      - PYTHONPATH=/app/src
    volumes:
      - ./src:/app/src:ro
      - ./config.yml:/app/config.yml:ro
    networks:
      - agent-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8051/health"]
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

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
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
EXPOSE 8051

# Run the agent server
CMD ["python", "src/agent/server.py"]
```

### Agent: .env.example

```bash
# Agent Configuration
AGENT_PORT=8051
AGENT_NAME=pv_finder
AGENT_HOST=0.0.0.0

# Framework Integration (Optional)
FRAMEWORK_REGISTRY_URL=http://agent-registry:8100
AUTO_REGISTER=false

# Domain-Specific Configuration
CBORG_API_KEY=your_cborg_key_here
CBORG_API_URL=https://cborg.lbl.gov/api

# API Keys
OPENAI_API_KEY=your_openai_key_here
GEMINI_API_KEY=your_gemini_key_here

# Proxy Settings (if needed)
HTTPS_PROXY=
NO_PROXY=localhost,127.0.0.1
```

### Agent: pyproject.toml

```toml
[build-system]
requires = ["setuptools>=65.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "pv-finder-agent"
version = "0.1.0"
description = "PV Finder Agent for ALS Control System"
readme = "README.md"
requires-python = ">=3.11"
license = {text = "MIT"}

dependencies = [
    "fastapi>=0.104.0",
    "uvicorn>=0.24.0",
    "pydantic>=2.5.0",
    "httpx>=0.25.0",
    "python-dotenv>=1.0.0",
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
```

### Agent: config.yml

```yaml
# Agent Configuration
agent:
  name: pv_finder
  version: "0.1.0"
  description: "Find PVs in the ALS control system"

# Capabilities provided by this agent
capabilities:
  - name: find_pv
    description: "Find PVs based on natural language query"
    always_active: true

# Agent-specific settings
settings:
  timeout: 30
  max_retries: 3

# Domain-specific configuration
cborg:
  api_url: ${CBORG_API_URL}
  timeout: 10
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
- 🔄 Simplify `config.yml` (remove applications list)
- ➕ Add standard Docker Compose setup

**Agent Repositories:**
- ➕ Create new repo for each agent
- ➕ Add FastAPI HTTP server (`server.py`)
- 📦 Move capability code from framework
- ➕ Add Docker Compose + Dockerfile
- ➕ Add dedicated configuration

**No Logic Changes:**
- All your existing capability logic stays the same
- LangGraph orchestration unchanged
- Registry system unchanged
- Just communication layer changes (local imports → HTTP)
