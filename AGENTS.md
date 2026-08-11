# Repository Agent Guide

## Start Here

This repository is a Python 3.12 uv workspace for LangChain/LangGraph exercises.

- Workspace root: `/home/laercio_bezerra/projects/study/langgraph-exercises`
- Workspace members: `packages/*`
- Current member: `packages/langgraph-module`
- Shared lockfile: `uv.lock`
- Root project: notebook/application environment, not an installable Python package
- Package-specific guidance: `packages/langgraph-module/AGENTS.md`

Read this file for repository-wide rules. When working inside `packages/langgraph-module`, also read that package's guide; the deeper guide contains package-specific rules and does not repeat this file.

## Workspace Layout

Every future workspace member follows this structure:

```text
packages/<distribution-name>/
├── pyproject.toml
├── README.md
└── src/<python_import_name>/
```

The directory directly under `packages/` is the uv/distribution name. The import package is inside `src/` and may use underscores. Keep one root `uv.lock`; do not create member-specific lockfiles.

The root `pyproject.toml` owns dependencies needed directly by notebooks or root-level application code. A member owns dependencies needed directly by its own source imports. Avoid relying on another project's transitive dependencies.

## Standard Workflow

Run commands from the repository root unless a command explicitly selects a package:

```bash
uv sync
uv lock --check
uv tree
uv run --package <distribution-name> python -c "import <python_import_name>"
uv build --package <distribution-name>
```

After changing project metadata or dependencies:

1. Update the relevant `pyproject.toml` rather than editing `uv.lock` manually.
2. Run `uv lock`.
3. Run `uv lock --check` and `uv sync`.
4. Run the package import/resource smoke checks and build checks relevant to the change.

The repository currently has no test suite or test runner configuration. For packaging changes, use offline import, resource, workspace, lock, sync, tree, and artifact checks. Do not make live model or web-search calls as part of routine verification.

## Coding Rules

- Preserve existing LangGraph graph topology and model behavior unless the task explicitly changes behavior.
- Keep changes scoped to the requested task; do not rewrite unrelated exercises or notebooks.
- Follow existing Python style and use explicit package imports for code executed as a module.
- Keep runtime resources package-safe. Do not use paths that depend on the process working directory for packaged files.
- Do not log, print, commit, or hard-code API keys or other secrets. `.env` is local-only and ignored.
- Prefer existing dependencies and patterns. Add a dependency only when direct imports require it, and update the owning project's metadata and lockfile together.
- Do not introduce a new test framework solely for packaging work unless explicitly requested.

## Git and Commit Rules

Before committing, inspect all three independently:

```bash
git status
git diff
git log -5 --oneline
```

Create focused, atomic commits with concise subjects describing why the change is needed. Do not use interactive git flags, rewrite history, force-push, or push unless explicitly requested.

For this project:

- Never stage or commit `.specs/` content unless the user explicitly requests it. It contains planning artifacts and may remain untracked.
- Stage implementation paths explicitly instead of using broad commands that could include `.specs/`.
- Do not add `Co-authored-by`, `Developed with`, or similar collaboration trailers to commit messages.
- Do not commit generated environments, credentials, build output, or cache files.
- Confirm the final staged file list before each commit and confirm `git status` afterward.

## Safety Boundaries

Never perform destructive repository operations without explicit confirmation. This includes deleting files outside an intentional source migration, dropping data, force-pushing, rewriting history, or checking out over uncommitted work.

When a command fails, diagnose the root cause and fix the project configuration or code rather than weakening security, dependency, or reproducibility controls.
