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

---

## List/Scroll Processing Pipeline

**Purpose**: Process items in a scrollable list, performing actions on each item or searching for specific content.

**Complexity**: High (20-50+ tool calls, 15-60 seconds)

**Use Cases**:
- Scrolling through a feed to find specific content
- Processing all items in a list
- Infinite scroll data collection
- Finding elements not initially visible

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: AI Agent receives task                              │
│ Input: "Scroll through contacts and find 'John Smith'"     │
│ Prompt: Main System Prompt                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Initialize                                           │
│ Tool: start_application_by_id("com.android.contacts")       │
│ Tool: dump_window_hierarchy()                               │
│ Output: Initial list state                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Check Current Visible Items                         │
│ AI analyzes hierarchy for list items                        │
│ Extracts: Names/content of currently visible items          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
         ┌────────────┴────────────┐
         │  Decision Point         │
         └────┬────────────────┬───┘
              │                │
    [Target Found]      [Target Not Found]
              │                │
              ▼                ▼
    ┌─────────────────┐  ┌─────────────────┐
    │ Click Target    │  │ Continue Scroll │
    │ and Exit        │  │ Loop            │
    └─────────────────┘  └─────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────┐
                │ Step 4: Get Device Info          │
                │ Tool: get_device_info()           │
                │ Output: {displayWidth, Height}    │
                │ Calculate scroll coordinates:     │
                │   fromY = height * 0.8            │
                │   toY = height * 0.2              │
                │   x = width / 2                   │
                └──────────────┬───────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────┐
                │ Step 5: Perform Swipe            │
                │ Tool: swipe(fromX=x, fromY=fromY, │
                │             toX=x, toY=toY,       │
                │             step=32)              │
                │ Output: "true"                    │
                └──────────────┬───────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────┐
                │ Step 6: Wait for Animation       │
                │ AI adds brief delay               │
                └──────────────┬───────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────┐
                │ Step 7: Dump Updated Hierarchy   │
                │ Tool: dump_window_hierarchy()     │
                │ Output: New list state            │
                └──────────────┬───────────────────┘
                               │
                               ▼
                ┌──────────────────────────────────┐
                │ Step 8: Compare with Previous    │
                │ AI checks if new items appeared   │
                │ - If no change: End of list       │
                │ - If changed: Check for target    │
                └──────────────┬───────────────────┘
                               │
                               ▼
         ┌─────────────────────┴────────────────────┐
         │  Decision Point                          │
         └────┬───────────────┬──────────────────┬──┘
              │               │                  │
    [Target Found]  [No Change/End]  [Continue Scrolling]
              │               │                  │
              ▼               ▼                  ▼
    ┌─────────────┐  ┌──────────────┐  ┌────────────┐
    │ Click Item  │  │ Report       │  │ Loop Back  │
    │ Success     │  │ Not Found    │  │ to Step 5  │
    └─────────────┘  └──────────────┘  └────────────┘
```

### Data Flow

```
User Request → Main Prompt → start_application_by_id()
     ↓                                    ↓
     ↓                      dump_window_hierarchy()
     ↓                                    ↓
     ↓                         [Visible Items List]
     ↓                                    ↓
     ↓                      Check for Target Item
     ↓                                    ↓
     ↓               ┌────────────────────┴──────────────┐
     ↓               ▼                                   ▼
     ↓         [Found]                              [Not Found]
     ↓               ↓                                   ↓
     ├──→ click_by_text("John Smith")      get_device_info()
     ↓               ↓                                   ↓
     ↓          [Success]                   [Screen Dimensions]
     ↓                                                   ↓
     ↓                               swipe(x, fromY, x, toY)
     ↓                                                   ↓
     ↓                                              [true]
     ↓                                                   ↓
     ↓                               dump_window_hierarchy()
     ↓                                                   ↓
     ↓                                      [New Items List]
     ↓                                                   ↓
     ↓                                Compare with Previous
     ↓                                                   ↓
     ↓               ┌───────────────────────────────────┴───────┐
     ↓               ▼                                           ▼
     ↓         [Items Changed]                        [No Change/End]
     ↓               ↓                                           ↓
     └──────→ Loop to Check Again                      Report Not Found
```

### Scroll Loop Pattern

```python
# Pseudocode for AI logic
previous_items = []
max_iterations = 20  # Prevent infinite loops
iteration = 0

