# KiCad 9 Migration Plan (Docker → Local)

**Date:** 2026-01-12
**Owner:** Shaurya Sethi

## Context & Current State
- Circuitron agents run SKiDL and ERC inside a KiCad 5 Docker image (`docker_session.py`, `tools.py` execution helpers, `agents.py` model settings with `parallel_tool_calls=False`).
- Tools using Docker today: `search_kicad_libraries`, `search_kicad_footprints`, `extract_pin_details`, `run_runtime_check`, `run_erc`, `execute_final_script`, plus ephemeral calc container in `execute_calculation`.
- Docker-specific concerns: Windows path conversion (`convert_windows_path_for_docker`), `/tmp` collisions, container health checks, retry logic, and mounted output copy (`copy_generated_files`).
- Outputs are copied from container to host (`circuitron_output/`), and environment variables target KiCad 5 paths (`KICAD5_SYMBOL_DIR`, `KICAD5_FOOTPRINT_DIR`, `KISYSMOD`).
- Tests and CI rely on Docker availability; progress notes highlight container cleanup and serialization to avoid race conditions.

## Goals
- Enable KiCad 9 + SKiDL execution on a locally installed toolchain (no Docker dependency when running locally).
- Preserve existing pipeline behavior (part search, runtime checks, ERC, final artifact generation) with minimal UX disruption.
- Provide two viable strategies:
  1. **Local-only migration** (drop Docker completely).
  2. **Dual-mode** (runtime-selectable local KiCad 9 or Docker fallback for legacy/CI).
- Keep tests reliable and fast via mocks/fakes; avoid requiring KiCad in CI until explicitly supported.

## Non-Goals
- Not defining the exact KiCad 9 library env vars or install layout yet—must be confirmed from official KiCad/SKiDL docs.
- Not changing agent prompts or outputs beyond what is required to support the new execution path.
- Not redesigning the UI/CLI beyond necessary switches and health checks.

## Cross-Cutting Unknowns / Risks to Resolve Early
- Exact KiCad 9 env variables and SKiDL defaults (e.g., `KICAD_SYMBOL_DIR`, footprint envs) and how SKiDL discovers libraries without Docker-provided paths.
- SKiDL version compatibility with KiCad 9 symbols/footprints; whether new symbols change naming that breaks searches.
- Windows vs. macOS vs. Linux installation paths; permissions for writing caches/temp files.
- ERC/runtime behavior differences between KiCad 5 vs 9 (warnings/errors semantics, API changes).
- Performance regressions when loading local libraries; need caching or lazy loads.

## Approach A — Full Local Migration (KiCad 9 Native)

### High-Level Strategy
- Remove Docker coupling and execute SKiDL/KiCad operations directly on the host, assuming KiCad 9 + SKiDL are installed.
- Replace `DockerSession` usages with a **LocalKiCadSession** abstraction that sets env vars, runs Python, and writes artifacts directly to the filesystem.
- Update configuration, prompts, and tooling to require local KiCad 9; deprecate Docker images.

### Work Breakdown
1. **Environment discovery & bootstrap**
   - Add a local KiCad checker (new helper in `utils.py` or `settings.py`) to locate KiCad 9 binaries and library paths; expose `settings.kicad_mode = "local"`.
   - Provide clear error messaging in CLI/setup if KiCad 9 or SKiDL is missing; suggest install steps per OS.
   - Decide on env vars to set before SKiDL import (confirm via KiCad 9 docs); update `setup_environment` to optionally load `.env` overrides.

2. **Execution engine refactor**
   - Introduce `kiCad_runner.py` (new module) with an interface (e.g., `KiCadRunner` protocol) and implement `LocalKiCadRunner` using subprocess or in-process execution.
   - Update tools to call the runner instead of `DockerSession`:
     - Searches (`search_kicad_libraries`, `search_kicad_footprints`, `extract_pin_details`)
     - Validation (`run_runtime_check`, `run_erc`)
     - Final execution (`execute_final_script`) — remove container mounts; write directly to target output directory.
   - Remove Windows-specific Docker copy/retry logic; retain temp-file hygiene.

3. **Library path management**
   - Define a single helper to compute KiCad 9 symbol/footprint env vars; allow overrides in config.
   - Ensure tools call `set_default_tool(<KiCad 9 enum>)` if SKiDL exposes a new constant; otherwise rely on `set_default_tool(KICAD)` and document fallback.

4. **Agent and pipeline adjustments**
   - Remove `parallel_tool_calls=False` constraints if no shared container state remains (verify thread safety of local SKiDL calls; may still serialize to avoid global state collisions in SKiDL).
   - Update prompts to remove Docker path guidance; ensure runtime/erc prompts still ask for `script_content` or host paths only.

