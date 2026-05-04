# skills

AI agent skills collection.

## Available Skills

### Nitpicker

Multi-agent post-implementation code review with severity-gated progressive review. Deploys specialist sub-agents in tiered batches — each tier must pass a gate check before the next tier runs. Domains: correctness, security, performance, concurrency, code quality, API design, test coverage, frontend, architecture.

## Installation

Install a skill using the `skills` CLI:

```bash
npx skills add https://github.com/rankobp/skills --skill nitpicker
```

Replace `nitpicker` with any skill name from this repository.
