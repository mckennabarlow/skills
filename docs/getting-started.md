# Getting Started

This guide explains how to use the skills in this repository with GitHub Copilot.

## What Is a Skill?

Each skill is a focused set of instructions or a prompt pattern designed to help Copilot produce better output for a specific task. Skills are stored as Markdown files so they are easy to read, copy, and adapt.

## How to Use a Skill

### Option 1: Paste into Copilot Chat

1. Open Copilot Chat in your editor (VS Code, JetBrains, etc.).
2. Find the skill you want in the `skills/` directory.
3. Copy the prompt text from the skill file.
4. Paste it at the start of your Copilot Chat conversation, then add your specific request.

### Option 2: Add to Custom Instructions

GitHub Copilot supports [custom instructions](https://docs.github.com/en/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot) that are automatically included in every conversation.

1. Open your editor settings.
2. Find the Copilot custom instructions setting.
3. Paste the relevant skill content there.

### Option 3: Use as a `.github/copilot-instructions.md` File

You can place instructions directly in your project repository so they apply automatically:

1. Create `.github/copilot-instructions.md` in your project.
2. Copy the skill content you want into that file.
3. Copilot will automatically include these instructions in chat sessions within that repository.

## Finding the Right Skill

Browse by category:

- **[Python](../skills/python/)** — Testing, type hints, docstrings, and Pythonic patterns
- **[JavaScript](../skills/javascript/)** — Async/await, modules, Node.js conventions
- **[TypeScript](../skills/typescript/)** — Strict typing, generics, interface design
- **[General](../skills/general/)** — Code review, commit messages, refactoring

## Adding a New Skill

See [CONTRIBUTING.md](../.github/CONTRIBUTING.md).
