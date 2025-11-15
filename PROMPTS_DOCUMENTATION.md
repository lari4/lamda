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

