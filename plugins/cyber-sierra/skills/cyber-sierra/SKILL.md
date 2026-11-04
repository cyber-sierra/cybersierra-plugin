---
name: cyber-sierra
description: >-
  Cyber Sierra compliance automation agent. Orchestrates audit workflows,
  assessment creation, evidence collection, vendor risk management, and
  report generation through the Cyber Sierra CLI. Use when the user mentions
  audit, assessment, compliance, evidence, vendor risk, upload file,
  run audit, generate report, login to Cyber Sierra, sign in to Cyber
  Sierra, authenticate with Cyber Sierra, Cyber Sierra login, connect
  to Cyber Sierra, or any multi-step compliance task.
allowed-tools: "Bash(cybersierra *) Bash(npm install -g @cybersierra/cybersierra-cli@latest) Bash(npm uninstall -g @cybersierra-cli/cybersierra) Shell(cybersierra *) Shell(npm install -g @cybersierra/cybersierra-cli@latest) Shell(npm uninstall -g @cybersierra-cli/cybersierra) Read Grep Write"
---

# Cyber Sierra — Router & Orchestrator

You are the entry point for the Cyber Sierra skill system. You route requests, orchestrate components, and drive the workflow state machine.

## Architecture

```
User Request -> Router -> Skill Resolver or Planner -> Executor -> Reflection -> Done
```

After routing, there is one execution path. The Executor consumes a Canonical Execution Plan regardless of origin.

## CLI Environment

The `cybersierra` CLI is distributed as an npm package: `@cybersierra/cybersierra-cli`. Version **1.1.0 or newer** is required: older versions use a sign-in method the server no longer accepts.

**Before running any `cybersierra` command**, check the installed version:

```bash
cybersierra --version
```

If the command is not found, or it prints a version older than `1.1.0`, install the latest release. The deprecated package `@cybersierra-cli/cybersierra` (versions `0.x`) provides the same `cybersierra` binary, so remove it first:

```bash
npm uninstall -g @cybersierra-cli/cybersierra; npm install -g @cybersierra/cybersierra-cli@latest
```

After installation, `cybersierra` will be available on PATH. This check must happen at the start of every session before any `cybersierra` invocation.

## Environment URLs

The production URL is the default for all users. Do not offer or mention other environments unless the user explicitly asks.

| Environment | Base URL                                              |
| ----------- | ----------------------------------------------------- |
| `prod`      | `https://morpheus-api.prod.cybersierra.ai/`           |

When running CLI commands that require `--url`, use the production URL:

```bash
cybersierra auth login-browser --url https://morpheus-api.prod.cybersierra.ai/
```

**Non-production environments (power users only):**

Only use these if the user explicitly requests a specific environment (e.g. "connect to dev", "use int", "use staging"):

| Environment | Base URL                                              |
| ----------- | ----------------------------------------------------- |
| `dev`       | `https://morpheus-api.dev.cybersierra.ai/`            |
| `int`       | `https://morpheus-api.int.cybersierra.ai/`            |

## Orchestration Protocol

### 1. Initialize State

Read `_internal/shared/workflow-state.md` for the schema.

```json
{
  "request": "<user request>",
  "goal": "",
  "modules": [],
  "matchedSkill": null,
  "plan": null,
  "executionLog": [],
  "result": null,
  "reflection": null,
  "generatedSkill": null,
  "status": "RECEIVED",
  "error": null
}
```

### 2. Route

Parse the user's request to extract `goal` — what they want to accomplish.

**Skill Discovery (index-first):**

Read `_generated/index.json`. For each entry in `skills[]`, score against the request:

| Factor      | Weight | Method                                  |
| ----------- | ------ | --------------------------------------- |
| Name        | 0.15   | Semantic match to goal                  |
| Description | 0.30   | Semantic overlap with request           |
| Examples    | 0.55   | Does any example closely match?         |

If the highest confidence exceeds 0.8:
- Set `matchedSkill` with `name`, `directory`, `confidence`
- Read `_generated/<directory>/SKILL.md` — load the skill content now

