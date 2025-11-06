# OpenWebUI Meta-Task Issues & Solutions

## Executive Summary

OpenWebUI's recent versions automatically generate meta-tasks after every successful chat, causing the Osprey agent's orchestrator to hallucinate non-existent capabilities and fail with validation errors. The immediate solution is to disable OpenWebUI's auto-generation features via environment variables.

---

## The Problem

### What's Happening

**OpenWebUI automatically generates meta-tasks** after every successful user interaction:
- "Generate a concise, 3-5 word title with an emoji..."
- "Generate 1-3 broad tags categorizing the main themes..."
- "Suggest 3-5 relevant follow-up questions..."

These are **not user queries** - they're system-generated requests for UI metadata.

### Why It Breaks Your Agent

Your Osprey agent treats these meta-tasks as real user queries:

```
User: "read the motor position"
  ↓
Agent: ✅ Successfully reads position
  ↓
OpenWebUI Auto-Triggers (background):
  → "### Task: Generate title..." → ❌ Agent fails
  → "### Task: Generate tags..." → ❌ Agent fails  
  → "### Task: Suggest questions..." → ❌ Agent fails
```

### The Orchestrator Hallucination Issue

When processing meta-tasks, the orchestrator **invents non-existent capabilities**:

**Hallucinated capabilities:**
- `question_generator`
- `tag_generator`
- `content_analyzer`
- `context_retrieval`
- `analysis`
- `content_generation`

**Available capabilities:**
- `respond`, `clarify`
- `motor_position_read`, `motor_position_set`
- `detector_image_capture`, `photogrammetry_scan_execute`
- `reconstruct_object`, `ply_quality_assessment`, `display_object`
- `memory`, `python`, `time_range_parsing`

**Why hallucination occurs:**
1. Task names semantically suggest specialized capabilities should exist
2. Available capabilities don't obviously map to "generate tags/questions"
3. Orchestrator LLM pattern-matches from training data where systems have `tag_generator`, `question_generator`, etc.

### Complete Error Pattern from Logs

**1. User query succeeds:**
```
User: "read the motor position..."
✅ Successfully executes: motor_position_read → respond
```

**2. OpenWebUI auto-generates follow-up question meta-task:**
```
Query: "### Task: Suggest 3-5 relevant follow-up questions..."
❌ Orchestrator hallucinates: 'analysis'
❌ Reclassification attempt 1: Orchestrator hallucinates: 'context_retrieval', 'content_generation'
❌ Reclassification exhausted → Error node
```

**3. OpenWebUI auto-generates title meta-task:**
```
Query: "### Task: Generate a concise, 3-5 word title with an emoji..."
✅ Successfully uses: respond (generates "📍 Motor Position Reading")
```

**4. OpenWebUI auto-generates tag meta-task:**
```
Query: "### Task: Generate 1-3 broad tags categorizing the main themes..."
❌ First attempt: Pydantic validation error (wrong parameter types)
❌ Retry attempt 1: Orchestrator hallucinates: 'analysis'
❌ Reclassification attempt 1: Orchestrator hallucinates: 'content_analyzer', 'tag_generator'
❌ Reclassification exhausted → Error node
```

**Key observations:**
- 2 out of 3 meta-tasks fail (follow-up questions and tags)
- 1 out of 3 succeeds (title - uses `respond` correctly)
- Multiple reclassification attempts, but orchestrator keeps hallucinating
- Different hallucinated capabilities each attempt shows LLM creativity/inconsistency

### The Validation Error

When validation detects hallucinated capabilities in `orchestration_node.py` (~line 100):

```python
if hallucinated_capabilities:
    error_msg = f"Orchestrator hallucinated non-existent capabilities: {hallucinated_capabilities}..."
    raise ValueError(error_msg)  # ❌ WRONG EXCEPTION TYPE!
```

**Problem**: This `ValueError` gets classified as **CRITICAL** instead of **RECLASSIFICATION**

**Result**:
- ❌ System doesn't properly retry
- ❌ Reclassification attempts exhaust (1/1)
- ❌ Routes to error node
- ❌ User sees error message after successful operations

---

## The Solutions

### 1. Immediate Fix: Disable OpenWebUI Auto-Generation ✅

**File to edit:** `services/open-webui/docker-compose.yml.j2`

**Add these environment variables:**

