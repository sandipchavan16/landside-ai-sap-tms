# Copilot Cloud Agent Onboarding Instructions

## Repository purpose and scope
- This repository currently contains **instruction documents** for SAP TM ABAP development agents, not executable application code.
- Primary source files:
  - `/home/runner/work/Landside-AI-SAP-TMS/Landside-AI-SAP-TMS/ABAP_CodingPractices.instructions.md`
  - `/home/runner/work/Landside-AI-SAP-TMS/Landside-AI-SAP-TMS/NamingConventions.instructions.md`
  - `/home/runner/work/Landside-AI-SAP-TMS/Landside-AI-SAP-TMS/BOPF_Changes.instructions.md`

## First-read order (mandatory)
1. `ABAP_CodingPractices.instructions.md`
2. `NamingConventions.instructions.md`
3. `BOPF_Changes.instructions.md`
4. `README.md`

Use this order before proposing ABAP code or naming decisions.

## Non-negotiable coding expectations
- Use modern Clean ABAP patterns (ABAP 7.40+ style).
- Follow repository naming conventions exactly (`Z` transportable, `Y` local; module and criticality codes).
- For TM BOPF-managed BO data:
  - Never read/write BO tables directly with SQL.
  - Use Service Manager APIs (`QUERY`, `RETRIEVE`, `RETRIEVE_BY_ASSOCIATION`, `DO_ACTION`, `MODIFY`) and Transaction Manager `save`.
  - Never use `COMMIT WORK` in TM enhancement logic.
- Prefer `DO_ACTION`; use `MODIFY` only when no action exists.
- Build mass operations outside loops (no per-record BOPF API calls in loops).

## Naming and design guardrails
- Ask for missing module code (`<MD>`) when required for naming.
- Keep identifiers in snake_case and self-describing English names.
- Enforce class-based exceptions and explicit error handling.
- Avoid macros, obsolete ABAP statements, and magic literals.
- Keep business logic out of event blocks/dialog modules; delegate to classes.

## Modification log expectations
- When generating ABAP changes, include/update mod-log headers and BOC/EOC markers as described in `ABAP_CodingPractices.instructions.md` section 9.

## Validation expectations
- Run available repo checks before and after changes.
- Current repository has no configured linter/build/test tooling files (no `package.json`, `pyproject.toml`, `pom.xml`, `Makefile`, or workflows found), so validation is primarily document consistency and instruction alignment.

## Known errors encountered in this repository and workarounds
1. **Error:** `.github` directory did not exist.
   - **Workaround:** Create `.github` before adding agent guidance files.
2. **Error:** Large instruction files may exceed single-read limits in some tooling.
   - **Workaround:** Read in sections (range/chunked reads) or search headings first, then open relevant ranges.
3. **Error:** No runnable test/build/lint automation discovered.
   - **Workaround:** Perform strict manual cross-check against the three instruction documents and keep changes minimal and documentation-focused.

## Change strategy for future agents
- Keep updates surgical and limited to requested scope.
- If adding new guidance, ensure it does not conflict with the three instruction documents; those documents take precedence.
- When conflicts appear, explicitly call out the conflict and propose the conservative option aligned with SAP TM BOPF safety and existing naming rules.
