# KiCad 9 migration plan doc drafted (12-01-2026)

## Summary
- Added a detailed plan outlining two strategies to move Circuitron from a KiCad 5 Docker-only setup to KiCad 9 local support: (A) full local migration and (B) dual-mode local/Docker.
- Captured current-state dependencies (DockerSession, tools, agent settings), risks, unknowns, acceptance criteria, and recommended next steps.

## Files Changed
- `docs/kicad9-migration-plan.md`

## Rationale
- KiCad 5 is outdated; we need a vetted refactor plan to guide engineers through migrating to KiCad 9 with either a clean local-only path or a dual-mode path that preserves Docker for CI/legacy use.

## Verification
- Docs-only change. No code executed; tests not run.

## Issues
- Environment variable names and SKiDL compatibility for KiCad 9 remain to be confirmed from official docs.

## Next Steps
- Research KiCad 9 + SKiDL environment requirements and update the plan with confirmed values.
- Decide on Approach A vs B based on CI constraints and user install expectations.
