# Complete Timing Analysis: Profiler vs Logs

## Overview
This document explains the discrepancy between Python profiler timing (0.043s) and actual wall clock timing (24.21s) for the classification node.

---

## Wall Clock Time (From Logs)

**Total Pipeline Time: 43 seconds**

```
├─ Task Extraction:      2.13s  (5%)
├─ Classification:      24.21s  (56%) 🔥 BOTTLENECK
├─ Orchestration:       10.59s  (25%) 🔥 BOTTLENECK  
├─ Motor Read:           ~2s    (5%)
└─ Response:             ~3s    (7%)
```

---

## Classification Timeline (From Logs)

**Start:** 10:53:06  
**End:** 10:53:30  
**Duration:** 24.21 seconds

### Detailed Capability Checks:
```
10:53:06 - Classifier starts
10:53:08 - Capability 'memory' >>> False
10:53:10 - Capability 'time_range_parsing' >>> False  
10:53:17 - Capability 'python' >>> True
10:53:19 - Capability 'motor_position_read' >>> True
10:53:21 - Capability 'motor_position_set' >>> False
10:53:22 - Capability 'detector_image_capture' >>> False
10:53:24 - Capability 'photogrammetry_scan_execute' >>> False
10:53:26 - Capability 'reconstruct_object' >>> False
10:53:28 - Capability 'ply_quality_assessment' >>> False
10:53:30 - Capability 'display_object' >>> False
10:53:30 - "Completed... in 24.21s"
```

**Analysis:** Each capability check takes ~2 seconds (LLM API call)  
**Total checks:** 10 capabilities × ~2 seconds = ~20-24 seconds

---

## Python Profiler Time (cProfile)

**Total Profiled Time: 71.5 seconds**

```
├─ Actual Python execution: ~5 seconds
├─ Waiting (kqueue/select): 66.7s (network I/O)
└─ Key functions:
   • motor_position_read.execute:  2.356s (API call)
   • bolt_api.get_current_angle:   2.355s (HTTP request)
   • classification.execute:       0.043s (Python code only)
   • orchestration.execute:        0.039s (Python code only)
```

### Profiler Output:
```python
ncalls  tottime  percall  cumtime  percall filename:lineno(function)
    11    0.000    0.000    0.043    0.004 classification_node.py:136(execute)
```

---

## The Key Insight

### Why the Massive Difference?

**Python profiler (cProfile) measures:**
- CPU time executing Python code
- Only the time your Python functions are actively running

**Logs measure:**
- Wall clock time (real elapsed time)
- Includes all waiting time for external resources

**LLM API calls:**
- Spend most time waiting for network responses
- Very little Python execution time
- Network I/O appears as `select.kqueue` waiting time

### The Breakdown:
```
classification_node.execute() actual work:     0.043s (Python)
Waiting for 10 LLM API responses:            24.170s (Network)
                                            ─────────
Total wall clock time:                       24.213s
```

---

## Conclusion

The profiler shows **0.043s** but logs show **24.21s** because:

1. **0.043s** = Pure Python code execution time in the classification node
2. **24.17s** = Network waiting time for LLM API responses
3. **24.21s** = Total wall clock time (0.043 + 24.17)

The difference (~24 seconds) is entirely network I/O waiting time, which doesn't count as "Python execution time" in cProfile but does count in real-world elapsed time.

---

## Bottleneck Summary

**Primary Bottlenecks:**
1. **Classification (24.21s):** 10 sequential LLM API calls
2. **Orchestration (10.59s):** Additional LLM processing

**Total LLM waiting time:** ~35 seconds out of 43 seconds (81% of pipeline)

**Recommendation:** Consider parallelizing capability checks or implementing caching for common task patterns.