```yaml
volumes:
  open-webui:

services:       
  open-webui:
    container_name: open-webui
    build:
      context: ./open-webui
      dockerfile: Dockerfile
    restart: unless-stopped
    ports:
      - "{{services.open_webui.port_host}}:{{services.open_webui.port_container}}"
    environment:
      # Timezone synchronization with host (from system configuration)
      - TZ={{system.timezone}}
      - OLLAMA_BASE_URL={{api.providers.ollama.host}}:{{api.providers.ollama.port}}
      - WEBUI_CUSTOM_CSS_URL=/static/custom.css
      - USER_MEMORY_DIR=/app/{{file_paths.user_memory_dir}}
      
      # === DISABLE AUTO-GENERATION FEATURES (Fix for orchestrator hallucination) ===
      - ENABLE_COMMUNITY_SHARING=false
      - ENABLE_MESSAGE_RATING=false
      - ENABLE_ADMIN_EXPORT=false
      # These disable the problematic meta-task generation
      - ENABLE_TITLE_GENERATION=false
      - ENABLE_TAGS_GENERATION=false  
      - ENABLE_SEARCH_QUERY=false
      - ENABLE_RETRIEVAL_QUERY=false
      # === END AUTO-GENERATION SETTINGS ===
      
    volumes:
      - open-webui:/app/backend/data
      - {{project_root}}/{{file_paths.agent_data_dir}}:/app/backend/open_webui/static/agent_data:ro
      - ./open-webui/custom.css:/app/backend/static/custom.css
      - {{project_root}}/config.yml:/app/config.yml:ro
      - {{project_root}}/src:/app/src:ro
    networks:
      - osprey-network
```

**What these settings do:**
- `ENABLE_TITLE_GENERATION=false` - Stops automatic title generation
- `ENABLE_TAGS_GENERATION=false` - Stops automatic tag generation
- `ENABLE_RETRIEVAL_QUERY=false` - Stops follow-up question suggestions

**Apply the changes:**

```bash
# Navigate to project directory
cd /path/to/mlex_bolt_agent

# Restart OpenWebUI
docker-compose restart open-webui

# Or full rebuild if restart doesn't work:
docker-compose down
docker-compose up -d --build open-webui
```

**Verify it's working:**

```bash
# Check logs - should see NO more "### Task:" queries
docker-compose logs -f pipelines | grep "### Task"
```

You should **NOT** see any lines like:
- `Query: '### Task: Suggest 3-5 relevant follow-up questions...'`
- `Query: '### Task: Generate a concise, 3-5 word title...'`
- `Query: '### Task: Generate 1-3 broad tags...'`

**Why this works:**
- Prevents meta-tasks at the source
- OpenWebUI never sends them
- Orchestrator never receives them
- No hallucination can occur

---

### 2. Long-term Fix: Fix Osprey Framework Exception Type

**File to edit:** `src/osprey/infrastructure/orchestration_node.py`

**Location:** Function `_validate_and_fix_execution_plan()` (around line 100)

**Current (incorrect) code:**
```python
if hallucinated_capabilities:
    error_msg = (
        f"Orchestrator hallucinated non-existent capabilities: {hallucinated_capabilities}. "
        f"Available capabilities: {registry.get_stats()['capability_names']}"
    )
    logger.error(error_msg)
    raise ValueError(error_msg)  # ❌ Gets classified as CRITICAL
```

**Fixed (correct) code:**
```python
if hallucinated_capabilities:
    error_msg = (
        f"Orchestrator hallucinated non-existent capabilities: {hallucinated_capabilities}. "
        f"Available capabilities: {registry.get_stats()['capability_names']}"
    )
    logger.error(error_msg)
    raise ReclassificationRequiredError(error_msg)  # ✅ Gets classified as RECLASSIFICATION
```

**Why this is needed:**
- Enables proper error recovery for ANY hallucination scenario
- Not just meta-tasks - any future capability hallucination
- Allows the router to properly retry with reclassification
- Framework already has infrastructure to handle `ReclassificationRequiredError`

**Impact:**
- Framework-level fix benefits all use cases
- Proper error recovery instead of immediate failure
- Better user experience even if new hallucination patterns emerge

---

## Why This Happened

### Timeline

1. **Older OpenWebUI versions**: No auto-generation features → Everything worked
2. **Recent OpenWebUI upgrade**: Added auto-generation (titles, tags, follow-up questions) → **enabled by default**
3. **Your agent**: Designed for real user queries, not meta-tasks → Starts failing

### Root Causes

