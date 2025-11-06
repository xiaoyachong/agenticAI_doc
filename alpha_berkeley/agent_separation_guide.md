# Beamline Control Service Architecture Guide

## Overview

This document outlines two architectural approaches for deploying the Alpha Berkeley Framework for beamline control systems, maintaining the existing file structure from the alpha-berkeley repository.

## Option 1: Package-Based Modular Design

### Conceptual Architecture

```
┌─────────────────────────────────────────────┐
│       alpha-berkeley-framework              │
│         (PyPI/GitHub Package)               │
│                                             │
│  • LangGraph Workflow Engine                │
│  • Base Node Classes                        │
│  • State Management                         │
│  • Common Utilities                         │
│  • Registry System                          │
└─────────────────┬───────────────────────────┘
                  │ pip install
                  ▼
┌─────────────────────────────────────────────┐
│      Beamline Application Repositories      │
├─────────────────┬───────────────────────────┤
│   bl531-app     │   bl701-app         │ ... │
│                 │                     │     │
│  Imports        │  Imports            │     │
│  framework      │  framework          │     │
│  Implements:    │  Implements:        │     │
│  • Capabilities │  • Capabilities     │     │
│  • Config       │  • Config           │     │
└─────────────────┴───────────────────────────┘
                  ▼
    Deployed as Independent Instances
    (Each is a complete framework instance)
```

### Repository Structure

#### Core Framework Package Repository (Minimal Changes)

```
alpha-berkeley-framework/            # Becomes a pip-installable package
├── src/
│   ├── applications/               # REMOVED (moved to separate repos)
│   ├── configs/                    # Keep for global configs
│   └── framework/
│       ├── __init__.py
│       ├── base/                   # No change - Base classes
│       ├── capabilities/           # Keep common capabilities only
│       │   ├── __init__.py
│       │   ├── memory.py          # Common capabilities
│       │   └── time_parser.py
│       ├── context/                # No change
│       ├── data_sources/           # No change
│       ├── graph/                  # No change - LangGraph orchestration
│       ├── infrastructure/         # No change
│       ├── nodes/                  # No change - Core nodes
│       ├── prompts/                # No change
│       ├── registry/               # No change
│       └── services/               # No change
├── build_dir/                      # No change
├── deployment/                     # Update for package deployment
├── docs/                          # No change
├── interfaces/                    # No change
├── tests/                         # No change
├── pyproject.toml                 # MODIFIED - Make pip-installable
└── setup.py                       # NEW - For package distribution
```

#### Beamline Application Repository (Example: BL5.3.1)

```
bl531-application/                  # New repo for beamline
├── src/
│   └── bl531/
│       ├── __init__.py
│       ├── main.py                # Entry point
│       ├── capabilities/          # BL5.3.1 specific capabilities
│       │   ├── __init__.py
│       │   ├── motor_control.py
│       │   ├── beam_alignment.py
│       │   └── xray_monochromator.py
│       ├── context/               # BL5.3.1 specific context
│       │   └── beamline_context.py
│       └── config/               # BL5.3.1 configuration
│           ├── config.yml
│           └── capabilities.yml
├── deployment/
│   ├── Dockerfile
│   └── docker-compose.yml
├── tests/
├── pyproject.toml                 # Depends on alpha-berkeley-framework
└── README.md
```

### Implementation Approach

The core framework becomes a Python package that beamline applications import. Each beamline application:

1. **Installs the framework**: `pip install alpha-berkeley-framework`
2. **Implements specific capabilities**: Following the framework's base classes
3. **Configures the workflow**: Using the framework's configuration system
4. **Runs the complete framework**: All nodes execute in-process with direct function calls

### Deployment

```yaml
# bl531-application/deployment/docker-compose.yml
services:
  bl531:
    build: .
    container_name: bl531-framework
    environment:
      - CONFIG_PATH=/app/config/config.yml
    ports:
      - "8531:8000"
```

---

## Option 2: Service-Oriented Architecture

### Conceptual Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 alpha-berkeley-framework                    │
│                  (Central Orchestrator)                     │
│                                                             │
│  ┌────────────────────────────────────────────────────┐     │
│  │              LangGraph Workflow Engine             │     │
│  │  src/framework/graph/  src/framework/nodes/        │     │
│  └─────────────────────────────────────────┬──────────┘     │
│                                            │                │
│  ┌─────────────────────────────────────────┼──────────┐     │
│  │    src/framework/infrastructure/        │          │     │
│  │         Service Registry                ▼          │     │
│  │  ┌──────────────────────────────────────────────┐  │     │
│  │  │ "BL5.3.1" → ServiceInfo(url, capabilities)   │  │     │
│  │  └──────────────────────────────────────────────┘  │     │
│  └────────────────────────────────────────────────────┘     │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP API
                         ▼
    ┌────────────────────┼────────────────────────────┐
    │                    │                            │