## Logs 
```
================================================================================
BOLT INTERACTIVE PROFILING SESSION
Date: October 13, 2025
Command: read the motor position
================================================================================

SESSION INITIALIZATION
--------------------------------------------------------------------------------
(alpha) xiaoyachong@Mac alpha_berkeley % ./profile_bolt_interactive.sh
🔍 Starting BOLT profiling...
Commands to try:
  - What's the current position?
  - Move to 90 degrees
  - Take an image
  - exit (to finish)

Using default value for 'execution_control.limits.max_planning_attempts' = 2


    ╔═════════════════════════════════════════════════════════════════╗
    ║                                                                 ║
    ║                                                                 ║
    ║  ░█████╗░██╗░░░░░██████╗░██╗░░██╗░█████╗░  ░█████╗░██╗░░░░░██╗  ║
    ║  ██╔══██╗██║░░░░░██╔══██╗██║░░██║██╔══██╗  ██╔══██╗██║░░░░░██║  ║  
    ║  ███████║██║░░░░░██████╔╝███████║███████║  ██║░░╚═╝██║░░░░░██║  ║
    ║  ██╔══██║██║░░░░░██╔═══╝░██╔══██║██╔══██║  ██║░░██╗██║░░░░░██║  ║
    ║  ██║░░██║███████╗██║░░░░░██║░░██║██║░░██║  ╚█████╔╝███████╗██║  ║
    ║  ╚═╝░░╚═╝╚══════╝╚═╝░░░░░╚═╝░░╚═╝╚═╝░░╚═╝  ░╚════╝░╚══════╝╚═╝  ║
    ║                                                                 ║
    ║                                                                 ║
    ║     Command Line Interface for the Alpha Berkeley Framework     ║
    ╚═════════════════════════════════════════════════════════════════╝


FRAMEWORK INITIALIZATION
--------------------------------------------------------------------------------
💡 Type 'bye' or 'end' to exit

🔄 Initializing configuration...
🔄 Initializing framework...

[10/13/25 10:52:53] INFO     Registry: Loaded application registry: bolt
                    INFO     Registry: Built merged registry with 1 applications
                    INFO     Registry: Initializing registry system...
                    INFO     Registry: Registered 10 context classes
                    INFO     Memory manager initialized with directory:
                             /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/_agent_data/user_memory
                    INFO     Registry: Registered 1 data sources
                    INFO     Registry: Registered 5 core nodes
                    INFO     Registry: Registered 1 services
                    INFO     Registry: Registered 12 capabilities
                    INFO     Registry: Successfully loaded 9/9 custom prompt builders for framework_defaults
                    INFO     Registry: Set default framework prompt provider to: framework_defaults
                    INFO     Registry: Registered 1 framework prompt providers
                    INFO     Registry: Registry initialization complete!
                                Components loaded:
                                   • 12 capabilities: memory, time_range_parsing, python, respond, clarify,
                             motor_position_read, motor_position_set, detector_image_capture,
                             photogrammetry_scan_execute, reconstruct_object, ply_quality_assessment, display_object
                                   • 17 nodes (including 5 core infrastructure)
                                   • 10 context types: MEMORY_CONTEXT, TIME_RANGE, PYTHON_RESULTS, MOTOR_POSITION,
                             MOTOR_MOVEMENT, DETECTOR_IMAGE, PHOTOGRAMMETRY_SCAN, RECONSTRUCT_OBJECT,
                             PLY_QUALITY_ASSESSMENT, DISPLAY_OBJECT
                                   • 1 data sources: core_user_memory
                                   • 1 services: python_executor
                    INFO     Registry: Registry export saved to:
                             /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/_agent_data/registry_exports/registry_export.json
                    INFO     Registry: Export contains: 12 capabilities, 10 context types
                    INFO     Builder: Creating framework graph using registry components
                    INFO     Builder: Using provided InMemorySaver checkpointer
                    INFO     Builder: Building graph with 17 nodes from registry
                    INFO     ✅ Builder: Successfully created async framework graph with 17 nodes and InMemorySaver
                             checkpointing enabled (R&D mode)
                    INFO     Gateway: Gateway initialized

✅ Framework initialized! Thread ID: cli_session_423f97fe
  • Use ↑/↓ arrow keys to navigate command history
  • Use ←/→ arrow keys to edit current line
  • Press Ctrl+L to clear screen
  • Type 'bye' or 'end' to exit, or press Ctrl+C


COMMAND EXECUTION: "read the motor position"
================================================================================

USER INPUT
--------------------------------------------------------------------------------
👤 You: read the motor position
🔄 Processing: read the motor position


TASK PROCESSING PIPELINE
--------------------------------------------------------------------------------

[10/13/25 10:53:04] INFO     Gateway: Processing message: 'read the motor position...'
                    INFO     Gateway: Processing as new conversation turn
                    INFO     Gateway: DEBUG: Fresh state created with 0 execution records
                    INFO     Gateway: Created fresh state for new conversation turn
🔄 Starting new conversation turn (execution_step_results: 0 records)...


STEP 1: ROUTING TO TASK EXTRACTION
--------------------------------------------------------------------------------
                    INFO     Router: No current task extracted, routing to task extraction
                    INFO     Task_Extraction: Starting Task Extraction and Processing
                    INFO     Registered data source: core_user_memory
                    INFO     Loaded 1 data sources from registry
                    INFO     No data sources available for current context
                    INFO     Task_Extraction: Retrieved data from 0 sources
🔄 Extracting actionable task from conversation

[10/13/25 10:53:06] INFO     Task_Extraction:  * Extracted: 'Read the motor position...'
                    INFO     Task_Extraction:  * Builds on previous context: False
                    INFO     Task_Extraction:  * Uses memory context: False
                    INFO     ✅ Task_Extraction: Completed Task Extraction and Processing in 2.13s
🔄 Task extraction completed


STEP 2: TASK CLASSIFICATION
--------------------------------------------------------------------------------
                    INFO     Router: No active capabilities, routing to classifier
                    INFO     Classifier: Starting Task Classification and Capability Selection
                    INFO     Classifier: Analyzing task requirements...
                    INFO     Classifier: Classifying task: Read the motor position
🔄 Analyzing task requirements...

[10/13/25 10:53:08] INFO     Classifier:  >>> Capability 'memory' >>> False
[10/13/25 10:53:10] INFO     Classifier:  >>> Capability 'time_range_parsing' >>> False
[10/13/25 10:53:17] INFO     Classifier:  >>> Capability 'python' >>> True
[10/13/25 10:53:19] INFO     Classifier:  >>> Capability 'motor_position_read' >>> True
[10/13/25 10:53:21] INFO     Classifier:  >>> Capability 'motor_position_set' >>> False
[10/13/25 10:53:22] INFO     Classifier:  >>> Capability 'detector_image_capture' >>> False
[10/13/25 10:53:24] INFO     Classifier:  >>> Capability 'photogrammetry_scan_execute' >>> False
[10/13/25 10:53:26] INFO     Classifier:  >>> Capability 'reconstruct_object' >>> False
[10/13/25 10:53:28] INFO     Classifier:  >>> Capability 'ply_quality_assessment' >>> False
[10/13/25 10:53:30] INFO     Classifier:  >>> Capability 'display_object' >>> False

                    INFO     Classifier: 4 capabilities required: ['respond', 'clarify', 'python',
                             'motor_position_read']
                    INFO     Classifier: Classification completed with 4 active capabilities
                    INFO     Classifier: Classification completed
                    INFO     ✅ Classifier: Completed Task Classification and Capability Selection in 24.21s
🔄 Task classification complete


STEP 3: ORCHESTRATION & EXECUTION PLANNING
--------------------------------------------------------------------------------
                    INFO     Router: No execution plan, routing to orchestrator
                    INFO     Orchestrator: Starting Execution Planning and Orchestration
                    INFO     Orchestrator: Planning for task: Read the motor position
                    INFO     Orchestrator: Available capabilities: ['respond', 'clarify', 'python',
                             'motor_position_read']
                    INFO     Orchestrator: Creating orchestrator prompt for task: "Read the motor position..."
                    INFO     Orchestrator: Active capabilities: ['respond', 'clarify', 'python', 'motor_position_read']
                    INFO     Orchestrator: Constructed orchestrator instructions using:
                    INFO     Orchestrator:  - 4 capabilities
                    INFO     Orchestrator:  - 7 structured examples
                    INFO     Orchestrator:  - 0 context types
                    INFO     Orchestrator: Creating execution plan with orchestrator LLM
🔄 Generating execution plan...

[10/13/25 10:53:41] INFO     Orchestrator: Orchestrator LLM execution time: 10.55 seconds
                    INFO     Orchestrator: ==================================================
                    INFO     Orchestrator:  << Step 1
                    INFO     Orchestrator:  << ├───── id: 'current_motor_position'
                    INFO     Orchestrator:  << ├─── node: 'motor_position_read'
                    INFO     Orchestrator:  << ├─── task: 'Read the current position of the sample rotation motor to
                             determine its angle in degrees'
                    INFO     Orchestrator:  << └─ inputs: '[]'
                    INFO     Orchestrator:  << Step 2
                    INFO     Orchestrator:  << ├───── id: 'motor_position_response'
                    INFO     Orchestrator:  << ├─── node: 'respond'
                    INFO     Orchestrator:  << ├─── task: 'Provide the user with the current motor position information
                             including motor ID, angle in degrees, and timestamp'
                    INFO     Orchestrator:  << └─ inputs: '[{'MOTOR_POSITION': 'current_motor_position'}]'
                    INFO     Orchestrator: ==================================================
                    INFO     ✅ Orchestrator: Final execution plan ready with 2 steps
                    INFO     Orchestrator: Planning mode not enabled - proceeding with normal execution
                    INFO     Orchestrator: Orchestration processing completed
                    INFO     ✅ Orchestrator: Completed Execution Planning and Orchestration in 10.59s
🔄 Execution plan created


STEP 4: EXECUTE PLAN - STEP 1 (motor_position_read)
--------------------------------------------------------------------------------
                    INFO     Router: Executing step 1/2 - capability: motor_position_read
                    INFO     Motor_Position_Read: Executing capability: motor_position_read
[10/13/25 10:53:43] INFO     Motor_Position_Read: State updates: step 1
🔄 Executing motor_position_read... (10%)
🔄 Reading motor position...
🔄 Reading position from DMC01:A...
🔄 Motor position read: DMC01:A at 49.99921875°


STEP 5: EXECUTE PLAN - STEP 2 (respond)
--------------------------------------------------------------------------------
                    INFO     Router: Executing step 2/2 - capability: respond
                    INFO     Respond: Executing capability: respond
                    INFO     Message_Generator: Using technical response mode (context type: specific_context)
🔄 Executing respond... (10%)
🔄 Gathering information for response...
🔄 Generating response...

[10/13/25 10:53:45] INFO     Respond: Generated response for: 'Provide the user with the current motor position
                             information including motor ID, angle in degrees, and timestamp'
[10/13/25 10:53:46] INFO     Respond: State updates: step 2
🔄 Response generated


EXECUTION COMPLETE
--------------------------------------------------------------------------------
📊 Execution completed (execution_step_results: 2 records)

🤖 The current position of Motor DMC01:A is 49.99921875°. This data was retrieved from Tiled data after execution through the queue server on 2025-10-13 at 10:53.

👤 You:

👋 Goodbye!


================================================================================
PROFILING ANALYSIS RESULTS
================================================================================

📊 Analyzing results...


TOP 30 SLOWEST FUNCTIONS
--------------------------------------------------------------------------------
Mon Oct 13 10:54:03 2025    bolt_profile_20251013_105251.prof

         4942082 function calls (4728733 primitive calls) in 71.517 seconds

   Ordered by: cumulative time
   List reduced from 14326 to 30 due to restriction <30>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
   3641/1    0.063    0.000   71.530   71.530 {built-in method builtins.exec}
        1    0.000    0.000   71.530   71.530 interfaces/CLI/direct_conversation.py:1(<module>)
        1    0.000    0.000   69.728   69.728 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/asyncio/runners.py:160(run)
        3    0.000    0.000   69.727   23.242 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/asyncio/base_events.py:614(run_until_complete)
        3    0.001    0.000   69.727   23.242 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/asyncio/base_events.py:591(run_forever)
        1    0.000    0.000   69.726   69.726 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/asyncio/runners.py:86(run)
      470    0.011    0.000   69.726    0.148 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/asyncio/base_events.py:1834(_run_once)
      470    0.005    0.000   66.768    0.142 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/selectors.py:553(select)
      476   66.761    0.140   66.761    0.140 {method 'control' of 'select.kqueue' objects}
     1085    0.002    0.000    2.942    0.003 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/asyncio/events.py:78(_run)
1173/1085    0.004    0.000    2.940    0.003 {method 'run' of '_contextvars.Context' objects}
    64/49    0.000    0.000    2.473    0.050 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/langgraph/_internal/_runnable.py:406(ainvoke)
        3    0.000    0.000    2.372    0.791 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/base/decorators.py:194(langgraph_node)
        1    0.000    0.000    2.356    2.356 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_read.py:38(execute)
        1    0.001    0.001    2.355    2.355 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/bolt_api.py:83(get_current_angle)
  2601/32    0.006    0.000    1.947    0.061 <frozen importlib._bootstrap>:1167(_find_and_load)
  2551/29    0.006    0.000    1.947    0.067 <frozen importlib._bootstrap>:1122(_find_and_load_unlocked)
  2466/33    0.006    0.000    1.943    0.059 <frozen importlib._bootstrap>:666(_load_unlocked)
  2381/30    0.003    0.000    1.943    0.065 <frozen importlib._bootstrap_external>:934(exec_module)
  5379/59    0.002    0.000    1.938    0.033 <frozen importlib._bootstrap>:233(_call_with_frames_removed)
   410/99    0.000    0.000    1.643    0.017 {built-in method builtins.__import__}
2466/1154    0.002    0.000    1.296    0.001 <frozen importlib._bootstrap>:1209(_handle_fromlist)
6744/5434    0.038    0.000    1.229    0.000 {built-in method builtins.__build_class__}
        1    0.000    0.000    1.044    1.044 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/gateway.py:1(<module>)
        1    0.000    0.000    1.043    1.043 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/models/__init__.py:1(<module>)
        1    1.004    1.004    1.004    1.004 {built-in method time.sleep}
        1    0.000    0.000    0.969    0.969 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/models/factory.py:1(<module>)
        3    0.000    0.000    0.943    0.314 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/subprocess.py:505(run)
     1049    0.012    0.000    0.914    0.001 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/pydantic/_internal/_model_construction.py:80(__new__)
        3    0.000    0.000    0.909    0.303 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/subprocess.py:1161(communicate)


GATEWAY & INFRASTRUCTURE
--------------------------------------------------------------------------------
Mon Oct 13 10:54:03 2025    bolt_profile_20251013_105251.prof

         4942082 function calls (4728733 primitive calls) in 71.517 seconds

   Ordered by: cumulative time
   List reduced from 14326 to 40 due to restriction <'gateway|classification|orchestration'>
   List reduced from 40 to 20 due to restriction <20>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    1.044    1.044 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/gateway.py:1(<module>)
       11    0.000    0.000    0.043    0.004 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/classification_node.py:136(execute)
        2    0.000    0.000    0.039    0.019 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/orchestration_node.py:218(execute)
       11    0.000    0.000    0.037    0.003 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/classification_node.py:240(select_capabilities)
       20    0.000    0.000    0.036    0.002 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/classification_node.py:285(_classify_capability)
        1    0.000    0.000    0.021    0.021 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/orchestration_node.py:484(_log_execution_plan)
        1    0.000    0.000    0.008    0.008 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/gateway.py:110(process_message)
        1    0.000    0.000    0.008    0.008 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/orchestration_node.py:332(create_system_prompt)
       10    0.000    0.000    0.007    0.001 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/prompts/defaults/classification.py:57(get_system_instructions)
        1    0.000    0.000    0.004    0.004 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/gateway.py:197(_handle_new_message_flow)
        1    0.000    0.000    0.001    0.001 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/gateway.py:92(__init__)
       10    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/prompts/defaults/classification.py:22(get_instructions)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/anthropic/types/shared/gateway_timeout_error.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/anthropic/types/beta_gateway_timeout_error.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/gateway.py:249(_has_pending_interrupts)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/prompts/defaults/classification.py:1(<module>)
       10    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/prompts/defaults/classification.py:30(_get_dynamic_context)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/orchestration_node.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/infrastructure/classification_node.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/framework/prompts/defaults/classification.py:9(DefaultClassificationPromptBuilder)


BOLT CAPABILITIES
--------------------------------------------------------------------------------
Mon Oct 13 10:54:03 2025    bolt_profile_20251013_105251.prof

         4942082 function calls (4728733 primitive calls) in 71.517 seconds

   Ordered by: cumulative time
   List reduced from 14326 to 24 due to restriction <'bolt/capabilities'>
   List reduced from 24 to 20 due to restriction <20>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    2.356    2.356 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_read.py:38(execute)
        1    0.000    0.000    0.003    0.003 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/__init__.py:1(<module>)
        1    0.000    0.000    0.002    0.002 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_read.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_set.py:176(_create_classifier_guide)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/detector_image_capture.py:147(_create_classifier_guide)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_set.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/photogrammetry_scan_execute.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/reconstruct_object.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/detector_image_capture.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/display_object.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/ply_quality_assessment.py:1(<module>)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/photogrammetry_scan_execute.py:236(_create_classifier_guide)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/reconstruct_object.py:181(_create_classifier_guide)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_read.py:149(_create_classifier_guide)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/ply_quality_assessment.py:177(_create_classifier_guide)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/display_object.py:163(_create_classifier_guide)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_read.py:108(_create_orchestrator_guide)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_set.py:28(MotorPositionSetCapability)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/photogrammetry_scan_execute.py:28(PhotogrammetryScanExecuteCapability)
        1    0.000    0.000    0.000    0.000 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/capabilities/motor_position_read.py:28(MotorPositionReadCapability)


API CALLS (HTTP Requests)
--------------------------------------------------------------------------------
Mon Oct 13 10:54:03 2025    bolt_profile_20251013_105251.prof

         4942082 function calls (4728733 primitive calls) in 71.517 seconds

   Ordered by: cumulative time
   List reduced from 14326 to 130 due to restriction <'bolt_api|requests'>
   List reduced from 130 to 15 due to restriction <15>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.001    0.001    2.355    2.355 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/bolt_api.py:83(get_current_angle)
        1    0.000    0.000    0.060    0.060 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/google/auth/transport/requests.py:1(<module>)
        1    0.000    0.000    0.055    0.055 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/__init__.py:1(<module>)
        1    0.000    0.000    0.015    0.015 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/exceptions.py:1(<module>)
        1    0.000    0.000    0.015    0.015 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/compat.py:1(<module>)
        1    0.000    0.000    0.007    0.007 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/api.py:1(<module>)
        1    0.000    0.000    0.007    0.007 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/sessions.py:1(<module>)
        1    0.000    0.000    0.007    0.007 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/adapters.py:1(<module>)
        1    0.000    0.000    0.007    0.007 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/compat.py:18(_resolve_char_detection)
        1    0.000    0.000    0.003    0.003 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/starlette/requests.py:1(<module>)
        1    0.000    0.000    0.002    0.002 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/utils.py:1(<module>)
        1    0.000    0.000    0.002    0.002 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests_toolbelt/__init__.py:1(<module>)
        1    0.000    0.000    0.002    0.002 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/packages.py:1(<module>)
        1    0.000    0.000    0.001    0.001 /Users/xiaoyachong/Documents/3RSE_finish/17bolt/alpha_berkeley/interfaces/CLI/../../src/applications/bolt/bolt_api.py:1(<module>)
        1    0.000    0.000    0.001    0.001 /Users/xiaoyachong/anaconda3/envs/alpha/lib/python3.11/site-packages/requests/models.py:1(<module>)


================================================================================
PROFILING SUMMARY
================================================================================

✅ Profile saved: bolt_profile_20251013_105251.prof
💡 View interactively: python -m pstats bolt_profile_20251013_105251.prof


KEY FINDINGS
--------------------------------------------------------------------------------

TOTAL EXECUTION TIME: 71.517 seconds

BREAKDOWN:
├─ Network I/O (kqueue waiting):     66.761s  (93.3%)
├─ Python code execution:            ~4.8s    (6.7%)
└─ Key operations:
   • Task Extraction:                2.13s
   • Classification:                 24.21s (wall clock) / 0.043s (CPU)
   • Orchestration:                  10.59s (wall clock) / 0.039s (CPU)
   • Motor API call:                 2.36s
   • Module imports:                 ~1.9s

BOTTLENECK ANALYSIS:
1. Classification node:    24.21s wall time, 0.043s CPU time
   - 10 capability checks × ~2s per LLM API call
   - 93% of time spent waiting for network responses

2. Orchestration node:     10.59s wall time, 0.039s CPU time
   - Single LLM call for execution planning
   - Nearly all time spent in network I/O

3. Motor position read:    2.36s (HTTP API call to BOLT hardware)

TOTAL LLM WAITING TIME:    ~35 seconds (81% of total execution)
```