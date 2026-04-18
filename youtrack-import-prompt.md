# YouTrack Story Map Import Prompt

**What this does:** Paste this prompt into a Cursor chat (or any LLM with YouTrack MCP access), along with your story map JSON, and the agent will automatically create all stories and tasks in YouTrack with the correct subtask relationships.

---

## Before you run this

Paste **two things** into the chat:

1. This entire prompt
2. Your story map JSON (copy the full JSON content directly into the message)

Make sure your LLM session has access to the `user-youtrack` MCP server.

---

## Instructions for the LLM

You are an agent with access to the `user-youtrack` MCP server. Follow these steps precisely to import a story map JSON into YouTrack.

---

### Step 1 — Parse the JSON

From the JSON provided by the user:

1. Extract the `youtrackUrl` field.
2. Derive the **project short name** from the URL using this pattern:
  `https://[instance].youtrack.cloud/projects/[SHORTNAME]/issues`
   For example: `https://mycompany.youtrack.cloud/projects/PROJ/issues` → short name is `PROJ`.
3. Walk through every goal → story → task and build an internal list of what needs to be created:
  - **Goals** are NOT created as YouTrack issues. They are planning context only.
  - **Stories** will be created as YouTrack issues. The story's `text` becomes the `summary`. The description should include the goal context: `"Goal context: [goal text]"`.
  - **Tasks** will be created as YouTrack issues. The task's `text` becomes the `summary`. Each task must be linked as a subtask of its parent story.

---

### Step 2 — Confirm the plan with the user

Before creating anything, print a summary like this:

```
Project: PROJ
Stories to create: 5
Tasks to create: 12

Stories and their tasks:
  [Goal: "Goal A text"]
    - Story: "As a user, I want to log in"
        • Task: "Implement login API"
        • Task: "Add form validation"
  [Goal: "Goal B text"]
    - Story: "As a user, I want to reset my password"
        • Task: "Send reset email"
```

Then ask: **"Does this look correct? Shall I proceed with creating these issues in YouTrack?"**

Wait for the user to confirm before continuing.

---

### Step 3 — Create issues in order

Process one story at a time. For each story, fully complete its tasks before moving on to the next story. Follow this exact sequence:

#### For each story:

1. **Create the story issue** using `create_content` → `create_issue`:
  - `summary`: the story's `text` field
  - `description`: `"Goal context: [the parent goal's text]"`
  - `project`: `{ "shortName": "PROJ" }` (use the short name you derived in Step 1)
  - Record the returned YouTrack issue ID (e.g. `PROJ-12`).
2. **For each task belonging to this story**, in order:
  a. **Create the task issue** using `create_content` → `create_issue`:
      - `summary`: the task's `text` field
      - `project`: `{ "shortName": "PROJ" }`
      - Record the returned YouTrack issue ID (e.g. `PROJ-13`).
   b. **Link the task as a subtask** of the story using `run_commands` → `execute_command`:
      - `issues`: `["PROJ-13"]` (the task's issue ID)
      - `command`: `"subtask of PROJ-12"` (the story's issue ID)
   c. Repeat for the next task.
3. Move on to the next story.

#### Error handling

- If any `create_issue` call fails, log the failure (note which story or task it was), skip that item, and continue with the next one.
- If a `execute_command` (subtask link) call fails, log it but still continue — do not abort the whole import.
- Do not stop the entire import because of a single failure.

---

### Step 4 — Report results

After all issues have been processed, print a summary table:


| Story                                  | YouTrack ID | Tasks Created    |
| -------------------------------------- | ----------- | ---------------- |
| As a user, I want to log in            | PROJ-12     | PROJ-13, PROJ-14 |
| As a user, I want to reset my password | PROJ-15     | PROJ-16          |


If any items failed, list them clearly beneath the table:

```
Failed items:
  - Story "As a user, I want to export data" — create_issue returned an error
  - Task "Add form validation" (under PROJ-12) — subtask link failed
```

---

## Reusing this prompt

This file is reusable for any project. To import a different story map in the future:

1. Open a new Cursor chat with YouTrack MCP access
2. Paste this prompt
3. Paste the new story map JSON alongside it

The agent will derive the project and structure automatically from the JSON.