while iteration < max_iterations:
    # Get current visible items
    hierarchy = dump_window_hierarchy()
    current_items = extract_items(hierarchy)

    # Check if target found
    if "John Smith" in current_items:
        click_by_text("John Smith")
        return SUCCESS

    # Check if no new items (end of list)
    if current_items == previous_items:
        return NOT_FOUND

    # Scroll down
    device_info = get_device_info()
    swipe(
        fromX=device_info.width / 2,
        fromY=device_info.height * 0.8,
        toX=device_info.width / 2,
        toY=device_info.height * 0.2
    )

    previous_items = current_items
    iteration += 1

return NOT_FOUND  # Max iterations reached
```

---

## Retry/Error Recovery Pipeline

**Purpose**: Handle failures gracefully by retrying operations with different strategies or recovering from errors.

**Complexity**: Medium-High (10-30 tool calls, 5-20 seconds)

**Use Cases**:
- Network-dependent operations
- Handling transient UI states
- Recovering from element not found errors
- Dealing with timing issues

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: AI Agent receives task                              │
│ Input: "Click the Submit button"                            │
│ Prompt: Main System Prompt                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: First Attempt                                        │
│ Tool: dump_window_hierarchy()                               │
│ Tool: click_by_text(text="Submit")                          │
│ Output: "false" (element not found)                         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Detect Failure                                       │
│ AI recognizes: click_by_text returned "false"               │
│ Initiates retry strategy                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Strategy 1 - Wait and Retry                         │
│ Reason: Page may still be loading                           │
│ Action: AI waits briefly                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Second Attempt                                       │
│ Tool: dump_window_hierarchy()                               │
│ Tool: click_by_text(text="Submit")                          │
│ Output: "false" (still not found)                           │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 6: Strategy 2 - Alternative Selector                   │
│ AI analyzes hierarchy for similar elements:                 │
│ - Buttons with "submit" in resourceId                       │
│ - Elements with "Submit" in description                     │
│ - Partial text match "Sub"                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 7: Third Attempt with Alternative                      │
│ Tool: click_by_resource_id(                                 │
│         resource_id="com.app:id/submit_btn")                │
│ Output: "true" (success!)                                   │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 8: Verify Success                                       │
│ Tool: dump_window_hierarchy()                               │
│ OR: get_last_toast()                                         │
│ Confirms: Action completed successfully                     │
└─────────────────────────────────────────────────────────────┘
```

### Retry Strategies

**Strategy 1: Simple Retry with Delay**
```
┌──────────────┐
│ Attempt 1    │ → [Fail]
└──────┬───────┘
       │
       ▼ Wait 1 second
┌──────────────┐
│ Attempt 2    │ → [Success/Fail]
└──────────────┘
```

**Strategy 2: Alternative Selectors**
```
┌──────────────────────┐
│ click_by_text()      │ → [Fail]
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ click_by_resource_id()│ → [Fail]
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ click_by_description()│ → [Success]
└──────────────────────┘
```

**Strategy 3: Scroll into View**
```
┌──────────────────────┐
│ click_by_text()      │ → [Fail: Element not visible]
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ swipe() to scroll    │ → [Scroll down]
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ dump_hierarchy()     │ → [Get new state]
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ click_by_text()      │ → [Success]
└──────────────────────┘
```

**Strategy 4: Dismiss Blocking Elements**
```
┌──────────────────────┐
│ click_by_text()      │ → [Fail: Blocked by popup]
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ dump_hierarchy()     │ → [Detect popup]
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ click_by_text("OK")  │ → [Dismiss popup]
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ click_by_text()      │ → [Success]
└──────────────────────┘
```

### Error Recovery Flow

```
                    [Operation Attempted]
                            │
                            ▼
                    ┌───────────────┐
                    │ Check Result  │
                    └───────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              ▼                           ▼
        [Success]                    [Failure]
              │                           │
              ▼                           ▼
        ┌──────────┐         ┌────────────────────────┐
        │ Continue │         │ Analyze Failure Type   │
        │ Workflow │         └────────┬───────────────┘
        └──────────┘                  │
                            ┌─────────┴──────────┬──────────┬─────────┐
                            ▼                    ▼          ▼         ▼
                    ┌──────────────┐   ┌──────────────┐  ┌────┐  ┌────────┐
                    │ Element Not  │   │ Wrong Screen │  │App │  │Network │
                    │ Found        │   │              │  │Crash│ │Error   │
                    └──────┬───────┘   └──────┬───────┘  └─┬──┘  └───┬────┘
                           │                  │             │         │
                           ▼                  ▼             ▼         ▼
                  ┌────────────────┐  ┌────────────┐  ┌────────┐  ┌─────────┐
                  │ Try Alternative│  │ Navigate   │  │Restart │  │ Retry   │
                  │ Selector       │  │ Back       │  │App     │  │ Later   │
                  └────────┬───────┘  └─────┬──────┘  └───┬────┘  └────┬────┘
                           │                │              │            │
                           └────────────────┴──────────────┴────────────┘
                                            │
                                            ▼
                                    [Retry Operation]
                                            │
                                            ▼
                              ┌─────────────────────────┐
                              │ Max Retries Exceeded?   │
                              └─────┬────────────┬──────┘
                                    │            │
                              [No]  │            │ [Yes]
                                    ▼            ▼
                            [Try Again]   [Report Failure]
```

