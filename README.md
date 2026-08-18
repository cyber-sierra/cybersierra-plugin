# Cyber Sierra — Claude Code Plugin

Compliance automation plugin for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Orchestrates audit workflows, assessment creation, evidence collection, vendor risk management, and report generation through the **Cyber Sierra CLI**.

## Directory Layout

```
.claude-plugin/
    plugin.json          # Plugin manifest — registers skills and assets
    marketplace.json     # Marketplace metadata for future publishing

skills/
    cyber-sierra/
        SKILL.md                     # Main skill: router & orchestrator
        _generated/
            index.json               # Learned-skill discovery index
        _internal/
            planner/
                SKILL.md             # Execution plan generator
                references/
                    execution-plan-schema.md
                    planning-rules.md
            reflection/
                SKILL.md             # Post-execution reflection & learning
                references/
                    learning-rules.md
                    reflection-schema.md
                    skill-template.md
            shared/
                contracts.md         # Component contracts
                manifest-usage.md    # How to query the CLI manifest
                workflow-state.md    # Pipeline state schema

```

## Cyber Sierra CLI

The `cybersierra` CLI is published as an npm package and installed automatically when the skill runs:

```bash
npm install -g @cybersierra/cybersierra-cli
```

The skill checks for `cybersierra` availability before every session and installs it if missing. No manual binary placement is required.

## Local Installation

1. **Clone this repository:**

   ```bash
   git clone https://github.com/cyber-sierra/claude-plugin.git
   cd claude-plugin
   ```

2. **Install the plugin in Claude Code:**

   ```bash
   claude plugin install /path/to/claude-plugin
   ```

3. **Verify the installation:**

   ```bash
   claude plugin list
   ```

   You should see `cyber-sierra` listed.

4. **(Optional) Pre-install the Cyber Sierra CLI:**

   ```bash
   npm install -g @cybersierra/cybersierra-cli
   ```

   If you skip this step, the plugin will install it automatically on first use.

## Usage

Once installed, ask Claude Code to perform compliance tasks:

- *"Run an audit for my organization"*
- *"Create a vendor risk assessment"*
- *"Collect evidence for SOC 2 controls"*
- *"Generate a compliance report"*
- *"Log in to Cyber Sierra"*

The plugin routes your request through the skill system, builds an execution plan from the CLI manifest, and runs it after your confirmation.

## How It Works

The plugin implements a multi-stage pipeline:

1. **Router** — Parses the request, checks for matching learned skills, identifies relevant CLI modules
2. **Planner** — Generates a Canonical Execution Plan from CLI manifest commands (when no learned skill matches)
3. **Executor** — Runs the plan step-by-step, handling authentication and user confirmation for write operations
4. **Reflection** — Evaluates execution quality and optionally persists successful workflows as reusable skills

## Publishing

This plugin is designed for eventual publication through the Claude Code plugin marketplace. The `.claude-plugin/marketplace.json` file contains the required metadata. Before publishing:

1. Populate `icon`, `changelog`, and `minClaudeCodeVersion` fields in `marketplace.json`
2. Ensure `@cybersierra/cybersierra-cli` is published and accessible on npm
3. Follow the Claude Code plugin publishing guide (when available)

## License

See repository root for license details.
