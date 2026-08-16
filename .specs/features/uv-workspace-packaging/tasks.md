# uv Workspace and LangGraph Package Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `spec-driven` skill and follow its Execute flow and Critical Rules. Each task must satisfy its requirement-linked done criteria and pass its stated gate before it is marked complete. Do not batch unrelated changes or weaken verification gates.

**Spec**: `.specs/features/uv-workspace-packaging/spec.md`
**Design**: `.specs/features/uv-workspace-packaging/design.md`
**Status**: Draft

---

## Test Coverage Matrix

> Generated from repository inspection and the approved spec. No `AGENTS.md`, contributor testing guide, test configuration, or test files were found. Because this feature is packaging/configuration focused, the matrix uses offline command-based smoke and build gates rather than introducing a new test framework.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Workspace/project metadata | none | Valid TOML, valid uv workspace discovery, compatible Python metadata | `pyproject.toml`, `packages/*/pyproject.toml`, `.python-version` | `uv lock --check`, `uv sync`, `uv tree` |
| Package import/resource wiring | package smoke | Import package from root; exercise all supervisor module imports in scope; read all three prompt resources without cwd dependence; no network calls | `packages/langgraph-module/src/**/*.py` and packaged resources | `uv run --package langgraph-module python -c ...` from root and a non-supervisor cwd |
| Distribution artifact | package build/integration | Build member and verify Python files plus all runtime-read markdown resources are present in wheel/sdist | `packages/langgraph-module/`, `dist/` | `uv build --package langgraph-module` plus archive inspection |
| Documentation | none | Package README contains setup, package/import naming, environment, and run instructions | `packages/langgraph-module/README.md` | Markdown/content inspection |

## Gate Check Commands

> Generated from repository inspection and the spec. These are provisional until task approval.

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | After metadata-only or documentation tasks | `uv lock --check` when metadata exists; otherwise structural inspection |
| Package smoke | After source/resource/import tasks | `uv run --package langgraph-module python -c "import langgraph_module"` plus prompt-resource smoke command |
| Full | After dependency and workspace integration tasks | `uv lock --check && uv sync && uv tree && uv build --package langgraph-module` |
| Artifact | After build/package-data changes | Inspect wheel/sdist contents and read packaged prompt files without changing directories |

---

## Execution Plan

Phases are ordered and run sequentially. Tasks within a phase execute in order.

### Phase 1: Workspace and Package Foundation

```text
T1 → T2 → T3
```

### Phase 2: Source Migration and Runtime Wiring

```text
T4 → T5 → T6
```

### Phase 3: Dependency Integration and Verification

```text
T7 → T8
```

## Task Breakdown

### T1: Configure the root uv workspace

**What**: Update the root `pyproject.toml` to define `[tool.uv.workspace] members = ["packages/*"]`, preserve root notebook/application metadata, configure root package mode appropriately, and declare the local member through `[tool.uv.sources]` when required by root imports.

**Where**: `pyproject.toml`

**Depends on**: None

**Reuses**: Existing root project metadata and dependencies.

**Requirements**: UVWS-01, UVROOT-02

**Tools**:

- MCP: NONE
- Skill: `langchain-dependencies` for dependency ownership review

**Done when**:

- [ ] The root contains the workspace glob `packages/*`.
- [ ] The root remains a valid uv project for notebook commands.
- [ ] Root configuration does not require a nonexistent root import package.
- [ ] The member source declaration, if needed by root usage, uses `{ workspace = true }` rather than a registry source.
- [ ] Gate passes: structural TOML inspection.

**Tests**: none — metadata/configuration layer
**Gate**: quick

---

### T2: Create the LangGraph member metadata

**What**: Create `packages/langgraph-module/pyproject.toml` with distribution name `langgraph-module`, PEP 621 metadata, compatible Python requirements, a uv-supported build backend, source-layout configuration, package-data configuration, and the package's direct runtime dependencies.

**Where**: `packages/langgraph-module/pyproject.toml`

**Depends on**: T1

**Reuses**: Existing dependency declarations and imports under `langgraph_module/`.

**Requirements**: UVPK-01, UVPK-04, UXRUN-05, UVROOT-03

**Tools**:

- MCP: NONE
- Skill: `langchain-dependencies`

**Done when**:

- [ ] The immediate package directory is exactly `packages/langgraph-module`.
- [ ] The distribution name and Python import name are documented/configured distinctly.
- [ ] Direct imports used by the moved package are declared by the member rather than relying only on root transitive dependencies.
- [ ] Markdown prompt resources are covered by the selected build backend's package-data rules.
- [ ] `requires-python` is compatible with the root policy.
- [ ] Gate passes: metadata parses and uv recognizes the member.