### Retry Decision Logic

```python
# Pseudocode for AI retry logic
max_retries = 3
retry_count = 0
strategies = [
    "exact_text",
    "partial_text",
    "resource_id",
    "description",
    "scroll_and_retry"
]

while retry_count < max_retries:
    strategy = strategies[retry_count]

    if strategy == "exact_text":
        result = click_by_text("Submit")
    elif strategy == "partial_text":
        result = click_by_text_contains("Sub")
    elif strategy == "resource_id":
        hierarchy = dump_window_hierarchy()
        resource_id = find_submit_button_id(hierarchy)
        result = click_by_resource_id(resource_id)
    elif strategy == "description":
        result = click_by_description("Submit")
    elif strategy == "scroll_and_retry":
        swipe_down()
        result = click_by_text("Submit")

    if result == "true":
        return SUCCESS

    retry_count += 1
    wait(delay=1 + retry_count)  # Exponential backoff

return FAILURE
```

---

## Multi-Step Sequence Pipeline

**Purpose**: Execute complex workflows that require multiple coordinated operations across different screens or apps.

**Complexity**: Very High (30-100+ tool calls, 30-120 seconds)

**Use Cases**:
- Complete e-commerce purchase flow
- Multi-app workflows (copy from one app, paste to another)
- Complex configuration tasks
- End-to-end testing scenarios

### Pipeline Flow (Example: E-commerce Purchase)

```
┌─────────────────────────────────────────────────────────────┐
│ Phase 1: Launch and Search                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴─────────────────┐
    │ start_application_by_id("taobao") │
    └─────────────────┬─────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ dump_window_hierarchy()                │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────┐
    │ click_by_resource_id("search_input")       │
    └─────────────────┬──────────────────────────┘
                      │
    ┌─────────────────┴────────────────────────────┐
    │ set_text_by_resource_id(text="iPhone 15")    │
    └─────────────────┬────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ press_key_code(66)  # KEYCODE_ENTER    │
    └─────────────────┬──────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Phase 2: Browse Results                                     │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ Wait for search results to load        │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ dump_window_hierarchy()                │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴────────────────────────────┐
    │ AI analyzes results, finds target product    │
    └─────────────────┬────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ click_by_text("iPhone 15 Pro Max")     │
    └─────────────────┬──────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Phase 3: Product Details                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ Wait for product page to load          │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ dump_window_hierarchy()                │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────┐
    │ Select options (color, size, etc.)         │
    ├────────────────────────────────────────────┤
    │ click_by_text("Black")                     │
    │ click_by_text("256GB")                     │
    └─────────────────┬──────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ click_by_text("Add to Cart")           │
    └─────────────────┬──────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Phase 4: Checkout                                            │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ click_by_text("Proceed to Checkout")   │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ dump_window_hierarchy()                │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴───────────────────────────┐
    │ Fill shipping information                   │
    ├─────────────────────────────────────────────┤
    │ set_text_by_resource_id("address", "...")   │
    │ set_text_by_resource_id("phone", "...")     │
    └─────────────────┬───────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ Select payment method                  │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ click_by_text("Place Order")           │
    └─────────────────┬──────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Phase 5: Verification                                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ Wait for confirmation                  │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ dump_window_hierarchy()                │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ OR: get_last_toast()                   │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴────────────────────────────┐
    │ Verify success message or order number      │
    └──────────────────────────────────────────────┘
```

### Cross-App Multi-Step Example