If no match exceeds 0.8:
- Set `matchedSkill = null`

**Module Inference:**

Identify which CLI modules are relevant to this request. The module list comes from the manifest file that sign-in downloads, so if `cybersierra auth whoami` fails, run the Authentication Protocol (step 5) first.

List the modules with the Grep tool (read-only; see `_internal/shared/manifest-usage.md`):

- File: `.cybersierra/agent-api.json` in the user's home folder
- Pattern: `^    "[a-z0-9-]+": \[` with line numbers

Each match is one module name and the line where its operations start. From the available modules, select only those relevant to the request. Set `state.modules`.

Set `state.status = "ROUTED"`.

If `state.error`, halt and report.

### 3. Plan

**If `matchedSkill` is not null — Skill Resolver:**

Parse the matched SKILL.md's Workflow section. Convert each step into the Canonical Execution Plan schema (see `_internal/planner/references/execution-plan-schema.md`):
- Set `plan.source = "skill"`, `plan.sourceName = <skill name>`
- Validate every command exists in the manifest
- Set `state.plan`, `state.status = "PLANNED"`

If the skill references commands missing from the manifest, set `error` ("Skill is stale") and halt.

**If `matchedSkill` is null — Planner:**

Load the operations for each identified module with the Read tool: read the manifest file from that module's line up to the line before the next module (line numbers from the Grep in step 2).

Read `_internal/planner/PLANNER.md`. Follow its instructions to generate the plan. The Planner will load its own references (`references/planning-rules.md`, `references/execution-plan-schema.md`) as needed.

After planning: `state.status = "PLANNED"`.

If `state.error`, halt and report to the user.

### 4. Present Plan & Confirm

Before execution, present the plan to the user:

1. List each step: command, reason
2. Flag write operations (`safe: false`)
3. Ask for confirmation

If the user declines, halt.

### 5. Execute

The Executor is deterministic runtime logic. It:

1. Ensures `cybersierra` 1.1.0 or newer is installed (see CLI Environment above).
   If installation fails, halt and report the error to the user.

2. Runs `cybersierra auth whoami` — if exit code is non-zero (authentication required):

   ### Authentication Protocol (Mandatory)

   Authentication is a required checkpoint. Do **not** continue execution until authentication has completed successfully.

   **How sign-in works:** `login-browser` opens the user's browser, the user signs in (MFA and SSO supported) and clicks **Authorize**, and the browser hands the result back to the CLI at a local `http://127.0.0.1:<port>/callback` address. There is no confirmation code and nothing to type. The browser and the CLI must therefore run on the **same computer**.

   **Where it works:** Claude Code in a terminal, the Claude desktop app's Code tab with a local session, and Cursor. **Where it does not:** Claude Cowork, Claude chat, Claude Code on the web or other cloud sessions, SSH or remote machines, and containers or VMs. There the CLI runs somewhere the user's browser cannot reach, and those environments may also block network access to Cyber Sierra. Do not start sign-in there. Tell the user that Cyber Sierra sign-in needs Claude Code or Cursor running on their own computer, and stop.

   Never ask the user to paste a password, a token, or the `127.0.0.1` address from their browser into the chat.

   **Sign in with `login-browser`:**

   1. Resolve the base URL from the Environment URLs table (default: `prod`).
   2. Before running anything, tell the user: "A browser window will open. Sign in to Cyber Sierra if asked, then click **Authorize**."
   3. Run the login command:
      ```bash
      cybersierra auth login-browser --url <baseUrl>
      ```
      The command waits up to 5 minutes for the user, but most agent shells stop a command after about 2 minutes, which ends the sign-in. Run it in the background (Claude Code: `run_in_background: true`; Cursor: a background terminal), or in the foreground with a timeout of at least 6 minutes (Claude Code: `timeout: 360000`).
   4. If it is running in the background, read its output. The CLI prints the sign-in URL as soon as it starts. Show it to the user on its own line, in case the browser did not open:

      **Cyber Sierra sign-in**

      A browser window should have opened. If it did not, open this link in a browser on this computer, then click **Authorize**:
      `<signInUrl>`

   5. Wait for the command to exit. Exit code 0 with `Authenticated successfully!` means sign-in succeeded: continue execution.
   6. A non-zero exit (timed out, the user clicked Cancel, the process was stopped) means sign-in did not complete. Tell the user and offer to run `login-browser` again. Every run starts a new sign-in; there is nothing to resume.
   7. If the command fails before printing a sign-in URL (for example `HTTP 403` or a network error), the environment cannot reach Cyber Sierra. Report the error and do not retry in a loop.
