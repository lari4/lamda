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