```
┌─────────────────────────────────────────────────────────────┐
│ Task: Copy text from SMS and paste to messaging app         │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴───────────────────────────────┐
    │ Step 1: Read SMS                                │
    ├─────────────────────────────────────────────────┤
    │ read_sms_database_by_sql(                       │
    │   "SELECT body FROM sms                         │
    │    WHERE address='12345'                        │
    │    ORDER BY date DESC LIMIT 1")                 │
    │ Output: [{"body": "Your code is: 123456"}]      │
    └─────────────────┬───────────────────────────────┘
                      │
    ┌─────────────────┴───────────────────────────────┐
    │ Step 2: Extract Code from SMS                   │
    │ AI parses: "123456"                             │
    └─────────────────┬───────────────────────────────┘
                      │
    ┌─────────────────┴───────────────────────────────┐
    │ Step 3: Set Clipboard                           │
    │ set_clipboard_text(text="123456")               │
    │ Output: "true"                                  │
    └─────────────────┬───────────────────────────────┘
                      │
    ┌─────────────────┴───────────────────────────────┐
    │ Step 4: Switch to Messaging App                 │
    │ start_application_by_id("com.whatsapp")         │
    │ Output: "true"                                  │
    └─────────────────┬───────────────────────────────┘
                      │
    ┌─────────────────┴───────────────────────────────┐
    │ Step 5: Navigate to Chat                        │
    │ dump_window_hierarchy()                         │
    │ click_by_text("Contact Name")                   │
    └─────────────────┬───────────────────────────────┘
                      │
    ┌─────────────────┴───────────────────────────────┐
    │ Step 6: Paste Code                              │
    │ click_by_resource_id("input_field")             │
    │ press_key_code(279)  # KEYCODE_PASTE            │
    │ Output: "true"                                  │
    └─────────────────┬───────────────────────────────┘
                      │
    ┌─────────────────┴───────────────────────────────┐
    │ Step 7: Send Message                            │
    │ click_by_resource_id("send_button")             │
    │ Output: "true"                                  │
    └─────────────────────────────────────────────────┘
```

---

## Real-Time Adaptive Waiting Pipeline

**Purpose**: Dynamically wait for UI changes and adapt to varying load times instead of using fixed delays.

**Complexity**: Medium (5-15 tool calls, 2-30 seconds variable)

**Use Cases**:
- Waiting for network operations to complete
- Handling variable page load times
- Waiting for animations or transitions
- Polling for specific UI changes

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Trigger Action                                       │
│ Tool: click_by_text("Load Data")                            │
│ Output: "true"                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Initialize Wait Loop                                │
│ max_wait_time = 30 seconds                                  │
│ poll_interval = 1 second                                    │
│ elapsed_time = 0                                            │
│ target_condition = "Data loaded successfully"              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────┐
    │         Wait Loop Start                     │
    └─────────────────┬───────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────┐
    │ Step 3: Check Current State                 │
    │ Tool: dump_window_hierarchy()               │
    │ Output: Current UI structure                │
    └─────────────────┬───────────────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────────────┐
    │ Step 4: Analyze Hierarchy                   │
    │ AI searches for:                            │
    │ - Loading indicators (spinners, progress)   │
    │ - Success messages                          │
    │ - Error messages                            │
    │ - Expected content                          │
    └─────────────────┬───────────────────────────┘
                      │
                      ▼
         ┌────────────┴─────────────┬──────────────┬──────────┐
         │                          │              │          │
         ▼                          ▼              ▼          ▼
   [Loading...]              [Success!]      [Error!]   [Timeout]
         │                          │              │          │
         │                          │              │          │
         ▼                          ▼              ▼          ▼
    ┌────────┐              ┌──────────┐    ┌─────────┐  ┌─────────┐
    │ Wait   │              │ Proceed  │    │ Handle  │  │ Report  │
    │ 1 sec  │              │ to Next  │    │ Error   │  │ Timeout │
    └───┬────┘              │ Step     │    └─────────┘  └─────────┘
        │                   └──────────┘
        ▼
    ┌────────────────────┐
    │ elapsed_time += 1  │
    └───┬────────────────┘
        │
        ▼
    ┌─────────────────────────┐
    │ elapsed_time > 30?      │
    └───┬─────────────────┬───┘
        │                 │
      [No]              [Yes]
        │                 │
        │                 ▼
        │         ┌──────────────┐
        │         │ Exit: Timeout│
        │         └──────────────┘
        │
        └──→ Loop Back to Step 3
```

### Adaptive Wait Patterns

**Pattern 1: Wait for Loading to Disappear**
```python
# AI Logic
while elapsed_time < max_wait:
    hierarchy = dump_window_hierarchy()

    # Check if loading indicator exists
    if "ProgressBar" not in hierarchy:
        # Loading finished
        return SUCCESS

    wait(1)
    elapsed_time += 1

return TIMEOUT
```

**Pattern 2: Wait for Specific Element to Appear**
```python
# AI Logic
while elapsed_time < max_wait:
    hierarchy = dump_window_hierarchy()

    # Check if target element visible
    if find_element(hierarchy, text="Welcome Back"):
        return SUCCESS

    wait(1)
    elapsed_time += 1

