# uv Workspace and LangGraph Package Specification

## Problem Statement

The repository currently has a single root `pyproject.toml`, while the LangGraph code lives in `langgraph_module/` without independent package metadata. This makes dependency ownership, package installation, and module execution ambiguous; it also contributes to fragile imports and prompt paths that depend on the current working directory.

Convert the repository to a uv workspace and make `langgraph_module` an independently described workspace package, while preserving the existing notebook-oriented root project and making the LangGraph code runnable through standard Python package commands.

## Goals

- [ ] The repository root is a valid uv workspace with one shared `uv.lock` file.
- [ ] `packages/langgraph-module` is a workspace member with its own `pyproject.toml` and `README.md`, and future workspace packages can be added as sibling directories under `packages/`.
- [ ] The package follows uv's normal package layout and can be installed and run from the workspace without changing directories.
- [ ] Dependencies are owned by the project that directly uses them, with root-only notebook dependencies kept out of the LangGraph package.
- [ ] Existing LangGraph examples continue to import and load their prompt resources through package-safe paths.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Publishing `langgraph_module` to PyPI | This change establishes local workspace packaging only. |
| Rewriting the LangGraph agents or changing their model behavior | The migration is packaging and execution focused. |
| Introducing a new test framework or CI pipeline | Verification uses existing tooling and uv commands unless implementation reveals an existing project convention. |
| Changing model providers, prompts, graph topology, or environment variable names | These are unrelated behavior changes. |
| Adding production deployment configuration such as `langgraph.json` | Deployment is not required to establish the workspace package. |

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Workspace structure | Keep the root as the workspace root/application and discover members through `[tool.uv.workspace] members = ["packages/*"]`; the current member shall be `packages/langgraph-module`, with future packages added as sibling directories. | uv workspace globs provide a scalable monorepo layout while requiring each immediate member directory to contain its own `pyproject.toml`. | No |
| Root packaging | Keep the root project metadata for notebooks and mark it non-package if it has no importable root package. | This preserves the existing notebook workflow without inventing a root Python package. | No |
| Member source layout | Migrate code to `packages/langgraph-module/src/langgraph_module/` while keeping package resources below the importable package. | This is uv's conventional package layout and prevents repository-root imports from masking installation problems. | No |
| Distribution name | Use a normalized distribution name such as `langgraph-module`; retain the Python import name `langgraph_module`. | The immediate directory under `packages/` shall be the exact uv/distribution name (for example, `packages/langgraph-module`), while Python import packages shall live under `src/` and may use underscores (for example, `src/langgraph_module`). | No |
| Dependency ownership | The member declares dependencies required by its Python imports; the root retains notebook-only dependencies and depends on the member with `tool.uv.sources` using `{ workspace = true }`. | Direct dependencies should be declared by the project that imports them, while the workspace provides one lockfile and editable local resolution. | No |
| Python compatibility | Keep the repository's current Python 3.12 policy and make `.python-version` resolve to a version accepted by every workspace member. | A single compatible interpreter avoids the current uv sync failure caused by an incompatible pin/requirement combination. | No |
| Runtime resources | Load prompt markdown through a package-relative/resource-safe mechanism and configure package data so installed/editable runs can access it. | Existing `open("prompts/...md")` calls depend on the caller's working directory and fail from the repository root. | No |
| Verification without credentials | Import/build/package checks must not make live model or web-search requests. | API credentials are environment-specific and should not be required to validate packaging. | No |

**Open questions:** none; defaults above are the proposed implementation choices and should be confirmed before implementation.

## User Stories

### P1: Workspace maintainer can synchronize the repository ⭐ MVP

**User Story**: As a repository maintainer, I want the root project to define a uv workspace so that all projects resolve from one lockfile and one standard workflow.

**Why P1**: Workspace metadata and a reproducible sync are the foundation for the package migration.

**Acceptance Criteria**:

1. WHEN uv reads the root `pyproject.toml` THEN it SHALL discover workspace members through `packages/*`, including `packages/langgraph-module` for this change and allowing future package directories under `packages/`.
2. WHEN a maintainer runs `uv lock` from the repository root THEN uv SHALL update one root-level `uv.lock` containing the workspace metadata and member dependency resolution.
3. WHEN a maintainer runs `uv sync` from the repository root THEN uv SHALL complete successfully using the configured Python version and install the workspace package in editable mode.
4. WHEN a maintainer runs `uv tree` from the repository root THEN the output SHALL identify the LangGraph package as a workspace-resolved project rather than a registry dependency.

