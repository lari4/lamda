# Agent Pipelines Documentation

This document describes all possible agent workflow pipelines in the Lambda Android Automation system. Each pipeline shows the sequence of operations, data flow between steps, and the prompts/tools used at each stage.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Basic Element Click Pipeline](#basic-element-click-pipeline)
3. [Text Input Pipeline](#text-input-pipeline)
4. [Form Submission Pipeline](#form-submission-pipeline)
5. [Conditional Branching Pipeline](#conditional-branching-pipeline)
6. [List/Scroll Processing Pipeline](#list-scroll-processing-pipeline)
7. [Retry/Error Recovery Pipeline](#retry-error-recovery-pipeline)
8. [Multi-Step Sequence Pipeline](#multi-step-sequence-pipeline)
9. [Real-Time Adaptive Waiting Pipeline](#real-time-adaptive-waiting-pipeline)
10. [Data Extraction Pipeline](#data-extraction-pipeline)
11. [State Machine Workflow Pipeline](#state-machine-workflow-pipeline)

---

## Architecture Overview

The Lambda MCP system uses a 5-layer architecture for AI-driven Android automation:

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Agent (Claude, etc.)                  │
│  - Receives main system prompt                              │
│  - Plans automation steps                                   │
│  - Calls MCP tools based on tool descriptions               │
└─────────────────────┬───────────────────────────────────────┘
                      │ MCP Protocol (JSON-RPC)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                      MCP Server                             │
│  - Routes tool calls to extensions                          │
│  - Manages sessions and contexts                            │
│  - Handles prompts, tools, and resources                    │
└─────────────────────┬───────────────────────────────────────┘
                      │ Python Method Calls
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              MCP Extensions (BaseMcpExtension)              │
│  - FireRpaMcpExtension (main automation)                    │
│  - SmsMcpExtension (SMS database access)                    │
│  - ExampleMcpExtension (utilities)                          │
└─────────────────────┬───────────────────────────────────────┘
                      │ gRPC Calls (Protobuf)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                   Device Class (gRPC Client)                │
│  - Stub caching                                             │
│  - Request serialization                                    │
│  - 3 Interceptors: Logging, Session, Exceptions             │
└─────────────────────┬───────────────────────────────────────┘
                      │ gRPC over Network
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              FIRERPA Service (Android Device)               │
│  - UI Automator integration                                 │
│  - Direct Android API access                                │
│  - Executes commands on physical/virtual device             │
└─────────────────────────────────────────────────────────────┘
```

### Key Principles

1. **Layout-Based Identification**: Always prefer using UI hierarchy over coordinates
2. **Non-Repeating Criteria**: Use unique selectors (avoid duplicate resource IDs)
3. **Proper Timing**: Add intervals between operations for page loading
4. **No Screenshot Detection**: Never use visual coordinate detection

---

## Basic Element Click Pipeline

**Purpose**: Find and click a single UI element on the screen.

**Complexity**: Simple (3-5 tool calls, 1-2 seconds)

**Use Cases**:
- Clicking buttons
- Selecting menu items
- Activating toggles

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: AI Agent receives task                              │
│ Input: User request "Click the Login button"               │
│ Prompt: Main System Prompt (Android Automation Guidelines) │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Dump Window Hierarchy                               │
│ Tool: dump_window_hierarchy(compressed=true)                │
│ Prompt: "Dumps android window's layout hierarchy as JSON"  │
│ Output: JSON with all UI elements, their properties         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: AI Analyzes Hierarchy                               │
│ AI parses JSON to find element matching "Login"            │
│ Checks: text, description, resourceId                       │
│ Selects best unique identifier (avoids duplicates)          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Execute Click                                        │
│ Tool: click_by_text(text="Login")                           │
│ OR: click_by_resource_id(resource_id="com.app:id/login_btn")│
│ Prompt: "Use full text matching to click on an element"    │
│ Output: "true" (success) or "false" (element not found)     │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Verify Result (Optional)                            │
│ Tool: dump_window_hierarchy() or current_top_application()  │
│ Confirms: Screen changed or expected result occurred        │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Request → Main Prompt → dump_window_hierarchy()
     ↓                              ↓
     ↓                         [JSON Hierarchy]
     ↓                              ↓
     ↓                    AI Analysis & Selection
     ↓                              ↓
     └──────────────→ click_by_text("Login")
                                    ↓
                              [true/false]
                                    ↓
                           Success/Failure Report
```

### Selector Priority

1. **Unique resource ID** (if not duplicated)
2. **Exact text match** (for visible text)
3. **Text contains** (for partial matching)
4. **Content description** (for accessibility labels)
5. **Text regex** (for pattern matching)
6. **Coordinates** (last resort, discouraged)

---

## Text Input Pipeline

**Purpose**: Input text into an input field on the screen.

**Complexity**: Simple (4-6 tool calls, 2-3 seconds)

**Use Cases**:
- Filling login forms
- Entering search queries
- Setting configuration values

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: AI Agent receives task                              │
│ Input: User request "Enter username 'john@email.com'"      │
│ Prompt: Main System Prompt                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Dump Window Hierarchy                               │
│ Tool: dump_window_hierarchy(compressed=true)                │
│ Output: JSON with all UI elements                           │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: AI Identifies Input Field                           │
│ Searches for: EditText, input fields with relevant labels   │
│ Looks for: hint text, nearby labels, resourceId             │
│ Avoids: Duplicate resource IDs                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Click Input Field (if needed)                       │
│ Tool: click_by_resource_id() or click_by_text()             │
│ Purpose: Focus the input field                              │
│ Output: "true" if successful                                │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Set Text                                             │
│ Tool: set_text_by_resource_id(resource_id="...",            │
│                                text="john@email.com")        │
│ OR: set_text_by_class_name(class_name="EditText", text="...")│
│ Prompt: "Use resourceId to input text into input element"  │
│ Output: "true" if successful                                │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 6: Verify Input (Optional)                             │
│ Tool: dump_window_hierarchy()                               │
│ Checks: Text appears in the field                           │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Request + Text → Main Prompt → dump_window_hierarchy()
     ↓                                       ↓
     ↓                              [JSON Hierarchy]
     ↓                                       ↓
     ↓                          AI Identifies Input Field
     ↓                                       ↓
     ↓                        click_by_resource_id() [Optional]
     ↓                                       ↓
     └──────────────→ set_text_by_resource_id(text="...")
                                             ↓
                                       [true/false]
                                             ↓
                                    Success Confirmation
```

### Alternative Approaches

**Approach 1: Resource ID** (Preferred when unique)
```
set_text_by_resource_id(
    resource_id="com.app:id/username_input",
    text="john@email.com"
)
```

**Approach 2: Class Name** (When resource ID is duplicate)
```
set_text_by_class_name(
    class_name="android.widget.EditText",
    text="john@email.com"
)
```

**Approach 3: Click then Input** (For complex forms)
```
1. click_by_text("Username")
2. set_text_by_class_name(class_name="EditText", text="john@email.com")
```