return TIMEOUT
```

**Pattern 3: Wait for Toast Message**
```python
# AI Logic
target_message = "Success"
while elapsed_time < max_wait:
    toast = get_last_toast()

    if target_message in toast:
        return SUCCESS

    wait(0.5)  # Toast checks can be more frequent
    elapsed_time += 0.5

return TIMEOUT
```

**Pattern 4: Exponential Backoff Polling**
```python
# AI Logic
wait_interval = 0.5
while elapsed_time < max_wait:
    hierarchy = dump_window_hierarchy()

    if check_condition(hierarchy):
        return SUCCESS

    wait(wait_interval)
    elapsed_time += wait_interval
    wait_interval = min(wait_interval * 1.5, 5)  # Cap at 5 seconds

return TIMEOUT
```

---

## Data Extraction Pipeline

**Purpose**: Extract structured data from Android apps or system databases for analysis or transfer.

**Complexity**: Medium-High (10-40 tool calls, 5-30 seconds)

**Use Cases**:
- Reading SMS messages
- Extracting app data
- Screen scraping
- Data migration tasks

### Pipeline Flow (Example: SMS Extraction)

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: AI Agent receives task                              │
│ Input: "Get all SMS from contact '12345' in last 7 days"   │
│ Prompt: Main System Prompt                                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Build SQL Query                                     │
│ AI constructs query based on requirements:                  │
│ - Filter by address (phone number)                          │
│ - Filter by date (last 7 days in milliseconds)             │
│ - Select relevant columns                                   │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Query SMS Database                                  │
│ Tool: read_sms_database_by_sql(                             │
│   sql="SELECT address, body, date, type                     │
│        FROM sms                                              │
│        WHERE address='12345'                                 │
│        AND date > strftime('%s', 'now', '-7 days') * 1000   │
│        ORDER BY date DESC"                                   │
│ )                                                            │
│ Output: JSON array of SMS messages                          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Parse and Structure Data                            │
│ AI receives: [                                              │
│   {                                                          │
│     "address": "12345",                                      │
│     "body": "Hello, how are you?",                          │
│     "date": 1704067200000,                                  │
│     "type": 1  // 1=received, 2=sent                        │
│   },                                                         │
│   ...                                                        │
│ ]                                                            │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Transform Data                                       │
│ AI converts timestamps, categorizes messages:                │
│ - Convert Unix timestamp to readable date                   │
│ - Separate sent vs received                                 │
│ - Count messages                                             │
│ - Extract key information                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 6: Present Results                                      │
│ AI formats output:                                           │
│ "Found 15 messages from contact 12345:                      │
│  - 8 received messages                                       │
│  - 7 sent messages                                           │
│  - Date range: 2024-01-01 to 2024-01-07                     │
│  - Latest message: 'See you tomorrow!'"                      │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Request → Main Prompt → AI Analyzes Requirements
     ↓                                    ↓
     ↓                         [Determine data source]
     ↓                                    ↓
     ↓               ┌────────────────────┴───────────────┐
     ↓               ▼                                    ▼
     ↓      [SMS Database]                      [UI Scraping]
     ↓               ↓                                    ↓
     ├──→ read_sms_database_by_sql(SQL)    dump_window_hierarchy()
     ↓               ↓                                    ↓
     ↓         [Raw JSON Data]                    [UI Hierarchy]
     ↓               ↓                                    ↓
     ↓         Parse Database          Extract Text from Elements
     ↓               ↓                                    ↓
     ↓               └─────────────┬──────────────────────┘
     ↓                             ↓
     ↓                   [Structured Data]
     ↓                             ↓
     ↓                     Transform & Format
     ↓                             ↓
     └──────────────→      [Present to User]
```

### Screen Scraping Pattern

```
┌─────────────────────────────────────────────────────────────┐
│ Task: Extract product details from screen                   │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ Step 1: Navigate to Product Page       │
    │ (Using previous pipelines)             │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ Step 2: Dump Hierarchy                 │
    │ dump_window_hierarchy()                │
    │ Output: Complete UI structure          │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────┐
    │ Step 3: Extract Data Points               │
    │ AI parses JSON to find:                   │
    │ - Product name (text="iPhone 15")         │
    │ - Price (text="$999")                     │
    │ - Rating (text="4.5 stars")               │
    │ - Availability (text="In Stock")          │
    │ - Description (from specific node)        │
    └─────────────────┬──────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────┐
    │ Step 4: Scroll for More Data (optional)   │
    │ swipe() to reveal more content             │
    │ dump_window_hierarchy() again              │
    │ Extract additional information             │
    └─────────────────┬──────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────┐
    │ Step 5: Structure Output                   │
    │ {                                          │
    │   "name": "iPhone 15",                     │
    │   "price": "$999",                         │
    │   "rating": "4.5",                         │
    │   "stock": "In Stock",                     │
    │   "description": "..."                     │
    │ }                                          │
    └────────────────────────────────────────────┘
```