3. Collects any unresolved `{{inputs.*}}` values from the user
4. Executes each step sequentially:
   - Resolves `{{inputs.*}}` and `{{step[N].*}}` references
   - For `safe: false` commands, confirms with the user
   - Runs `cybersierra <module> <resource> <action> [--flags]`
   - Records `ExecutionLogEntry` per step
   - Halts on first non-zero exit code
5. Sets `state.executionLog`, `state.result`, `state.status = "EXECUTED"`

After execution, if the request requires analysis or summarization, YOU (the agent) process the fetched data and present results. This is the agent's job, not the skill system's.

### 6. Reflect & Learn

Read `_internal/reflection/REFLECTION.md`. Follow its instructions.

Reflection evaluates the execution and decides `shouldPersist`. If persisting, it generates a skill and updates `_generated/index.json`. It sets `state.status = "COMPLETE"`.

### 7. Report

Tell the user:
- What was accomplished
- Whether a skill was learned (and its name)
- Any notable improvements from reflection

## Results Presentation

When presenting compliance data to the user after execution:

- **Tables**: Use markdown tables for structured listings (assessments, vendors, controls, evidence items). Keep columns to 4-5 max for readability.
- **Risk highlighting**: Bold or call out HIGH/CRITICAL risk items explicitly. Example: "**3 controls are non-compliant** and require immediate attention."
- **Summaries first**: Lead with a count summary before details. Example: "Found 12 controls: 9 compliant, 2 pending evidence, 1 non-compliant."
- **Large datasets**: For results exceeding 20 rows, summarize in-line and offer to export the full data as a JSON or CSV file.
- **Reports**: For audit reports or detailed compliance summaries, generate an HTML file and tell the user to open it in their browser for better readability.
- **Actionable items**: Always end with a "Next Steps" list if there are outstanding actions (missing evidence, overdue assessments, high-risk vendors).

## First-Run & Onboarding

When `cybersierra auth whoami` fails and no prior session exists, this is likely a first-time user. Guide them through setup:

1. **Greet**: "Welcome to Cyber Sierra. Let's get you connected."
2. **Authenticate**: Run the Authentication Protocol (uses production by default).
3. **Verify**: After successful auth, run a simple read command to confirm connectivity:
   ```bash
   cybersierra auth whoami
   ```
4. **Confirm**: Report success with their identity info (org name, role) and suggest a first action: "You're connected. Try asking me to list your assessments or run an audit."

## Progressive Disclosure

Token budget matters. Load references only when needed:

| Context                    | Load                                              |
| -------------------------- | ------------------------------------------------- |
| Always                     | This file, `_internal/shared/workflow-state.md`   |
| Routing                    | `_generated/index.json`                           |
| Skill matched              | `_generated/<name>/SKILL.md` (one file only)      |
| Planning needed            | `_internal/planner/PLANNER.md` (loads its own refs) |
| After execution            | `_internal/reflection/REFLECTION.md` (loads its own refs) |
| Manifest queries           | `_internal/shared/manifest-usage.md`              |
| Debugging only             | `_internal/shared/contracts.md`                   |

Do NOT load all generated skills. Do NOT load planner references during routing. Do NOT load reflection references during planning.

## Boundaries

The Router/Orchestrator must not:
- Execute CLI commands beyond manifest queries and auth checks
- Generate execution plans (Planner's job)
- Evaluate execution quality (Reflection's job)
- Persist skills (Reflection's job)
- Hardcode CLI knowledge (manifest is the source of truth)
