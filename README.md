# Code Style Guidelines

This repository contains language-specific coding guidelines for use with AI coding assistants.

## Usage

Each language has its own markdown file (e.g. `PYTHON.md`) containing conventions, tooling setup, and best practices for that language.

### Claude Code

When starting a project, update `CLAUDE.md` to import the relevant language guidelines:

```
@PYTHON.md
```

Replace `PYTHON.md` with the appropriate file for your language.

### Other LLM tools

For tools that don't support `@file` imports (e.g. Cursor, Copilot, ChatGPT), paste the contents of the relevant language file directly into your system prompt or rules file. For example:

- **Cursor** — add to `.cursor/rules/` or the project system prompt
- **GitHub Copilot** — add to `.github/copilot-instructions.md`
- **Windsurf** — add to `.windsurfrules`

## Available guidelines

| File | Language |
|------|----------|
| `PYTHON.md` | Python |