---

## State Machine Workflow Pipeline

**Purpose**: Handle complex scenarios with multiple states, transitions, and conditional loops that may require returning to previous states.

**Complexity**: Very High (Variable, 40-200+ tool calls, 60+ seconds)

**Use Cases**:
- Game automation with multiple states
- Complex multi-branch workflows
- Workflows that require state persistence
- Scenarios with unpredictable transitions

### State Machine Concept

A state machine workflow tracks the current state and transitions between states based on conditions:

```
                    ┌──────────────┐
                    │  INIT STATE  │
                    └──────┬───────┘
                           │
                           ▼
              ┌────────────────────────┐
              │  Check Prerequisites   │
              └────┬──────────────┬────┘
                   │              │
         [Ready]   │              │  [Not Ready]
                   │              │
                   ▼              ▼
        ┌───────────────┐  ┌──────────────┐
        │  MAIN STATE   │  │ SETUP STATE  │
        └───────┬───────┘  └──────┬───────┘
                │                 │
                │                 └──────┐
                │                        │
                ▼                        ▼
    ┌──────────────────────┐   ┌─────────────────┐
    │  Process Data        │   │ Grant Permissions│
    └──────┬───────────────┘   └─────────┬───────┘
           │                              │
           ▼                              │
    ┌──────────────┐                     │
    │ Check Result │                     │
    └──────┬───────┘                     │
           │                              │
     ┌─────┴─────┐                       │
     │           │                       │
  [Success]  [Failure]                   │
     │           │                       │
     ▼           ▼                       ▼
┌─────────┐ ┌─────────┐         ┌──────────────┐
│ SUCCESS │ │ RETRY   │────────→│ Loop Back to │
│ STATE   │ │ STATE   │         │  MAIN STATE  │
└─────────┘ └─────────┘         └──────────────┘
```

### Pipeline Flow (Example: App Automation with Multiple States)

