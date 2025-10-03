# BL531 Application - Beamstop Diode Read Capability

## Overview

This document describes the BL531 application implementation for reading the beamstop_diode signal value. The application consists of client-side code that interfaces with an existing Bluesky Queue Server through REST API calls.

## Table of Contents

- [Application Architecture](#application-architecture)
- [Server Side Configuration](#server-side-configuration)
- [Client Side Implementation](#client-side-implementation)
- [File Structure](#file-structure)
- [Usage](#usage)

## Application Architecture

The BL531 application follows a modular capability-based architecture:

- **BL531 API Layer**: Handles communication with the queue server via REST API
- **Capability Layer**: Defines the beamstop_diode read capability
- **Context Layer**: Stores and manages operation results
- **Registry**: Registers capabilities and context types with the framework

### Communication Flow

```
User Query → Framework → BeamstopDiodeReadCapability → BL531API → Queue Server → Tiled → Result
```

1. User asks: "What is the beamstop diode value?"
2. Framework classifies the task and routes to BeamstopDiodeReadCapability
3. Capability calls `bl531_api.get_beamstop_diode()`
4. API executes count plan via queue server
5. Queue server runs the plan and stores data in Tiled
6. API retrieves result from Tiled
7. Result is packaged in context and returned to user

## Server Side Configuration

### What Already Exists (No Modifications Needed ✓)

The BL531 application uses existing queue server infrastructure. No changes are required to the server-side code.

#### 1. Device Definition

In `queue-server/startup_bl531/01_devices.py`:

```python
beamstop_diode = EpicsSignal("bl201-beamstop:current", name="beamstop_diode")
```

This device is already defined and available.

#### 2. Built-in Count Plan

The count plan is automatically available because it's imported in `queue-server/startup_bl531/15_plans.py`:

```python
from bluesky.plans import (
    count,  # <-- Built-in Bluesky plan
    scan as _scan,
    # ... other plans
)
```

#### 3. Auto-Generated Registry

When the queue server starts, it automatically scans all startup files and generates `existing_plans_and_devices.yaml`.

Example from `existing_plans_and_devices.yaml`:

```yaml
existing_devices:
  beamstop_diode:
    classname: EpicsSignal
    is_flyable: false
    is_movable: true
    is_readable: true
    module: ophyd.signal

existing_plans:
  count:
    description: Take one or more readings from detectors.
    module: bluesky.plans
    name: count
    parameters:
    - annotation:
        type: collections.abc.Sequence[__READABLE__]
      convert_device_names: true
      description: list of 'readable' objects
      name: detectors
    - annotation:
        type: typing.Optional[int]
      default: '1'
      description: 'number of readings to take; default is 1'
      name: num
    properties:
      is_generator: true
```

#### 4. How the Auto-Generation Works

The queue server process:

1. Scans startup files (`00_base.py`, `01_devices.py`, `15_plans.py`, etc.)
2. Discovers all devices and plans defined in those files
3. Imports built-in Bluesky plans (like `count`, `scan`, etc.)
4. Generates `existing_plans_and_devices.yaml` listing everything available
5. Exposes these via REST API for client applications

#### 5. Existing Infrastructure

- ✓ Tiled server running at `http://127.0.0.1:8000`
- ✓ Queue server running at `http://127.0.0.1:8003`
- ✓ TiledWriter callback configured to store data
- ✓ API authentication configured

### Why No Modifications Are Needed

The BL531 application only needs to read the beamstop_diode value, which is accomplished by:

1. Calling the existing `count` plan through the queue server API
2. Retrieving the stored data from Tiled

Both of these capabilities already exist in your server configuration!

## Client Side Implementation

### File Structure

```
src/applications/bl531/
├── __init__.py
├── bl531_api.py
├── capabilities/
│   ├── __init__.py
│   └── beamstop_diode_read.py
├── context_classes.py
├── config.yml
└── registry.py
```

### 1. BL531 API (bl531_api.py)

This module handles all communication with the queue server and Tiled.

```python
"""
BL531 Beamline API Interface.
Simple interface for beamstop_diode signal read operation.
"""
import requests
import time
from datetime import datetime
from dataclasses import dataclass
from configs.logger import get_logger

logger = get_logger("bl531", "api")


@dataclass
class CurrentDiodeReading:
    """Beamstop diode reading data."""
    value: float
    condition: str
    timestamp: datetime


class BL531API:
    """BL531 beamline API for beamstop_diode."""
    
    def __init__(self):
        self.session = requests.Session()
        self.session.headers.update({
            "accept": "application/json",
            "Authorization": "Apikey test",
            "Content-Type": "application/json"
        })
        self.base_url = "http://127.0.0.1:8003/api"
        self.tiled_url = "http://127.0.0.1:8000"
        
    def _execute_plan(self, plan_name: str, kwargs: dict) -> str:
        """Execute a plan and return item_uid."""
        response = self.session.post(
            f"{self.base_url}/queue/item/execute",
            json={
                "item": {
                    "name": plan_name,
                    "args": [],
                    "kwargs": kwargs,
                    "item_type": "plan",
                    "user": "UNAUTHENTICATED_SINGLE_USER",
                    "user_group": "primary"
                }
            }
        )
        response.raise_for_status()
        return response.json()["item"]["item_uid"]
    
    def _wait_for_completion(self, timeout: int = 300) -> bool:
        """Wait for queue completion."""
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            response = self.session.get(f"{self.base_url}/re/runs/active")
            response.raise_for_status()
            data = response.json()
            
            if not data.get("run_list", []):
                return True
            
            time.sleep(1)
        
        raise TimeoutError(f"Operation did not complete within {timeout} seconds")
    
    def _get_run_id(self, item_uid: str) -> str:
        """Get run_id from item_uid."""
        response = self.session.get(f"{self.base_url}/history/get")
        response.raise_for_status()
        history_data = response.json()
        
        for item in history_data["items"]:
            if item["item_uid"] == item_uid:
                run_uids = item["result"].get("run_uids", [])
                if run_uids:
                    return run_uids[0]
                raise ValueError(f"No run_uids found for item {item_uid}")
        
        raise ValueError(f"No matching item found for UID: {item_uid}")
    
    def get_beamstop_diode(self) -> CurrentDiodeReading:
        """Get current value of beamstop_diode."""
        try:
            # Execute count plan to read the signal
            item_uid = self._execute_plan(
                "count",
                {
                    "detectors": ["beamstop_diode"],
                    "num": 1
                }
            )
            
            # Wait for completion
            self._wait_for_completion()
            
            # Get run_id
            run_id = self._get_run_id(item_uid)
            
            # Get data from Tiled
            from tiled.client import from_uri
            import os
            
            api_key = os.getenv("TILED_SINGLE_USER_API_KEY")
            tiled_client = from_uri(self.tiled_url, api_key=api_key)
            run_data = tiled_client[run_id]
            
            # Extract value from primary data
            primary = run_data['primary']['data']
            value = float(primary['beamstop_diode'].read()[0])
            
            return CurrentDiodeReading(
                value=value,
                condition="success",
                timestamp=datetime.now()
            )
            
        except Exception as e:
            logger.error(f"Error reading beamstop_diode: {e}")
            return CurrentDiodeReading(
                value=-999.0,
                condition=f"Error: {str(e)}",
                timestamp=datetime.now()
            )


# Global API instance
bl531_api = BL531API()
```

### 2. Context Class (context_classes.py)

Defines the data structure for storing beamstop_diode readings.

```python
"""BL531 Context Classes."""
from datetime import datetime
from typing import Dict, Any, Optional, ClassVar
from pydantic import Field
from framework.context.base import CapabilityContext


class BeamstopDiodeReadContext(CapabilityContext):
    """Context for beamstop_diode read operation."""
    
    CONTEXT_TYPE: ClassVar[str] = "BEAMSTOP_DIODE_READ"
    CONTEXT_CATEGORY: ClassVar[str] = "LIVE_DATA"
    
    value: float = Field(description="Diode current value")
    condition: str = Field(description="Reading status")
    timestamp: datetime = Field(description="Reading timestamp")
    
    def get_access_details(self, key_name: Optional[str] = None) -> Dict[str, Any]:
        """Provide structured access information."""
        key_ref = key_name if key_name else "key_name"
        
        return {
            "diode_value": self.value,
            "status": self.condition,
            "access_pattern": f"context.{self.CONTEXT_TYPE}.{key_ref}.value",
            "available_fields": ["value", "condition", "timestamp"]
        }
    
    def get_human_summary(self, key: str) -> dict:
        """Generate human-readable summary."""
        return {
            "summary": f"Beamstop diode current: {self.value} A (status: {self.condition})"
        }
```

### 3. Read Capability (capabilities/beamstop_diode_read.py)

The main capability that executes the beamstop_diode read operation.

```python
"""Read beamstop_diode value capability."""
from typing import Dict, Any, Optional

from framework.base import (
    BaseCapability, 
    capability_node,
    ClassifierActions, 
    ClassifierExample, 
    TaskClassifierGuide
)
from framework.base.errors import ErrorClassification, ErrorSeverity
from framework.registry import get_registry
from framework.state import AgentState, StateManager
from configs.logger import get_logger
from configs.streaming import get_streamer

from applications.bl531.context_classes import BeamstopDiodeReadContext
from applications.bl531.bl531_api import bl531_api

logger = get_logger("bl531", "beamstop_diode_read")
registry = get_registry()


@capability_node
class BeamstopDiodeReadCapability(BaseCapability):
    """Read current value of beamstop_diode signal."""
    
    name = "beamstop_diode_read"
    description = "Read current value of beamstop_diode"
    provides = ["BEAMSTOP_DIODE_READ"]
    requires = []
    
    @staticmethod
    async def execute(state: AgentState, **kwargs) -> Dict[str, Any]:
        """Execute beamstop_diode read."""
        step = StateManager.get_current_step(state)
        streamer = get_streamer("bl531", "beamstop_diode_read", state)
        
        try:
            streamer.status("Reading beamstop_diode...")
            
            # Read diode value
            diode_data = bl531_api.get_beamstop_diode()
            
            # Create context
            context = BeamstopDiodeReadContext(
                value=diode_data.value,
                condition=diode_data.condition,
                timestamp=diode_data.timestamp
            )
            
            # Store context
            context_updates = StateManager.store_context(
                state,
                registry.context_types.BEAMSTOP_DIODE_READ,
                step.get("context_key"),
                context
            )
            
            streamer.status(f"Beamstop diode current: {diode_data.value} A")
            return context_updates
            
        except Exception as e:
            logger.error(f"Beamstop diode read error: {e}")
            raise
    
    @staticmethod
    def classify_error(exc: Exception, context: dict) -> ErrorClassification:
        """Classify errors."""
        if isinstance(exc, (ConnectionError, TimeoutError)):
            return ErrorClassification(
                severity=ErrorSeverity.RETRIABLE,
                metadata={
                    "user_message": "Communication timeout, retrying...",
                    "technical_details": str(exc)
                }
            )
        
        return ErrorClassification(
            severity=ErrorSeverity.CRITICAL,
            metadata={
                "user_message": f"Beamstop diode read error: {str(exc)}",
                "technical_details": str(exc)
            }
        )
    
    @staticmethod
    def get_retry_policy() -> Dict[str, Any]:
        """Define retry policy."""
        return {
            "max_attempts": 3,
            "delay_seconds": 0.5,
            "backoff_factor": 1.5
        }
    
    def _create_classifier_guide(self) -> Optional[TaskClassifierGuide]:
        """Provide task classification guidance."""
        return TaskClassifierGuide(
            instructions="""Determine if the user wants to READ the current value of beamstop_diode in BL531 beamline.

BL531 CONTEXT: beamstop_diode is a signal that measures beam current.""",
            examples=[
                ClassifierExample(
                    query="What is the beamstop diode value?",
                    result=True,
                    reason="Request to read beamstop_diode value."
                ),
                ClassifierExample(
                    query="Read the beamstop diode",
                    result=True,
                    reason="Direct request to read the signal."
                ),
                ClassifierExample(
                    query="Check beamstop current",
                    result=True,
                    reason="Request to check beamstop_diode reading."
                ),
                ClassifierExample(
                    query="Get beamstop diode reading",
                    result=True,
                    reason="Request for beamstop_diode value."
                ),
                ClassifierExample(
                    query="Move beamstop to 5mm",
                    result=False,
                    reason="This is about motor movement, not reading beamstop_diode."
                ),
                ClassifierExample(
                    query="What tools do you have?",
                    result=False,
                    reason="General question, not about beamstop_diode."
                ),
            ],
            actions_if_true=ClassifierActions()
        )
```

### 4. Capabilities Init (capabilities/__init__.py)

```python
"""BL531 Beamline Capabilities Module."""
from .beamstop_diode_read import BeamstopDiodeReadCapability

__all__ = [
    'BeamstopDiodeReadCapability',
]
```

### 5. Registry (registry.py)

Registers the capability and context class with the framework.

```python
"""BL531 Beamline Application Registry."""
from framework.registry import (
    CapabilityRegistration,
    ContextClassRegistration,
    RegistryConfig,
    RegistryConfigProvider
)


class BL531RegistryProvider(RegistryConfigProvider):
    
    def get_registry_config(self) -> RegistryConfig:
        return RegistryConfig(
            capabilities=[
                CapabilityRegistration(
                    name="beamstop_diode_read",
                    module_path="applications.bl531.capabilities.beamstop_diode_read",
                    class_name="BeamstopDiodeReadCapability",
                    description="Read beamstop_diode value",
                    provides=["BEAMSTOP_DIODE_READ"],
                    requires=[]
                ),
            ],
            
            context_classes=[
                ContextClassRegistration(
                    context_type="BEAMSTOP_DIODE_READ",
                    module_path="applications.bl531.context_classes",
                    class_name="BeamstopDiodeReadContext"
                ),
            ]
        )
```

### 6. Config (config.yml)

Application configuration file.

```yaml
# BL531 Beamline Application Configuration

pipeline:
  name: "bl531"

logging:
  logging_colors:
    beamstop_diode_read: "cyan"
```

### 7. Application Init (__init__.py)

```python
"""BL531 Beamline Application."""
```

## Usage

### Example User Queries

The application can handle the following types of queries:

- ✓ "What is the beamstop diode value?"
- ✓ "Read the beamstop diode"
- ✓ "Check beamstop current"
- ✓ "Get beamstop diode reading"
- ✓ "Measure beamstop diode"

### Example Interaction

**User:** "What is the beamstop diode value?"

**System:** Reading beamstop_diode...  
**System:** Beamstop diode current: 2.3e-6 A
