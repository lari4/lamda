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