```
┌─────────────────────────────────────────────────────────────┐
│ STATE: INIT                                                  │
│ Initialize workflow, check device state                     │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ is_screen_on()                         │
    └─────────────────┬──────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
      [true]                    [false]
         │                         │
         ▼                         ▼
    Continue                  wake_up()
         │                         │
         │                         │
         └────────┬────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ STATE: CHECK_APP                                             │
│ Verify if target app is installed and has permissions       │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────────┐
    │ is_application_installed("com.target.app")     │
    └─────────────────┬──────────────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
      [true]                    [false]
         │                         │
         ▼                         ▼
    ┌─────────────┐         ┌──────────────┐
    │ Check Perms │         │ STATE: ERROR │
    │ STATE       │         │ Report Issue │
    └─────┬───────┘         └──────────────┘
          │
          ▼
    ┌──────────────────────────────────────┐
    │ list_application_permissions()        │
    └─────────────────┬────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
   [Has Camera]              [No Camera]
         │                         │
         ▼                         ▼
    ┌─────────────┐         ┌──────────────────┐
    │ STATE:      │         │ STATE: GRANT     │
    │ LAUNCH_APP  │         │ grant_application│
    └─────┬───────┘         │ _permission()    │
          │                 └──────┬───────────┘
          │                        │
          └────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ STATE: LAUNCH_APP                                            │
│ Start application and verify it's running                   │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────────┐
    │ start_application_by_id("com.target.app")      │
    └─────────────────┬──────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────────┐
    │ Wait for app launch (Adaptive Wait Pipeline)   │
    └─────────────────┬──────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────────────┐
    │ is_application_running_foreground()            │
    └─────────────────┬──────────────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
      [true]                    [false]
         │                         │
         ▼                         ▼
    ┌─────────────┐         ┌──────────────┐
    │ STATE:      │         │ STATE: RETRY │
    │ NAVIGATE    │         │ (max 3 times)│
    └─────────────┘         └──────┬───────┘
                                   │
                                   └──→ Back to LAUNCH_APP
                                         or ERROR state
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ STATE: NAVIGATE                                              │
│ Navigate through app to target screen                       │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ dump_window_hierarchy()                │
    └─────────────────┬──────────────────────┘
                      │
         ┌────────────┴────────────┬──────────────┐
         │                         │              │
   [Main Screen]              [Login Screen] [Unknown Screen]
         │                         │              │
         ▼                         ▼              ▼
    ┌─────────────┐         ┌──────────────┐  ┌─────────────┐
    │ STATE:      │         │ STATE: LOGIN │  │ STATE: BACK │
    │ EXECUTE     │         └──────┬───────┘  │ press_key   │
    │ TASK        │                │          │ (BACK)      │
    └─────────────┘                │          └─────┬───────┘
                                   │                │
                                   │                │
                                   └────────────────┘
                                          │
                                          └──→ Back to NAVIGATE
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ STATE: EXECUTE_TASK                                          │
│ Perform the main automation task                            │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ Execute specific workflow               │
    │ (Form Submission, Data Extraction, etc.)│
    └─────────────────┬──────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
     [Success]                 [Failure]
         │                         │
         ▼                         ▼
    ┌─────────────┐         ┌──────────────┐
    │ STATE:      │         │ STATE: RETRY │
    │ VERIFY      │         │ Or ROLLBACK  │
    └─────┬───────┘         └──────┬───────┘
          │                        │
          │                        └──→ Back to previous
          │                              state or EXECUTE
          ▼
┌─────────────────────────────────────────────────────────────┐
│ STATE: VERIFY                                                │
│ Confirm task completion and check results                   │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ dump_window_hierarchy()                │
    │ OR: get_last_toast()                   │
    │ OR: read_sms_database_by_sql()         │
    └─────────────────┬──────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
    [Verified]               [Not Verified]
         │                         │
         ▼                         ▼
    ┌─────────────┐         ┌──────────────┐
    │ STATE:      │         │ STATE: RETRY │
    │ CLEANUP     │         │ or ERROR     │
    └─────┬───────┘         └──────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│ STATE: CLEANUP                                               │
│ Clean up, close app, report results                         │
└─────────────────────┬───────────────────────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ stop_application_by_id() [optional]    │
    └─────────────────┬──────────────────────┘
                      │
    ┌─────────────────┴──────────────────────┐
    │ Report success to user                 │
    └─────────────────┬──────────────────────┘
                      │
                      ▼
                ┌──────────┐
                │   END    │
                └──────────┘
```

### State Machine Implementation Pattern

```python
# Pseudocode for AI state machine logic
class AppAutomationStateMachine:
    def __init__(self):
        self.state = "INIT"
        self.retry_count = {}
        self.max_retries = 3
        self.context = {}

    def run(self):
        while self.state != "END" and self.state != "ERROR":
            if self.state == "INIT":
                self.handle_init()
            elif self.state == "CHECK_APP":
                self.handle_check_app()
            elif self.state == "GRANT":
                self.handle_grant_permissions()
            elif self.state == "LAUNCH_APP":
                self.handle_launch_app()
            elif self.state == "NAVIGATE":
                self.handle_navigate()
            elif self.state == "EXECUTE_TASK":
                self.handle_execute_task()
            elif self.state == "VERIFY":
                self.handle_verify()
            elif self.state == "CLEANUP":
                self.handle_cleanup()
                self.state = "END"

        return self.state == "END"

    def handle_init(self):
        if is_screen_on() == "false":
            wake_up()
        self.state = "CHECK_APP"

    def handle_check_app(self):
        if is_application_installed("com.target.app") == "false":
            self.state = "ERROR"
            return

        perms = list_application_permissions("com.target.app")
        if "CAMERA" not in perms:
            self.state = "GRANT"
        else:
            self.state = "LAUNCH_APP"

    def handle_grant_permissions(self):
        result = grant_application_permission("com.target.app", "CAMERA")
        if result == "true":
            self.state = "LAUNCH_APP"
        else:
            self.state = "ERROR"

    def handle_launch_app(self):
        start_application_by_id("com.target.app")
        wait_adaptive(condition="app_launched", max_wait=10)

        if is_application_running_foreground("com.target.app") == "true":
            self.state = "NAVIGATE"
        else:
            if self.retry("LAUNCH_APP"):
                # Try again
                pass
            else:
                self.state = "ERROR"

    def handle_navigate(self):
        hierarchy = dump_window_hierarchy()
        screen_type = detect_screen_type(hierarchy)

        if screen_type == "main":
            self.state = "EXECUTE_TASK"
        elif screen_type == "login":
            perform_login()
            self.state = "NAVIGATE"  # Re-check after login
        else:
            press_key_code(4)  # Back button
            self.state = "NAVIGATE"

    def handle_execute_task(self):
        # Use other pipelines (Form Submission, etc.)
        result = execute_main_workflow()

        if result == SUCCESS:
            self.state = "VERIFY"
        else:
            if self.retry("EXECUTE_TASK"):
                # Try again
                pass
            else:
                self.state = "ERROR"

    def handle_verify(self):
        verification = verify_task_completion()

        if verification == SUCCESS:
            self.state = "CLEANUP"
        else:
            if self.retry("EXECUTE_TASK"):
                self.state = "EXECUTE_TASK"
            else:
                self.state = "ERROR"

    def handle_cleanup(self):
        # Optional: stop_application_by_id()
        report_success()

    def retry(self, state_name):
        if state_name not in self.retry_count:
            self.retry_count[state_name] = 0

        self.retry_count[state_name] += 1
        return self.retry_count[state_name] <= self.max_retries
```

