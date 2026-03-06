# Contributing

Thank you for contributing to the Copilot Skills repository!

## How to Add a New Skill

1. **Choose the right category** — Place your skill in the most appropriate subdirectory under `skills/`:
   - `skills/python/` for Python-specific skills
   - `skills/javascript/` for JavaScript-specific skills
   - `skills/typescript/` for TypeScript-specific skills
   - `skills/general/` for language-agnostic skills

2. **Create a new Markdown file** — Name the file descriptively (e.g., `testing.md`, `async-patterns.md`).

3. **Use the standard template**:

   ```markdown
   # Skill Name

   ## Purpose
   Brief description of what this skill helps Copilot do better.

   ## Prompt / Instructions
   The actual instructions or prompt text to use with Copilot.

   ## Example
   Optional: show an example input/output pair.
   ```

4. **Update the category README** — Add a row to the skills table in the relevant `README.md`.

## Guidelines

- Keep each skill focused on a single task or concept.
- Write prompt instructions in clear, imperative language.
- Include an example when it significantly improves clarity.
- Prefer specific, actionable instructions over vague guidance.
