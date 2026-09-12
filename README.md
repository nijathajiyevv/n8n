# n8n

Personal collection of n8n workflow automations. Each subfolder is one standalone project: an exported workflow JSON plus a README explaining what it does, why it exists, and how to set it up.

## Structure

Each project folder follows the same layout:

```
project-name/
├── README.md              — what the workflow does, architecture, setup steps
└── workflow-name.json     — the exported n8n workflow, importable as-is
```

## Using these workflows

1. Open the project folder you want and read its README first — setup steps and required credentials differ per project.
2. Import the `.json` file into your n8n instance (`Workflows → Import from File`).
3. Create your own credentials for any external service the workflow uses (Gmail, OpenRouter, etc.). Exported workflows never contain real credentials or API keys — only placeholder references you replace with your own.
4. Check each project's README for any account-specific IDs (label IDs, calendar IDs, etc.) that need to be filled in manually, since these are stripped from the export and cannot be reused across accounts.

## Before adding a new project

Exported n8n JSON can carry account-specific data beyond credentials: label/calendar/sheet IDs, and any personal details hardcoded into node parameters or prompts (real sender names, real addresses, account-specific identifiers). Review and strip these before committing a new workflow to this repo.
