# Copilot Skills

A personal repository of reusable GitHub Copilot skills, custom instructions, and prompt patterns organized by language and use case.

## What Are Copilot Skills?

Copilot skills are reusable instructions, prompt patterns, and configurations that guide GitHub Copilot to produce more consistent, higher-quality results for specific tasks. They capture best practices and preferences so you don't have to repeat context every time.

## Repository Structure

```
. (repository root)
├── README.md                  # Main README (this file)
├── docs/                      # General documentation and guides
│   └── getting-started.md
├── skills/
│   ├── python/                # Python-specific skills
│   │   └── README.md
│   ├── javascript/            # JavaScript-specific skills
│   │   └── README.md
│   ├── typescript/            # TypeScript-specific skills
│   │   └── README.md
│   └── general/               # Language-agnostic skills
│       └── README.md
└── .github/
    └── CONTRIBUTING.md
```

## Getting Started

1. Browse the `skills/` directory and find the category that matches your task.
2. Open the relevant skill file to read the instructions or prompt pattern.
3. Copy or adapt the skill into your Copilot chat session or custom instructions.

See [docs/getting-started.md](docs/getting-started.md) for a more detailed walkthrough.

## Categories

| Category | Description |
|---|---|
| [Python](skills/python/) | Skills for Python development — testing, typing, documentation, and more |
| [JavaScript](skills/javascript/) | Skills for JavaScript — async patterns, DOM manipulation, Node.js |
| [TypeScript](skills/typescript/) | Skills for TypeScript — strict typing, generics, interfaces |
| [General](skills/general/) | Language-agnostic skills — code review, refactoring, writing commit messages |

## Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](.github/CONTRIBUTING.md) before submitting a new skill.

## License

This repository is for personal use. Feel free to adapt anything here for your own workflow.