**OpenWebUI Side:**
- New features enabled by default
- No documentation about potential agent compatibility issues
- Meta-tasks formatted like real queries

**Osprey Framework Side:**
- Wrong exception type for capability hallucination
- Should use `ReclassificationRequiredError` not `ValueError`
- Error classification logic expects specific exception types

**Orchestrator LLM Side:**
- Too creative - invents plausible capability names
- Semantic matching: "generate questions" → "question_generator" seems logical
- Training data bias: Has seen systems with specialized generators

---

## Testing After Fix

### Expected Behavior

**Test 1: Normal user queries should work**
```
User: "read the motor position"
Expected: ✅ Successfully reads and displays position
```

**Test 2: No error messages after successful operations**
```
User: "read the motor position"
Expected: ✅ Response shown, NO follow-up errors
```

**Test 3: No meta-task queries in logs**
```bash
docker-compose logs -f pipelines | grep "### Task"
Expected: No output (no meta-tasks being sent)
```

### If Issues Persist

1. **Check environment variables are applied:**
   ```bash
   docker-compose config | grep ENABLE_TITLE_GENERATION
   ```
   Should show: `ENABLE_TITLE_GENERATION: 'false'`

2. **Verify container restart:**
   ```bash
   docker-compose ps
   ```
   Check that open-webui container was recently restarted

3. **Check OpenWebUI logs:**
   ```bash
   docker-compose logs open-webui | tail -50
   ```
   Look for any errors or warnings

4. **Hard rebuild if needed:**
   ```bash
   docker-compose down
   docker-compose build --no-cache open-webui
   docker-compose up -d
   ```

---

## Additional Notes

### Template System Consideration

Since your `docker-compose.yml.j2` uses Jinja2 templating:
- The environment variables need to be in the `.j2` template file
- If you regenerate from templates, manual edits to `docker-compose.yml` will be lost
- Best practice: Add these settings to your template source

### Alternative: OpenWebUI UI Settings

You can also disable these features through OpenWebUI's web interface:
1. Log in as admin
2. Go to **Settings** → **Interface**
3. Disable:
   - Auto-generate chat titles
   - Auto-generate tags
   - Show suggested follow-up questions

**However**, environment variables are preferred because:
- ✅ Persistent across container restarts
- ✅ Version-controlled in your repository
- ✅ Apply to all users automatically

---

## GitHub Issue Template

If you want to report the framework bug to the Osprey repository:

**Title:** Orchestrator hallucination raises wrong exception type, preventing reclassification recovery

**Description:**

When the orchestrator detects hallucinated (non-existent) capabilities during execution plan validation, it raises a generic `ValueError` instead of `ReclassificationRequiredError`. This causes the error to be classified as `CRITICAL` instead of `RECLASSIFICATION`, preventing the system from properly recovering through task reclassification.

**Root Cause:**
- File: `src/osprey/infrastructure/orchestration_node.py`
- Function: `_validate_and_fix_execution_plan()` (around line 100)
- Issue: Raises `ValueError` when `ReclassificationRequiredError` is expected

**Expected Behavior:**
When orchestrator hallucinates capabilities, the system should:
1. Detect hallucination ✓
2. Raise `ReclassificationRequiredError` ✗
3. Router classifies as `RECLASSIFICATION` severity ✗
4. Route to classifier for capability reclassification ✗

**Actual Behavior:**
Currently:
1. Detects hallucination ✓
2. Raises `ValueError` ✗
3. Router classifies as `CRITICAL` severity ✗
4. Routes to error node instead of classifier ✗

**Fix:**
```python
if hallucinated_capabilities:
    error_msg = (
        f"Orchestrator hallucinated non-existent capabilities: {hallucinated_capabilities}. "
        f"Available capabilities: {registry.get_stats()['capability_names']}"
    )
    logger.error(error_msg)
    raise ReclassificationRequiredError(error_msg)  # ✅ Correct exception type
```

**Impact:**
- Severity: Medium - prevents automatic recovery from orchestrator hallucination errors
- Affects: All users experiencing orchestrator hallucination scenarios
- Workaround: None - system fails instead of recovering

---

## Contact & Support

- **Osprey Framework**: https://github.com/als-apg/osprey
- **Your Agent**: https://github.com/mlexchange/mlex_bolt_agent
- **OpenWebUI Docs**: https://docs.openwebui.com

---

*Document created: 2025-11-06*
*Issue first observed: After recent OpenWebUI upgrade*
*Solution verified: Environment variable approach*