**Independent Test**: Remove/recreate the environment through the supported uv workflow, run `uv lock --check`, `uv sync`, and `uv tree`, and verify all commands exit successfully with the member resolved locally.

---

### P1: LangGraph code is an independently described package ⭐ MVP

**User Story**: As a developer, I want `langgraph_module` to have its own package metadata and documentation so that it can be developed and executed independently of notebook files.

**Why P1**: Independent metadata is the requested deliverable and defines package dependency boundaries.

**Acceptance Criteria**:

1. WHEN a developer inspects `packages/langgraph-module/` THEN it SHALL contain a valid `pyproject.toml` with PEP 621 project metadata, a version, a compatible `requires-python`, a package README, and dependencies for the package's direct runtime imports; its importable Python package SHALL be under `packages/langgraph-module/src/langgraph_module/`.
2. WHEN a developer runs `uv run --package langgraph-module python -c "import langgraph_module"` THEN the import SHALL succeed from the repository root without modifying `PYTHONPATH`.
3. WHEN a developer runs `uv build --package langgraph-module` (or the uv-supported equivalent for the selected build backend) THEN a distributable artifact SHALL be created without requiring notebook dependencies.
4. WHEN the package is installed in editable or built form THEN the `langgraph_module` Python import name SHALL remain stable.
5. WHEN a developer reads `packages/langgraph-module/README.md` THEN it SHALL document the package purpose, workspace-aware setup, required environment variables at a high level, and at least one supported run/import command.

**Independent Test**: Build the member and run an import smoke test through `uv run --package`, then inspect the package README and metadata.

---

### P1: Existing examples run with package-safe imports and resources ⭐ MVP

**User Story**: As a developer, I want to run the supervisor example as a Python module from the workspace root so that execution does not depend on the current directory or sibling-module import hacks.

**Why P1**: Packaging is incomplete if the existing code only works when launched from its implementation directory.

**Acceptance Criteria**:

1. WHEN the supervisor modules import one another THEN they SHALL use package-qualified or explicit relative imports that work under `python -m langgraph_module.multi_agent.supervisor.main`.
2. WHEN the supervisor, researcher, or copywriter code loads a prompt THEN it SHALL resolve the prompt relative to package resources rather than the process working directory.
3. WHEN a package-safe import/smoke command is run from the repository root with network/model calls disabled or mocked THEN all supervisor modules SHALL import without `ModuleNotFoundError` or `FileNotFoundError` for prompts.
4. WHEN the supported supervisor module command is invoked from the repository root with valid runtime credentials THEN it SHALL start using the same graph behavior as before; no packaging change SHALL require `cd` into `prompts/` or the supervisor directory.
5. WHEN package resources are included in a built artifact THEN all three supervisor prompt files and any other runtime-read markdown resources used by the code SHALL be present and readable.

**Independent Test**: Run a no-network import/resource smoke test from the root and, when credentials are available, run the documented module command as an end-to-end manual check.

---

### P2: Root notebook workflow remains usable

**User Story**: As a learner, I want the existing root notebooks and root-level uv commands to keep their dependencies so that the workspace migration does not break the course exercises.

**Why P2**: The repository remains a study project, not only a distributable package.

**Acceptance Criteria**:

1. WHEN root notebook dependencies are audited THEN dependencies used only by notebooks remain declared by the root project rather than being moved unnecessarily into `langgraph_module`.
2. WHEN `uv run` is executed from the root for an existing notebook-oriented command THEN the command resolves the root environment and the workspace member consistently.
3. WHEN the member is selected explicitly with `uv run --package langgraph-module` THEN root-only notebook dependencies are not required for importing the member.

**Independent Test**: Run a root dependency/import smoke check and a member-only import/build check, confirming each project has the intended dependency boundary.

## Edge Cases

