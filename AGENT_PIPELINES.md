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

---

## Form Submission Pipeline

**Purpose**: Fill multiple input fields and submit a form (e.g., login, registration, search).

**Complexity**: Medium (10-15 tool calls, 5-10 seconds)

**Use Cases**:
- Login forms (username + password)
- Registration forms (multiple fields)
- Search with filters
- Settings configuration

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: AI Agent receives task                              │
│ Input: "Log in with username 'user@email.com' and password" │
│ Prompt: Main System Prompt                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Ensure Correct App is Running                       │
│ Tool: current_top_application_info()                        │
│ Verifies: Correct app is in foreground                      │
│ Alternative: start_application_by_id("com.example.app")     │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Dump Window Hierarchy                               │
│ Tool: dump_window_hierarchy(compressed=true)                │
│ Output: Complete UI structure                               │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Fill First Field (Username)                         │
│ Tool: set_text_by_resource_id(                              │
│         resource_id="com.app:id/username",                   │
│         text="user@email.com")                               │
│ Output: "true"                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Wait for Page Load (if needed)                      │
│ AI adds small delay between operations                      │
│ Per main system prompt: "Each operation should have         │
│ a certain interval; otherwise, the page may not be          │
│ fully loaded."                                               │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 6: Fill Second Field (Password)                        │
│ Tool: set_text_by_resource_id(                              │
│         resource_id="com.app:id/password",                   │
│         text="SecurePass123")                                │
│ Output: "true"                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 7: Submit Form                                          │
│ Tool: click_by_text(text="Login")                           │
│ OR: click_by_resource_id(resource_id="com.app:id/login_btn")│
│ OR: press_key_code(key_code=66)  # KEYCODE_ENTER            │
│ Output: "true"                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 8: Wait for Navigation                                 │
│ AI waits for screen transition                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 9: Verify Success                                       │
│ Tool: dump_window_hierarchy()                               │
│ OR: current_top_application_info()                          │
│ OR: get_last_toast()                                         │
│ Checks: New screen, success message, or error toast         │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Request → Main Prompt → current_top_application_info()
     ↓                                    ↓
     ↓                           [App Info JSON]
     ↓                                    ↓
     ↓                      dump_window_hierarchy()
     ↓                                    ↓
     ↓                           [UI Hierarchy]
     ↓                                    ↓
     ↓                    AI Identifies All Fields
     ↓                                    ↓
     ├──→ set_text_by_resource_id(username) → [true]
     ↓                                    ↓
     ├──→ set_text_by_resource_id(password) → [true]
     ↓                                    ↓
     └──→ click_by_text("Login")        → [true]
                                          ↓
                              get_last_toast() / dump_window_hierarchy()
                                          ↓
                                  [Success/Error Verification]
```

### Error Handling

The AI agent checks for errors at multiple points:

1. **After each input**: Verify `true` response
2. **After submit**: Check for error toast messages
3. **After navigation**: Verify expected screen appears

If error detected:
```
get_last_toast() → Check for error message
  ↓
If error found → Report to user or retry
  ↓