5. **Testing & CI**
   - Add fakes/mocks for `KiCadRunner` so unit tests don’t require KiCad; keep behavior/contract tests.
   - Update existing Docker-specific tests to cover local runner paths and path handling.
   - Gate integration tests that require real KiCad behind an opt-in flag (e.g., `CIRCUITRON_E2E_KICAD=1`).

6. **Cleanup & docs**
   - Remove Docker image settings (`kicad_image`, Docker cleanup helpers) and unused code paths.
   - Update README/SETUP to document KiCad 9 installation, env vars, troubleshooting.
   - Prune `docker_session.py` or archive it if fully removed.

### Rollout / Migration Steps
- Phase 1: Introduce runner interface + Local runner behind a feature flag; keep Docker for safety but unused in default path.
- Phase 2: Switch default to Local; deprecate Docker code; update docs.
- Phase 3: Delete Docker-specific code/tests after stabilization.

### Acceptance Criteria
- Pipeline runs end-to-end on a machine with KiCad 9 + SKiDL installed, producing schematics/netlists/PCBs without Docker.
- All unit tests pass without requiring Docker; optional KiCad E2E suite documented.
- Clear failure modes when KiCad/SKiDL are missing or misconfigured.

## Approach B — Dual-Mode (Local KiCad 9 + Docker Fallback)

### High-Level Strategy
- Keep Docker-backed execution for CI/legacy while adding a pluggable local runner. Users choose via config/CLI/env; auto-detect local KiCad and fall back to Docker if unavailable.

### Work Breakdown
1. **Runner abstraction**
   - Define `KiCadRunner` interface with operations: `exec_python(code)`, `exec_script(path)`, `exec_erc(script_path, wrapper)`, `copy_outputs(pattern, dest)` (where applicable).
   - Implement `LocalKiCadRunner` and adapt existing `DockerSession` into `DockerKiCadRunner` (wrapper over current class) without changing callers.

2. **Configuration & selection**
   - Add `settings.kicad_mode` (`auto` | `local` | `docker`) and optional `KICAD_LOCAL_BIN`, `KICAD_SYMBOL_DIR`, `KICAD_FOOTPRINT_DIR` overrides.
   - Selection logic: try local when mode is `auto` and KiCad 9 is detected; else use Docker (with clear warning if image missing).

3. **Tool refactor**
   - Inject runner into tools (`search_*`, `run_erc`, `run_runtime_check`, `execute_final_script`) via module-level singleton or dependency injection.
   - Ensure footprint/symbol search scripts branch on env vars appropriate to KiCad 9 vs Docker 5; parameterize env setup strings.
   - Preserve container copy logic only inside `DockerKiCadRunner`; local runner writes directly to filesystem.

4. **Pipeline & agent updates**
   - Keep `parallel_tool_calls=False` when runner is Docker; allow configurable concurrency when runner is local (after confirming SKiDL thread safety).
   - Update prompts to avoid container path references; maintain guidance to pass `script_content`.

5. **Testing strategy**
   - Mock `KiCadRunner` in unit tests; add contract tests that run against both runner implementations using fixtures.
   - Keep Docker-based integration tests for CI environments where the image is available; add skip markers when Docker not present.

6. **Docs & UX**
   - Document mode selection, env requirements, and troubleshooting matrix (local vs Docker).
   - Provide CLI flags (e.g., `--kicad-mode local|docker|auto`) and surface mode in UI status.

### Acceptance Criteria
- Users can choose mode and see clear status; default `auto` prefers KiCad 9 local when available.
- Both runners pass the same contract tests for searches, runtime checks, ERC, and artifact generation (within version-difference tolerance).
- CI can still run with Docker even if KiCad 9 is absent on the runner.

## Shared Tasks & Recommendations
- Create a compatibility matrix (OS × mode × KiCad version × SKiDL version) and track verified combinations.
- Add telemetry/logs to report selected runner and env paths for supportability.
- Establish a small golden-design regression suite (inputs → expected ERC/runtime outcomes) to compare KiCad 5 vs 9 behavior.
- Provide migration playbook for users: prerequisites, config changes, rollback steps.

## Open Questions
- Which KiCad 9 environment variables does SKiDL honor by default? Need confirmation from official docs.
- Are there SKiDL API changes between versions bundled with KiCad 5 vs 9 that affect `search`/`search_footprints` parsing?
- Do ERC warning/error strings change in KiCad 9, requiring parser updates in `run_erc`?
- Should we keep the calculation helper in Docker or also move it local for consistency and speed?

## Suggested Next Steps (immediate)
1. Research KiCad 9 + SKiDL env requirements; draft env helper API.
2. Spike a `KiCadRunner` interface with a stub `LocalKiCadRunner` that simply executes a trivial SKiDL script to validate environment detection.
3. Decide rollout strategy (Approach A vs B) based on CI constraints and user install expectations.
4. Update documentation outline and test strategy accordingly.