**Tests**: none — package metadata/configuration layer
**Gate**: quick

---

### T3: Document member setup and execution

**What**: Create the member README with workspace setup, the distinction between `langgraph-module` and `langgraph_module`, environment-variable guidance without secrets, offline import verification, and the supported supervisor module command.

**Where**: `packages/langgraph-module/README.md`

**Depends on**: T2

**Reuses**: Commands and behavior defined in the spec and design.

**Requirements**: UVPK-05, UXRUN-04, UVROOT-02

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] README explains root-level `uv sync` and member-selected `uv run` usage.
- [ ] README distinguishes distribution/uv name `langgraph-module` from import name `langgraph_module`.
- [ ] README documents required environment variables at a high level without including credentials.
- [ ] README documents `python -m langgraph_module.multi_agent.supervisor.main` through uv.
- [ ] Gate passes: content inspection.

**Tests**: none — documentation layer
**Gate**: quick

---

### T4: Move the source and runtime resources into the src layout

**What**: Move the existing `langgraph_module` package and its supervisor resources into `packages/langgraph-module/src/langgraph_module/`, preserving Python files, `__init__.py` files, prompts, example content, and AI-generated markdown content without behavior edits.

**Where**: `langgraph_module/**` → `packages/langgraph-module/src/langgraph_module/**`

**Depends on**: T2, T3

**Reuses**: Existing source/resource contents.

**Requirements**: UVPK-01, UVPK-04, UXRUN-05

**Tools**:

- MCP: NONE
- Skill: NONE

**Done when**:

- [ ] No implementation source remains in the old repository-level `langgraph_module/` location.
- [ ] The import package is located below `packages/langgraph-module/src/`.
- [ ] All three supervisor prompt files remain present with unchanged content.
- [ ] Existing package subpackages retain their `__init__.py` files.
- [ ] Gate passes: file inventory comparison and package layout inspection.

**Tests**: package smoke — run the resource/import smoke gate after the move
**Gate**: package smoke

---

### T5: Make supervisor imports package-qualified

**What**: Update sibling imports in the supervisor entrypoint and agent modules to explicit relative or fully qualified imports that work when the application is launched with `python -m langgraph_module.multi_agent.supervisor.main`.

**Where**: `packages/langgraph-module/src/langgraph_module/multi_agent/supervisor/*.py`

**Depends on**: T4

**Reuses**: Existing module interfaces and graph objects.

**Requirements**: UXRUN-01, UXRUN-03

**Tools**:

- MCP: NONE
- Skill: `python-patterns`

**Done when**:

- [ ] `supervisor.py` no longer relies on top-level sibling imports.
- [ ] `main.py` imports the supervisor graph through the package namespace.
- [ ] Importing all supervisor modules from the repository root produces no `ModuleNotFoundError`.
- [ ] No graph topology or model behavior is changed.
- [ ] Gate passes: offline package import smoke command with external calls disabled/mocked as needed.

**Tests**: package smoke — all supervisor imports in scope
**Gate**: package smoke

---

### T6: Make prompt and package-resource loading cwd-independent

**What**: Replace process-relative prompt reads with package-relative/resource-safe loading and ensure the build configuration includes every markdown resource read at runtime.

**Where**: `packages/langgraph-module/src/langgraph_module/multi_agent/supervisor/{researcher.py,supervisor.py,copywriter.py}` and member build metadata if needed

**Depends on**: T4, T5

**Reuses**: Existing prompt files and their current formatting variables.

**Requirements**: UXRUN-02, UXRUN-03, UXRUN-05

**Tools**:

- MCP: NONE
- Skill: `python-patterns`

**Done when**:

- [ ] Prompt lookup does not depend on the process working directory.
- [ ] All three supervisor prompts are readable from the editable package.
- [ ] The smoke command succeeds when run from the repository root and from another working directory.
- [ ] Package resource loading does not perform network calls or expose secrets.
- [ ] Gate passes: package-resource smoke command.

**Tests**: package smoke — all prompt resources and cwd variants
**Gate**: package smoke

---

### T7: Finalize dependency ownership and regenerate the workspace lockfile

**What**: Audit direct imports in the moved package, move only package runtime dependencies into the member metadata, retain notebook-only dependencies at the root, align `.python-version` and `requires-python`, and regenerate the single root `uv.lock`.

**Where**: `pyproject.toml`, `packages/langgraph-module/pyproject.toml`, `.python-version`, `uv.lock`

**Depends on**: T6

**Reuses**: Existing dependency versions and the current lockfile resolution where compatible.

**Requirements**: UVWS-02, UVWS-03, UVWS-04, UVROOT-01, UVROOT-03