┌───▼──────────┐   ┌─────▼────────┐   ┌───────────────▼──┐
│ BL5.3.1      │   │ BL7.0.1      │   │ BL12.2.2         │
│ Service      │   │ Service      │   │ Service          │
└──────────────┘   └──────────────┘   └──────────────────┘
```

### Repository Structure

#### Framework Orchestrator Repository (Using Existing Structure)

```
alpha-berkeley-framework/
├── src/
│   ├── applications/               # REMOVED (services are separate)
│   ├── configs/                    # No change
│   └── framework/
│       ├── base/                   # No change
│       ├── capabilities/           # REMOVED (moved to services)
│       ├── context/                # No change
│       ├── data_sources/           # No change
│       ├── graph/                  # No change
│       ├── infrastructure/         # MODIFIED - Add service communication
│       │   ├── __init__.py
│       │   ├── gateway.py         # No change
│       │   ├── service_registry.py # NEW - Track remote services
│       │   └── service_client.py   # NEW - HTTP client for services
│       ├── nodes/                  # MODIFIED - Use service_client
│       │   ├── router.py          # Modified to call remote services
│       │   └── ...
│       ├── prompts/                # No change
│       ├── registry/               # No change
│       └── services/               # No change
├── deployment/
│   ├── orchestrator/              # NEW - Orchestrator deployment
│   │   ├── Dockerfile
│   │   └── docker-compose.yml
│   └── registry/                  # NEW - Registry service
│       ├── Dockerfile
│       └── docker-compose.yml
└── ...
```

#### Beamline Service Repository (Example: BL5.3.1)

```
beamline-531-service/              # Separate repository
├── src/
│   └── service/
│       ├── __init__.py
│       ├── api.py                 # FastAPI endpoints
│       ├── capabilities/          # Capability implementations
│       │   ├── motor_control.py
│       │   └── beam_alignment.py
│       ├── hardware/              # Hardware interfaces
│       │   └── epics_client.py
│       └── config/
│           └── beamline.yml
├── deployment/
│   ├── Dockerfile
│   └── docker-compose.yml
└── pyproject.toml
```

### Implementation Approach

The framework becomes a central orchestrator that:

1. **Runs the LangGraph workflow**: Classifier, Orchestrator, Router nodes
2. **Maintains a service registry**: Tracks available beamline services
3. **Makes HTTP calls**: Router node calls remote services via HTTP
4. **Manages state**: Workflow state and checkpointing remain central

Each beamline service:

1. **Implements a REST API**: Exposes capabilities as HTTP endpoints
2. **Registers with orchestrator**: On startup, announces available capabilities
3. **Handles hardware**: Direct EPICS/hardware communication
4. **Operates independently**: Can be deployed/scaled separately

### Communication Flow

```python
# In framework/nodes/router.py (modified)
async def router_node(state: State) -> State:
    # Get service info from registry
    service = registry.get_service(state["agent_id"])
    
    # Call service via HTTP
    client = ServiceClient()
    result = await client.call_capability(
        service_url=service.url,
        capability=state["capability"],
        parameters=state["parameters"]
    )
    
    return {"result": result}
```

## Key Differences

### Option 1 (Package-Based):
- Minimal changes to existing framework structure
- Framework becomes a pip package
- Each beamline imports and extends the framework
- All execution happens in-process (no HTTP)
- Simple deployment, one container per beamline

### Option 2 (Service-Oriented):
- Framework stays as orchestrator only
- Capabilities moved to separate service repositories
- Communication via HTTP APIs
- More complex deployment with multiple services
- Better isolation but higher complexity

## Recommendation

**Option 1 (Package-Based)** is recommended for the Alpha Berkeley Framework because:

1. **Minimal structural changes** to the existing codebase
2. **Preserves the framework design** while enabling modularity
3. **Simpler architecture** without distributed system complexity
4. **Better performance** for real-time hardware control
5. **Easier transition** from the current monolithic structure

The existing `src/framework/` structure remains largely unchanged, just packaged for distribution. Beamline-specific code moves to separate repositories that import and extend the framework.