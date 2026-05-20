# Nexus Docs

The documentation site for the [Nexus platform](https://github.com/nexus-holdings).

Built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) + a brass-on-charcoal design system + hand-crafted SVG diagrams.

Lives at: **https://nexus-holdings.github.io/nexus-docs/** (once Pages is configured)

## Local development

```bash
uv venv                                  # one-time
uv pip install -e .                      # or install mkdocs-material + mkdocs-d2-plugin
uv run mkdocs serve                      # live-reload on http://127.0.0.1:8000
```

Or just build:

```bash
uv run mkdocs build --strict             # output goes to ./build
```

## Structure

```
docs/
├── index.md                  Landing page
├── getting-started/          Overview, install, first ticket, cheat sheet
├── architecture/             4-layer model + the flywheel thesis
├── components/               Per-component pages (Paperclip, Nexus Core, plugins, ...)
├── concepts/                 Domain primitives (tickets, agents, contracts, evals, ...)
├── guides/                   Worked recipes (create-a-company, debug-a-ticket, ...)
├── reference/                API surfaces, env vars, CLI commands, file locations
├── faq.md                    Top questions for cold arrivals
└── roadmap.md                What's shipped, in flight, planned

assets/
└── diagrams/                 Hand-crafted SVG diagrams (see ~/.claude/skills/structured-diagram)
```

## Deployment

Pushes to `main` trigger `.github/workflows/deploy.yml`, which builds with `mkdocs build --strict` and publishes to GitHub Pages.

## Contributing

- All content is generic — no real customer / person names. See the sanitisation pass in `CONTENT-AUDIT.md`.
- Diagrams: prefer hand-crafted SVG over Mermaid for anything load-bearing (state machines, DFAs, multi-flow). The `structured-diagram` skill at `~/.claude/skills/structured-diagram/` codifies the rules.
- Every claim should be verifiable against source. The reference pages were built by greping live code.
