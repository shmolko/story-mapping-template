You are an agent with access to the `user-youtrack` MCP server. Your task is to import a story map into YouTrack by reading a JSON file located in the same directory as this file, then creating the appropriate issues and subtask relationships.

---

## Step 1 — Find and read the story map JSON

Look in the same directory as this file for a `.json` file. Read it. It contains the story map data you will import.

---

## Step 2 — Parse the JSON

From the JSON:

1. Extract the `youtrackUrl` field.
2. Derive the **project short name** from the URL:
   - Pattern: `https://[instance].youtrack.cloud/projects/[SHORTNAME]/issues`
   - Example: `https://mycompany.youtrack.cloud/projects/PROJ/issues` → short name is `PROJ`
3. Walk the structure: goal → story → task. Build an internal plan:
   - **Goals** are NOT created as YouTrack issues. They provide context only.
   - **Stories** become YouTrack issues. Use the story's `text` as `summary`. Set `description` to `"Goal context: [parent goal text]"`.
   - **Tasks** become YouTrack issues. Use the task's `text` as `summary`. Each task must be linked as a subtask of its parent story.

---

## Step 3 — Confirm the plan

Before creating anything, print a summary:

```
Project: PROJ
Stories to create: N
Tasks to create: M

[Goal: "Goal A"]
  - Story: "story text"
      • Task: "task text"
      • Task: "task text"
[Goal: "Goal B"]
  - Story: "story text"
      • Task: "task text"
```

Ask: "Does this look correct? Shall I proceed?"

Wait for confirmation before continuing.

---

## Step 4 — Create issues

Process one story at a time. Fully complete all tasks for a story before moving to the next.

### For each story:

1. Call `create_content` with `action: "create_issue"`:
   - `summary`: story's `text`
   - `description`: `"Goal context: [parent goal text]"`
   - `project`: `{ "shortName": "PROJ" }`
   - Record the returned issue ID (e.g. `PROJ-12`).

2. For each task under this story:
   a. Call `create_content` with `action: "create_issue"`:
      - `summary`: task's `text`
      - `project`: `{ "shortName": "PROJ" }`
      - Record the returned issue ID (e.g. `PROJ-13`).
   b. Call `run_commands` with `action: "execute_command"`:
      - `issues`: `["PROJ-13"]`
      - `command`: `"subtask of PROJ-12"`

### Error handling

- If a `create_issue` call fails: log it, skip that item, continue with the next.
- If an `execute_command` (subtask link) call fails: log it, continue — do not abort the import.

---

## Step 5 — Report results

Print a summary table:

| Story | YouTrack ID | Tasks Created |
| ----- | ----------- | ------------- |
| story text | PROJ-12 | PROJ-13, PROJ-14 |

If any items failed, list them below the table:

```
Failed:
  - Story "..." — create_issue error
  - Task "..." (under PROJ-12) — subtask link failed
```