If no error but wrong screen → dump_window_hierarchy() and analyze
```

---

## Conditional Branching Pipeline

**Purpose**: Make decisions based on screen state and execute different actions accordingly.

**Complexity**: Medium (8-20 tool calls, 5-15 seconds)

**Use Cases**:
- Handling optional popups/dialogs
- Adapting to different app states
- Checking permissions before granting
- Conditional navigation based on content

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: AI Agent receives task                              │
│ Input: "Open Settings and enable notifications if disabled" │
│ Prompt: Main System Prompt                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Navigate to Target                                  │
│ Tool: start_application_by_id("com.android.settings")       │
│ Output: "true"                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Dump Window Hierarchy                               │
│ Tool: dump_window_hierarchy(compressed=true)                │
│ Output: Complete UI structure                               │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: AI Analyzes Current State                           │
│ Checks JSON hierarchy for:                                  │
│ - Is popup/dialog visible?                                  │
│ - Is setting already enabled?                               │
│ - What elements are available?                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
         ┌────────────┴────────────┐
         │  Decision Point         │
         └────┬────────────────┬───┘
              │                │
    [Popup Found]        [No Popup]
              │                │
              ▼                ▼
    ┌─────────────────┐  ┌─────────────────┐
    │ Branch A:       │  │ Branch B:       │
    │ Dismiss Popup   │  │ Continue        │
    └─────────────────┘  └─────────────────┘
              │                │
              │   ┌────────────┘
              │   │
              ▼   ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Execute Branch A Action (if popup)                  │
│ Tool: click_by_text("Dismiss")                              │
│ OR: click_by_text("OK")                                     │
│ OR: press_key_code(key_code=4)  # KEYCODE_BACK              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 6: Navigate to Notifications Settings                  │
│ Tool: click_by_text("Notifications")                        │
│ Output: "true"                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 7: Dump Updated Hierarchy                              │
│ Tool: dump_window_hierarchy(compressed=true)                │
│ Output: Notifications screen structure                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 8: Check Current Toggle State                          │
│ AI analyzes hierarchy for toggle element:                   │
│ - Looks for Switch/ToggleButton                             │
│ - Checks "checked" attribute in JSON                        │
│ - Determines if already enabled                             │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
         ┌────────────┴────────────┐
         │  Decision Point         │
         └────┬────────────────┬───┘
              │                │
    [Already Enabled]   [Currently Disabled]
              │                │
              ▼                ▼
    ┌─────────────────┐  ┌─────────────────┐
    │ Report:         │  │ Action:         │
    │ "Already ON"    │  │ Enable Toggle   │
    └─────────────────┘  └─────────────────┘
                               │
                               ▼
                    ┌─────────────────────────┐
                    │ Step 9: Toggle Switch   │
                    │ Tool: click_by_resource_│
                    │   id("android:id/switch")│
                    │ Output: "true"          │
                    └─────────┬───────────────┘
                              │
                              ▼
                    ┌─────────────────────────┐
                    │ Step 10: Verify Change  │
                    │ Tool: dump_window_      │
                    │   hierarchy()            │
                    │ Checks: "checked"="true"│
                    └─────────────────────────┘
```

### Data Flow with Branching

```
User Request → Main Prompt → start_application_by_id()
     ↓                                    ↓
     ↓                               [true]
     ↓                                    ↓
     ↓                      dump_window_hierarchy()
     ↓                                    ↓
     ↓                           [UI Hierarchy]
     ↓                                    ↓
     ↓                        AI Analyzes State
     ↓                                    ↓
     ↓               ┌────────────────────┴────────────────┐
     ↓               ▼                                     ▼
     ↓        [Popup Found]                        [No Popup]
     ↓               ↓                                     ↓
     ├──→ click_by_text("Dismiss")                     [Skip]
     ↓               ↓                                     ↓
     ↓               └─────────────┬─────────────────────┘
     ↓                             ↓
     ├──→ click_by_text("Notifications")
     ↓                             ↓
     ↓               dump_window_hierarchy()
     ↓                             ↓
     ↓                   [Toggle State Check]
     ↓                             ↓
     ↓               ┌─────────────┴──────────────┐
     ↓               ▼                            ▼
     ↓        [checked=true]               [checked=false]
     ↓               ↓                            ↓
     └──→    Report Success          click_by_resource_id()
                                                  ↓
                                            [true]
                                                  ↓
                                          Verify Success
```

### Conditional Patterns

**Pattern 1: IF-THEN**
```
IF element_exists("Popup") THEN
    click_by_text("Dismiss")
END IF
Continue with main task
```

**Pattern 2: IF-THEN-ELSE**
```
IF toggle_state == "checked" THEN
    Report "Already enabled"
ELSE
    click_by_resource_id(toggle_id)
    Verify state changed
END IF
```

**Pattern 3: Multiple Conditions**
```
IF permission_granted("CAMERA") THEN
    Proceed with camera task
ELSE IF permission_dialog_visible() THEN
    click_by_text("Allow")
    Wait and verify
ELSE
    Request permission first
    grant_application_permission("CAMERA")
END IF
```