- WHEN `uv sync` is run with a Python interpreter that does not satisfy all workspace `requires-python` declarations THEN it SHALL fail with a clear compatibility error; the implementation SHALL not weaken requirements silently.
- WHEN a package command is launched from a directory other than the supervisor directory THEN prompt loading and package imports SHALL still work.
- WHEN the member is built into a wheel/sdist THEN required markdown prompt resources SHALL not be omitted.
- WHEN a user imports the member without API keys THEN import-time packaging checks SHALL not expose secrets, perform network calls, or fail solely because credentials are absent; live execution may still require credentials.
- WHEN the root and member declare compatible but different dependency ranges THEN uv SHALL resolve one consistent lockfile or report a dependency conflict during lock/sync rather than producing an unreproducible environment.
- WHEN the package is selected by its distribution name versus its Python import name THEN documentation SHALL distinguish `langgraph-module` (uv/distribution) from `langgraph_module` (Python import).

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| --- | --- | --- | --- |
| UVWS-01 | P1: Workspace maintainer | Design/Implementation | Pending |
| UVWS-02 | P1: Workspace maintainer | Design/Implementation | Pending |
| UVWS-03 | P1: Workspace maintainer | Design/Implementation | Pending |
| UVWS-04 | P1: Workspace maintainer | Design/Implementation | Pending |
| UVPK-01 | P1: Independently described package | Design/Implementation | Pending |
| UVPK-02 | P1: Independently described package | Design/Implementation | Pending |
| UVPK-03 | P1: Independently described package | Design/Implementation | Pending |
| UVPK-04 | P1: Independently described package | Design/Implementation | Pending |
| UVPK-05 | P1: Independently described package | Design/Implementation | Pending |
| UXRUN-01 | P1: Package-safe execution | Design/Implementation | Pending |
| UXRUN-02 | P1: Package-safe execution | Design/Implementation | Pending |
| UXRUN-03 | P1: Package-safe execution | Design/Implementation | Pending |
| UXRUN-04 | P1: Package-safe execution | Design/Implementation | Pending |
| UXRUN-05 | P1: Package-safe execution | Design/Implementation | Pending |
| UVROOT-01 | P2: Root workflow | Design/Implementation | Pending |
| UVROOT-02 | P2: Root workflow | Design/Implementation | Pending |
| UVROOT-03 | P2: Root workflow | Design/Implementation | Pending |

**Coverage**: 17 total, 17 mapped to stories and implementation tasks to be created, 0 unmapped.

## Proposed Target Layout

```text
.
├── pyproject.toml                 # workspace root and notebook/application metadata
├── README.md                      # repository/course documentation
├── .python-version                # one compatible Python 3.12 pin
├── uv.lock                        # single workspace lockfile
├── notebooks/
└── packages/
    ├── langgraph-module/          # uv/distribution name; workspace member
    │   ├── pyproject.toml
    │   ├── README.md
    │   └── src/
    │       └── langgraph_module/ # Python import package name
    │           ├── __init__.py
    │           └── multi_agent/
    │               └── supervisor/
    │                   ├── prompts/
    │                   ├── example_content/
    │                   ├── ai_files/
    │                   └── *.py
    └── <future-distribution-name>/
        ├── pyproject.toml
        ├── README.md
        └── src/
            └── <python_import_name>/
```

The implementation may preserve neither the current repository-level source location nor a package directory directly under the workspace root. Every workspace package SHALL use the pattern `packages/<uv-or-distribution-name>/pyproject.toml` with its Python import package below `packages/<uv-or-distribution-name>/src/<python_import_name>/`; the default target is the conventional `src/` layout above.

## Verification Plan

The implementation MUST document and run, as applicable:

1. `uv lock --check` from the repository root.
2. `uv sync` from the repository root using the repository's compatible Python pin.
3. `uv tree` to verify workspace resolution and dependency ownership.
4. `uv run --package langgraph-module python -c "import langgraph_module"`.
5. A package-resource smoke check that reads the supervisor prompt files from the installed/editable package without changing directories.
6. A build check for the member and inspection of the resulting artifact to confirm required markdown resources are included.
7. A root-level module/import smoke check that does not invoke external model or search services.
8. A manual live supervisor run only when the developer supplies valid credentials; this is not a required offline gate.

## Success Criteria

- [ ] A fresh developer can run the documented root uv workflow without the current Python compatibility or missing-prompt failure.
- [ ] `langgraph_module` has independent metadata, dependencies, documentation, and a stable import path.
- [ ] The workspace has one reproducible lockfile and local member resolution.
- [ ] The supervisor package can be imported and its prompt resources loaded from the repository root and from a built/installable artifact.
- [ ] Notebook dependencies and package runtime dependencies are separated according to direct usage.