---

## Summary and Best Practices

### Pipeline Complexity Levels

| Pipeline Type | Tool Calls | Duration | Complexity |
|--------------|-----------|----------|------------|
| Basic Element Click | 3-5 | 1-2s | Simple |
| Text Input | 4-6 | 2-3s | Simple |
| Form Submission | 10-15 | 5-10s | Medium |
| Conditional Branching | 8-20 | 5-15s | Medium |
| List/Scroll Processing | 20-50+ | 15-60s | High |
| Retry/Error Recovery | 10-30 | 5-20s | Medium-High |
| Multi-Step Sequence | 30-100+ | 30-120s | Very High |
| Adaptive Waiting | 5-15 | 2-30s | Medium |
| Data Extraction | 10-40 | 5-30s | Medium-High |
| State Machine | 40-200+ | 60+s | Very High |

### Common Tool Sequences

**1. Standard Interaction Pattern:**
```
dump_window_hierarchy() → Analyze → click_by_*() or set_text_*()
```

**2. Navigation Pattern:**
```
start_application_by_id() → Wait → is_application_running_foreground() → dump_window_hierarchy()
```

**3. Verification Pattern:**
```
Execute action → dump_window_hierarchy() or get_last_toast() → Verify success
```

**4. Scroll Pattern:**
```
dump_window_hierarchy() → get_device_info() → swipe() → Wait → dump_window_hierarchy()
```

### Data Flow Principles

1. **Always dump hierarchy before interactions**: The AI needs current UI state
2. **Parse outputs before next step**: Tool outputs inform subsequent actions
3. **Verify critical operations**: Check success of important actions
4. **Use context from previous steps**: Data flows forward through the pipeline
5. **Maintain state awareness**: Track where you are in multi-step workflows

### Integration Between Pipelines

Pipelines can be nested and combined:

```
State Machine Pipeline
  ├── Contains: Multi-Step Sequence Pipeline
  │     ├── Uses: Form Submission Pipeline
  │     │     ├── Uses: Text Input Pipeline
  │     │     └── Uses: Basic Element Click Pipeline
  │     └── Uses: Conditional Branching Pipeline
  │           └── Uses: Retry/Error Recovery Pipeline
  └── Uses: Data Extraction Pipeline
        └── Uses: List/Scroll Processing Pipeline
              └── Uses: Adaptive Waiting Pipeline
```

### Key Success Factors

1. **Layout-Based Selectors**: Always prefer element properties over coordinates
2. **Unique Identifiers**: Avoid duplicate resource IDs
3. **Proper Timing**: Allow intervals between operations
4. **Error Handling**: Implement retry logic for robustness
5. **State Verification**: Confirm each step succeeded before proceeding
6. **Adaptive Waiting**: Poll for changes instead of fixed delays
7. **Data Validation**: Verify extracted data makes sense

### Common Pitfalls to Avoid

1. **Using duplicate resource IDs** (violates main system prompt)
2. **Not waiting for page loads** between operations
3. **Fixed delays** instead of adaptive waiting
4. **Ignoring tool output** (missing "false" responses)
5. **No retry logic** for transient failures
6. **Coordinate-based clicking** (discouraged by system prompt)
7. **Not checking app state** before operations

---

## Conclusion

This documentation covers the complete spectrum of agent pipelines in the Lambda Android Automation system, from simple single-click operations to complex state machine workflows. Each pipeline builds upon fundamental MCP tools and follows the quality assurance principles defined in the main system prompt.

By combining these pipeline patterns, AI agents can automate virtually any Android application workflow reliably and efficiently.

