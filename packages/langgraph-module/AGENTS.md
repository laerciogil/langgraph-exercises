# `langgraph-module` Agent Guide

## Scope

This guide applies to the `langgraph-module` workspace member only. Read the repository-root `AGENTS.md` first for workspace, dependency, verification, safety, and git rules.

The member has:

- Distribution/uv name: `langgraph-module`
- Python import name: `langgraph_module`
- Source root: `packages/langgraph-module/src/langgraph_module/`
- Build backend: Hatchling
- Runtime resources: supervisor prompts, example content, and AI markdown files under the Python package

## Package Commands

From the repository root, use the package selector for member-specific commands:

```bash
uv run --package langgraph-module python -c "import langgraph_module"
uv run --package langgraph-module python -m langgraph_module.multi_agent.supervisor.main
uv build --package langgraph-module
```

The supervisor command requires provider credentials in the environment. Offline imports, resource checks, and builds must not require credentials or make external API calls.

## Source and Resource Rules

- Add Python code below `src/langgraph_module/`; do not place importable implementation at the member root.
- Preserve the distinction between the distribution name (`langgraph-module`) and import name (`langgraph_module`).
- Use explicit relative or fully qualified imports for sibling modules. Code must work with `python -m langgraph_module...`, not only when launched from the implementation directory.
- Load prompts and other packaged markdown with package-relative/resource-safe access, such as `importlib.resources`. Never assume the current working directory is the supervisor directory.
- When adding a runtime resource, keep it under the appropriate package directory and update Hatchling package-data/build configuration if needed.
- Preserve prompt content and existing graph behavior unless the task explicitly requests a behavior change.
- Avoid import-time network calls. Constructing graph/model objects may occur at import time because of existing code, but verification must not invoke model or search operations.

## Dependency Rules

Declare a dependency in this member's `pyproject.toml` when source code imports it directly. Keep notebook-only dependencies in the workspace root. Do not depend on a package merely because it happens to be installed by the root project.

After changing member dependencies or build metadata:

```bash
uv lock
uv lock --check
uv sync
uv tree
```

Do not edit `uv.lock` by hand. Keep the member's Python requirement compatible with the root `.python-version` and all other workspace members.

## Package Verification

For source, import, or resource changes, verify from the repository root and from a different working directory:

```bash
uv run --package langgraph-module python -c "import langgraph_module"
```

Also import the supervisor modules without invoking external services and verify all runtime prompts are readable from package resources. For distribution changes, build the member and inspect both wheel and sdist contents for:

- `langgraph_module/__init__.py`
- Supervisor Python modules
- `prompts/researcher.md`
- `prompts/copywriter.md`
- `prompts/supervisor.md`
- Any newly added runtime markdown resources

A credentialed supervisor run is a manual check only; never add credentials to source, documentation, logs, or commits.

## Documentation Expectations

Keep `README.md` accurate when setup, commands, environment variables, package names, or supported execution paths change. Document both names when referring to the package so agents do not confuse the uv distribution name with the Python import name.