**Tools**:

- MCP: NONE
- Skill: `langchain-dependencies`

**Done when**:

- [ ] The package is independently resolvable without root-only notebook dependencies.
- [ ] Root direct notebook imports retain appropriate root declarations.
- [ ] The Python pin is accepted by every workspace project.
- [ ] `uv lock` completes and produces a single root lockfile with workspace members.
- [ ] Gate passes: `uv lock --check && uv sync && uv tree`.

**Tests**: none — dependency/lock configuration layer; full uv gate required
**Gate**: full

---

### T8: Verify built artifacts and end-to-end workspace commands

**What**: Run the complete offline verification plan, build the member, inspect wheel/sdist contents for Python files and all runtime markdown resources, and record any implementation adjustments needed to satisfy the spec.

**Where**: Workspace root and `packages/langgraph-module/`

**Depends on**: T7

**Reuses**: All workspace/package artifacts and commands from the spec.

**Requirements**: UVWS-02, UVWS-03, UVPK-02, UVPK-03, UVPK-04, UXRUN-03, UXRUN-04, UXRUN-05, UVROOT-02

**Tools**:

- MCP: NONE
- Skill: `spec-driven`

**Done when**:

- [ ] `uv lock --check` passes.
- [ ] `uv sync` passes using the documented Python version.
- [ ] `uv tree` identifies the member as workspace-resolved.
- [ ] `uv run --package langgraph-module python -c "import langgraph_module"` passes.
- [ ] The prompt-resource smoke check passes from a non-supervisor working directory.
- [ ] `uv build --package langgraph-module` passes.
- [ ] Built artifacts contain the import package and all three supervisor prompts.
- [ ] The documented module command is recorded for credentialed manual execution; offline verification does not call external services.

**Tests**: package smoke/integration and artifact inspection
**Gate**: full + artifact

---

## Phase Execution Map

```text
Phase 1 → Phase 2 → Phase 3

Phase 1: T1 ──→ T2 ──→ T3

Phase 2: T4 ──→ T5 ──→ T6

Phase 3: T7 ──→ T8
```

Cross-phase dependency rule: Phase 2 starts after the package metadata and README exist; T5 requires the migrated source from T4, and T6 requires both the migrated source and repaired imports. T7 waits for all source/runtime wiring because dependency audit and lock resolution must reflect the final package contents.

## Task Granularity Check

| Task | Scope | Status |
| --- | --- | --- |
| T1 | Root workspace metadata | Granular |
| T2 | Member package metadata | Granular |
| T3 | Member README | Granular |
| T4 | Source/resource relocation | Granular, one cohesive migration |
| T5 | Package import repair | Granular |
| T6 | Package resource loading | Granular |
| T7 | Dependency/lock finalization | Granular, one cohesive dependency boundary |
| T8 | Final verification and artifact inspection | Granular verification gate |

## Diagram-Definition Cross-Check

| Task | Depends On (task body) | Diagram Shows | Status |
| --- | --- | --- | --- |
| T1 | None | Phase 1 start | Match |
| T2 | T1 | T1 → T2 | Match |
| T3 | T2 | T2 → T3 | Match |
| T4 | T2, T3 | T2 → T3 → T4 | Match |
| T5 | T4 | T4 → T5 | Match |
| T6 | T4, T5 | T4 → T5 → T6 | Match |
| T7 | T6 | T6 → T7 | Match |
| T8 | T7 | T7 → T8 | Match |

## Test Co-location Validation

| Task | Code Layer Created/Modified | Matrix Requires | Task Says | Status |
| --- | --- | --- | --- | --- |
| T1 | Workspace metadata | none | none + quick gate | OK |
| T2 | Package metadata | none | none + quick gate | OK |
| T3 | Documentation | none | none + quick gate | OK |
| T4 | Package layout/resources | package smoke | package smoke | OK |
| T5 | Package import wiring | package smoke | package smoke | OK |
| T6 | Package resource wiring | package smoke | package smoke | OK |
| T7 | Dependency/lock metadata | none | none + full uv gate | OK |
| T8 | Distribution artifact and integration | package smoke/integration | package smoke/integration + artifact inspection | OK |

## MCP and Skill Selection

Before execution, confirm tool choices for each task. Proposed defaults:

| Task | MCP | Skill |
| --- | --- | --- |
| T1 | NONE | `langchain-dependencies` |
| T2 | NONE | `langchain-dependencies` |
| T3 | NONE | NONE |
| T4 | NONE | NONE |
| T5 | NONE | `python-patterns` |
| T6 | NONE | `python-patterns` |
| T7 | NONE | `langchain-dependencies` |
| T8 | NONE | `spec-driven` |
