# AI Prompts Documentation

This document contains all AI prompts used in the Lambda Android Automation application, organized by thematic categories.

## Table of Contents

1. [Main System Prompt](#main-system-prompt)
2. [Device Control Prompts](#device-control-prompts)
3. [Application Management Prompts](#application-management-prompts)
4. [Element Interaction Prompts](#element-interaction-prompts)
5. [Data Access Prompts](#data-access-prompts)

---

## Main System Prompt

### Android Automation AI Agent

**Location:** `extensions/firerpa.py:21-33`

**Purpose:** This is the primary system prompt that defines the core behavior and guidelines for the Android automation AI agent. It instructs the AI to act as an expert in Android automation, emphasizing quality assurance practices and proper element identification strategies.

**Key Features:**
- Establishes the AI agent as an Android automation expert
- Provides quality assurance guidelines for reliable automation
- Sets communication preferences
- Defines operational constraints

**Prompt:**
```markdown
# Android Automation Guidelines

You are an expert in Android automation, capable of using specialized tools to accurately complete user-requested operations. You understand and can precisely execute each step in the automation process.

## Quality Assurance
- Prioritize using layout information for identifying operational elements. You should always use non-repeating criteria for judgment.
- Resource-id may be duplicated, and duplicate ids should not be used.
- Each operation should have a certain interval; otherwise, the page may not be fully loaded.
- Never use screenshot to detect coordinates.

## Communication
- Follow the user's language preferences
```

---

## Device Control Prompts

These prompts are tool descriptions that guide the AI agent on how to interact with and control Android device hardware and system features.

### 1. Dump Window Hierarchy

**Location:** `extensions/firerpa.py:41`

**Purpose:** Instructs the AI to retrieve the complete UI layout hierarchy of the current Android window in JSON format. This is essential for the AI to understand the structure of the screen before performing any interactions.

**Use Case:** Used as the first step in most automation workflows to analyze available UI elements.

**Prompt:**
```
Dumps android window's layout hierarchy as JSON string.
```

### 2. Click at Coordinates

**Location:** `extensions/firerpa.py:45`

**Purpose:** Enables the AI to perform a tap gesture at specific x,y coordinates on the screen. However, the main system prompt discourages using coordinates in favor of element-based interactions.

**Use Case:** Fallback option when element selectors cannot identify the target.

**Prompt:**
```
Perform a click at arbitrary coordinates on the display.
```

### 3. Swipe Gesture

**Location:** `extensions/firerpa.py:49`

**Purpose:** Instructs the AI to perform a swipe gesture between two points on the screen, commonly used for scrolling or navigating between screens.

**Use Case:** Scrolling through lists, swiping between pages, pull-to-refresh actions.

**Prompt:**
```
Perform a swipe between two points.
```

### 4. Drag Gesture

**Location:** `extensions/firerpa.py:53`

**Purpose:** Guides the AI to perform a drag operation from one point to another, typically used for drag-and-drop interactions or reordering items.

**Use Case:** Moving icons, reordering list items, drag-and-drop operations.

**Prompt:**
```
Perform a drag between two points.
```

### 5. Get Device Information

**Location:** `extensions/firerpa.py:57`

**Purpose:** Instructs the AI to retrieve comprehensive device information including screen dimensions, brand, model, and other hardware specifications.

**Use Case:** Used to adapt automation logic based on device characteristics.

**Prompt:**
```
Get device information such as screen width, height, brand, etc.
```

### 6. Wake Up Device

**Location:** `extensions/firerpa.py:73`

**Purpose:** Enables the AI to turn on the device screen when it's off.

**Use Case:** Ensuring the device is awake before starting automation tasks.

**Prompt:**
```
Wake up the device.
```

### 7. Sleep Device

**Location:** `extensions/firerpa.py:78`

**Purpose:** Instructs the AI to turn off the device screen.

**Use Case:** Saving power after completing automation tasks.

**Prompt:**
```
Turn off the device screen.
```

### 8. Check Screen Status

**Location:** `extensions/firerpa.py:81`

**Purpose:** Guides the AI to check if the device screen is currently lit up.

**Use Case:** Verifying screen state before performing visual operations.

**Prompt:**
```
Check if the device screen is lit up.
```

### 9. Check Lock Status

**Location:** `extensions/firerpa.py:85`

**Purpose:** Instructs the AI to determine if the device screen is locked.

**Use Case:** Ensuring the device is unlocked before performing automation tasks.

**Prompt:**
```
Check is the device screen locked.
```

### 10. Get Clipboard Content

**Location:** `extensions/firerpa.py:89`

**Purpose:** Enables the AI to read the current clipboard content.

**Use Case:** Retrieving copied text, verifying clipboard operations.

**Prompt:**
```
Get the device clipboard content.
```

### 11. Set Clipboard Content

**Location:** `extensions/firerpa.py:93`

**Purpose:** Instructs the AI to set text to the device clipboard.

**Use Case:** Preparing text for paste operations, sharing data between apps.

**Prompt:**
```
Set the device clipboard content.
```

### 12. Press Key Code

**Location:** `extensions/firerpa.py:97`

**Purpose:** Guides the AI to simulate hardware button presses using Android KeyEvent codes.

**Use Case:** Pressing back button, home button, volume controls, etc.

**Prompt:**
```
Simulates a short press using a key code.
```

### 13. Show Toast Message

**Location:** `extensions/firerpa.py:65`

**Purpose:** Instructs the AI to display a temporary toast notification on the screen.

**Use Case:** Providing user feedback, debugging automation steps.

**Prompt:**
```
Display a toast message on the screen.
```

### 14. Get Last Toast

**Location:** `extensions/firerpa.py:101`

**Purpose:** Enables the AI to retrieve the last toast message that was displayed on the device.

**Use Case:** Verifying app feedback messages, detecting error notifications.

**Prompt:**
```
Get the last displayed toast on the system.
```

### 15. Execute Shell Script

**Location:** `extensions/firerpa.py:69`

**Purpose:** Instructs the AI to execute shell commands in the device's foreground shell environment.

**Use Case:** Advanced system operations, file management, system configuration.

**Prompt:**
```
Execute script in the device's shell foreground.
```

---

## Application Management Prompts

These prompts guide the AI agent on how to manage, control, and interact with Android applications.

### 1. List Installed Applications

**Location:** `extensions/firerpa.py:61`

**Purpose:** Instructs the AI to retrieve a list of all package names for applications installed on the device.

**Use Case:** Discovering available apps before launching or checking if a specific app is installed.

**Prompt:**
```
List the package names of installed applications on the device.
```

### 2. Get Current Application Info

**Location:** `extensions/firerpa.py:144`

**Purpose:** Guides the AI to get information about the currently running foreground application.

**Use Case:** Verifying which app is currently active, context-aware automation.

**Prompt:**
```
Get information about the currently running foreground application.
```

### 3. Start Application

**Location:** `extensions/firerpa.py:148`

**Purpose:** Instructs the AI to launch an Android application using its package name.

**Use Case:** Opening specific apps as part of automation workflows.

**Prompt:**
```
Use the package name to launch an Android app.
```

### 4. Stop Application

**Location:** `extensions/firerpa.py:152`

**Purpose:** Enables the AI to close/force-stop an Android application using its package name.

**Use Case:** Cleaning up after automation, closing apps to free resources.

**Prompt:**
```
Use the package name to close an Android app.
```

### 5. Check Installation Status

**Location:** `extensions/firerpa.py:156`

**Purpose:** Guides the AI to verify if a specific application is installed on the device.

**Use Case:** Prerequisite checking before attempting to launch or interact with an app.

**Prompt:**
```
Use the package name to check if the application is installed.
```

### 6. Check Foreground Status

**Location:** `extensions/firerpa.py:160`

**Purpose:** Instructs the AI to determine if a specific application is currently running in the foreground.

**Use Case:** Verifying app launch success, ensuring correct app is active before proceeding.

**Prompt:**
```
Check if the application is running in the foreground using the package name.
```

### 7. List Application Permissions

**Location:** `extensions/firerpa.py:164`

**Purpose:** Enables the AI to retrieve all manifest permissions declared by an application.

**Use Case:** Understanding app capabilities, auditing permissions.

**Prompt:**
```
Get all manifest permissions of the application using the package name.
```

### 8. Grant Permission

**Location:** `extensions/firerpa.py:168`

**Purpose:** Instructs the AI to grant runtime permissions to an application.

**Use Case:** Ensuring apps have necessary permissions before automation, testing permission-dependent features.

**Prompt:**
```
Grant the application runtime permissions.
```

### 9. Revoke Permission

**Location:** `extensions/firerpa.py:172`

**Purpose:** Guides the AI to revoke runtime permissions from an application.

**Use Case:** Testing app behavior without certain permissions, security testing.

**Prompt:**
```
Revoke the application's runtime permissions.
```

### 10. Check Permission Status

**Location:** `extensions/firerpa.py:176`

**Purpose:** Instructs the AI to check if a specific permission has been granted to an application.

**Use Case:** Verifying permission state before attempting permission-dependent operations.

**Prompt:**
```
Check if the application has been granted runtime permissions.
```

---

## Element Interaction Prompts

These prompts guide the AI agent on how to identify and interact with UI elements on the screen using various selector strategies.

### 1. Click by Exact Text

**Location:** `extensions/firerpa.py:108`

**Purpose:** Instructs the AI to find and click an element by matching its exact text content.

**Use Case:** Clicking buttons, links, or labels with known exact text.

**Prompt:**
```
Use full text matching to click on an element.
```

### 2. Click by Text Contains

**Location:** `extensions/firerpa.py:112`

**Purpose:** Guides the AI to find and click an element containing a specific substring in its text.

**Use Case:** Clicking elements when only part of the text is known or when text varies slightly.

**Prompt:**
```
Use text contains matching to click on an element.
```

### 3. Click by Text Regex

**Location:** `extensions/firerpa.py:116`

**Purpose:** Instructs the AI to find and click an element using regular expression matching on its text.

**Use Case:** Complex text patterns, variable content, flexible matching scenarios.

**Prompt:**
```
Use text regex matching to click on an element.
```

### 4. Click by Exact Description

**Location:** `extensions/firerpa.py:120`

**Purpose:** Enables the AI to find and click an element by matching its exact content description (accessibility label).

**Use Case:** Clicking elements without visible text but with accessibility descriptions.

**Prompt:**
```
Use full description matching to click on an element.
```

### 5. Click by Description Contains

**Location:** `extensions/firerpa.py:124`

**Purpose:** Guides the AI to find and click an element containing a specific substring in its content description.

**Use Case:** Partial matching on accessibility labels.

**Prompt:**
```
Use description contains matching to click on an element.
```

### 6. Click by Description Regex

**Location:** `extensions/firerpa.py:128`

**Purpose:** Instructs the AI to find and click an element using regular expression matching on its content description.

**Use Case:** Complex description patterns, flexible accessibility label matching.

**Prompt:**
```
Use description regex matching to click on an element.
```

### 7. Click by Resource ID

**Location:** `extensions/firerpa.py:132`

**Purpose:** Guides the AI to find and click an element using its resource ID. Important note: warns that duplicate resource IDs cannot be used.

**Use Case:** Precise element targeting when resource IDs are unique (as emphasized in main system prompt).

**Prompt:**
```
Use resourceId to click on an element, if the resource-id is duplicated, it cannot be used.
```

### 8. Set Text by Resource ID

**Location:** `extensions/firerpa.py:136`

**Purpose:** Instructs the AI to input text into an input field identified by its resource ID.

**Use Case:** Filling forms, entering search queries, text input automation.

**Prompt:**
```
Use resourceId to input text into an input element, if the resource-id is duplicated, it cannot be used.
```

### 9. Set Text by Class Name

**Location:** `extensions/firerpa.py:140`

**Purpose:** Guides the AI to input text into an input field identified by its class name (e.g., android.widget.EditText).

**Use Case:** Generic text input when resource ID is not available or duplicated.

**Prompt:**
```
Use className to input text into an input element.
```

---

## Data Access Prompts

These prompts guide the AI agent on how to read and access various types of data from the Android device.

### 1. Read System Property

**Location:** `extensions/firerpa.py:105` and `extensions/example_mcp_extension.py:20`

**Purpose:** Instructs the AI to read Android system properties by name (similar to the `getprop` shell command).

**Use Case:** Retrieving device configuration, SDK version, manufacturer info, etc.

**Prompt:**
```
Read android system property by name.
```

### 2. Read File Content

**Location:** `extensions/example_mcp_extension.py:23-25`

**Purpose:** Guides the AI to read file content from the device using an absolute file path. Returns content as base64-encoded blob.

**Use Case:** Reading configuration files, logs, exported data files.

**Implementation Note:** This is a resource endpoint, not a tool. Uses MCP resource protocol with URI scheme `file://{absolute_path}`.

**Prompt:**
```
Read file content on the device by full path
```

### 3. Read SMS Database via SQL

**Location:** `extensions/mcp_sms_reader.py:21-22`

**Purpose:** Instructs the AI to query the Android SMS database using SQLite syntax. Important: read-only operations only, no write operations allowed. The AI is guided to learn table structure when needed.

**Use Case:** Reading text messages, analyzing SMS history, extracting message metadata.

**Security:** Enforces read-only access via `PRAGMA query_only`.

**Prompt:**
```
Reads the SMS database using SQL statements in SQLite syntax; read-only, no write operations allowed.
The database is standard android mmssms.db, you should always learn the tables or table structure if needed.
```

### 4. Greeting Tool (Example)

**Location:** `extensions/example_mcp_extension.py:16`

**Purpose:** Example demonstration tool that sends a greeting message. Not used in production automation.

**Use Case:** Testing MCP functionality, example implementation reference.

**Prompt:**
```
Send a greeting to others.
```

---

## Summary

This documentation covers **44 distinct AI prompts** organized into 5 thematic categories:

1. **Main System Prompt**: 1 core system prompt establishing AI agent behavior
2. **Device Control Prompts**: 15 prompts for hardware and system control
3. **Application Management Prompts**: 10 prompts for app lifecycle and permissions
4. **Element Interaction Prompts**: 9 prompts for UI element identification and interaction
5. **Data Access Prompts**: 4 prompts for reading device data (including 1 example tool)

### Architecture Notes

The application uses the **Model Context Protocol (MCP)** standard by Anthropic to provide a structured interface between AI models and Android automation tools. Each prompt serves as either:

- **System Prompt**: High-level behavioral guidelines for the AI agent
- **Tool Description**: Instructions for specific automation capabilities
- **Resource Description**: Data access endpoints with URI schemes

The main system prompt emphasizes quality assurance principles, particularly:
- Preferring layout-based element identification over coordinates
- Avoiding duplicate resource IDs
- Maintaining proper timing between operations
- Never using screenshots for coordinate detection

All tool descriptions are concise, action-oriented prompts that guide the AI to use the underlying Android automation APIs correctly and safely.

