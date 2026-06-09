# ABAP Coding Practices — Agent Instructions

> Governs how you write/review ABAP. Apply every rule; when you skip one (e.g. legacy constraint), say so and why.
> Names → `NamingConventions.instructions.md`. SAP TM/BOPF → `BOPF_Changes.instructions.md`.

## 0. Agent Contract
- Apply all rules below to every snippet you produce or review.
- **Default to ABAP 7.40+ syntax**: inline `DATA(...)`, `NEW`/`VALUE`/`CONV`/`COND`/`SWITCH`, table expressions, string templates `|...|`. Use older syntax only if the user states an older release, and flag it.
- Comment **why**, never **how** (§2.3).
- After non-trivial code, append the §10 checklist confirming what was applied.

## 1. Style & Clean ABAP
- **Pretty-Printer**: 2-space indent, keywords UPPER, operands lower.
- **snake_case, English-only** identifiers. `run_document_processing( )` not `RunDocProc( )`.
- **No type-encoding prefixes** (`lv_`/`lt_`/`ls_`/`lo_`/`lr_`/`lc_`/`gv_`). Name for *what it is*: `order`, `orders`, `order_count`, `document_processor`.
  - *Kept (role, not type)*: parameter `i_`/`e_`/`c_`/`r_`/`t_`, screen `p_`/`s_`/`c_`/`r_`/`b_`, constant `co_` — per `NamingConventions`.
- **Symbolic comparison operators** only (`=`, `<>`, `>=`…), never alphabetic (`EQ`, `NE`…); don't mix.

### 1.1 No obsolete statements — use the modern form (and stay clean against [SAP/abap-cleaner](https://github.com/SAP/abap-cleaner))
| Obsolete | Modern | abap-cleaner rule |
|---|---|---|
| `MOVE a TO b` | `b = a` | Replace obsolete MOVE … TO with `=` |
| `ADD 1 TO n` | `n += 1` | Replace obsolete ADD … TO with `+=` |
| `WRITE TO` / `CONCATENATE` | string templates `\|...\|` | Use string templates to assemble text |
| `TABLE … WITH HEADER LINE`, `TABLES`, `NODES` | explicit table + WA/field symbol | — |
| `CALL FUNCTION … EXCEPTIONS` | class-based exceptions | — |
| `CALL METHOD o->m` | `o->m( )` | Replace CALL METHOD with functional call |
| `CREATE OBJECT r` | `r = NEW #( )` | Replace CREATE OBJECT with NEW |
| `RAISE EXCEPTION TYPE …` | `RAISE EXCEPTION NEW …` | Replace RAISE … TYPE with RAISE … NEW |
| `DESCRIBE TABLE t LINES n` | `n = lines( t )` | Replace DESCRIBE TABLE … LINES |
| `READ TABLE … INTO` (single hit) | `t[ … ]` | Replace READ TABLE with table expression |
| `EXIT` outside loop | `RETURN` | Replace EXIT outside loop with RETURN |

Also honour: `IS NOT` over `NOT … IS`; omit optional `EXPORTING`/`RECEIVING`; remove `me->` when unambiguous; unchain `DATA:` lists; `FINAL` for immutables; delete unused vars; comment with `"` not `*`; CamelCase for CDS names.

## 2. General Rules
- **2.1 Separation of concerns** — UI / Controller / Business Logic / Data Access. Event blocks & dialog modules contain no logic; they delegate to a class method.
- **2.2 Self-describing code** — names read like prose; decompose large methods into small named ones.
- **2.3 Commenting** — comment the *business reason* (ideally in the method header), never *how*. Good: `" Clearing docs use company-specific type per FI policy §4.2`. Bad: `" Call change_document`.
- **2.4 Local scope** — declare at the lowest scope; locals > attributes > globals; pass data via parameters, never globals.
- **2.5 No magic values** — name them as `co_` constants describing business meaning.
- **2.6 Max nesting depth 3** — flatten with early exits (`RETURN`/`CONTINUE`/`CHECK`):
```abap
LOOP AT data ASSIGNING FIELD-SYMBOL(<entry>).
  IF a <> b.   <entry>-field = 'ccc'. CONTINUE. ENDIF.
  IF c <> d.   <entry>-field = 'bbb'. CONTINUE. ENDIF.
  CHECK <entry> IS NOT INITIAL.
  <entry>-field = 'aaa'.
ENDLOOP.
```
- **2.7 No dead code** — no commented-out blocks, unused vars, unreachable branches. Verify with SLIN / ATC.
- **2.8 Never ignore errors** — every failable op raises/logs/messages. Always check `sy-subrc` after DB ops; if no action needed, document it: `" missing entry is valid per business rule`.
- **2.9 Class-based exceptions only** (`CX_*`/`ZCX_*`); `RAISE EXCEPTION NEW`; one class per problem type; handle close to the raise.
- **2.10 Booleans** — `abap_bool` + `abap_true`/`abap_false`, never `c LENGTH 1` or `'X'`.
- **2.11 Explicit, typed declarations** — prefer inline `DATA(...)`; no `TABLES`/`NODES`/header lines.
- **2.12 Exit with `RETURN`** only (not `CHECK`/`EXIT`).
- **2.13 Thin dialog/event blocks** — call class methods.
- **2.14 No new macros** — undebuggable, unchecked; refactor existing ones to methods.
- **2.15 Table type matches access**: index→`STANDARD`, index+key→`SORTED`, key-only→`HASHED`.
- **2.16 Row access**: read-only narrow row→work area; wide/modified→field symbol (`ASSIGNING`); pass-by-ref→`REFERENCE INTO`. In loops always prefer field symbols.
- **2.17 No whole-table change inside its own loop** — no `DELETE table WHERE`/`MODIFY table`; change only the assigned row.
- **2.18 Wrap shared data** (shared memory/objects/app buffer) in Data Access Classes with getters/setters.
- **2.19 Accessibility** — labels, tooltips, column headers; no info by colour alone; titled frames.
- **2.20 No `sy-*` in UI** — map to a named, labelled variable first.

## 3. i18n / l10n
- No hardcoded literals or text constants — use message classes / text symbols (`TEXT-xxx`); DB text via text tables (foreign key, SE63).
- One original language (English); English-only names; numbered message placeholders `&1`–`&4`, never anonymous `&`.

## 4. OO Design
- **Prefer classes**; FMs only where the framework requires (RFC/update/BAdI); no new `FORM`s.
- **SOLID**: single responsibility; open/closed; Liskov; interface segregation; depend on interfaces + inject dependencies.
- **Law of Demeter** — `order->get_ship_to_city( )`, not `order->get_header( )->get_partner( )->...->get_city( )`.
- **No static utility classes** — model responsibilities as objects (`NEW zcl_file( ... )->get_str_tab( )`).
- **Patterns** where they fit: Factory, Strategy, Observer, Decorator, Template Method.

## 5. Database
- ABAP SQL only (Native SQL only when unavoidable; document why).
- Check `sy-subrc` after every DB op (raise on miss).
- Select only required fields — never `SELECT *`.
- `FOR ALL ENTRIES`: guard the driver table with `IF … IS NOT INITIAL` (empty driver returns the whole table).

## 6. Performance
- **No DB call inside `LOOP`/`DO`/`WHILE`/`SELECT…ENDSELECT`** — collect keys, one bulk fetch, read from internal table in the loop:
```abap
IF orders IS NOT INITIAL.
  SELECT vbeln, matnr, kwmeng FROM vbap
    FOR ALL ENTRIES IN @orders WHERE vbeln = @orders-vbeln
    INTO TABLE @DATA(items).
ENDIF.
LOOP AT orders ASSIGNING FIELD-SYMBOL(<order>).
  READ TABLE items ASSIGNING FIELD-SYMBOL(<item>) WITH KEY vbeln = <order>-vbeln.
ENDLOOP.
```
- Prefer `JOIN` over FAE+range when joining DB tables; on HANA, FAE+FDA can beat JOINs (keep patch level current).
- Batch with `WHERE … IN ( … )`; pre-compute keys before fetching.
- Profile non-trivial code with SAT/ST05; document findings in the design note.

## 7. ABAP Unit
- Test only the public interface; verify behaviour, not implementation.
- Isolate (`SETUP`/`TEARDOWN`), order-independent, repeatable; mock external deps (DB/RFC/file) via injection.
- Class `zcl_<component>_test`; method names describe the scenario; use `cl_abap_unit_assert` helpers (`assert_true`/`assert_subrc`/…).
- Design for testability: inject collaborators via constructor/setter behind interfaces.

## 8. S/4HANA — BOPF & CDS
- **BOPF**: never `SELECT/UPDATE/INSERT/DELETE` on BO-managed tables; go through `/BOBF/IF_TRA_SERVICE_MANAGER`. For SAP TM follow `BOPF_Changes.instructions.md`.
- **CDS = data model only** (associations/joins/annotations/aliases); business rules belong in ABAP/BOs, not `WHERE` clauses.
- Build a Basic/Interface view per table (e.g. `I_Material`); consumption views reference it, never the raw table.
- Every CDS view has a DCL; prefer `DEFINE VIEW ENTITY`.

## 9. Modification Logs
**9.1 Header** when creating a Method/Program/FM:
```abap
*&---------------------------------------------------------------------*
*& Object   : <name>      WRICEF/Feature : <id>
*& Transport: <TR>        Release        : <rel>      Author: <uid>
*& Purpose  : <what the code does>
*&---------------------------------------------------------------------*
*& CR/Defect ID-Description | Transport  | Date       | Rel  | Author |
*& DC 6000017454: FRO fix   | NS1K964849 | 30.07.2025 | July | SSH811 |
*&---------------------------------------------------------------------*
```
**9.2 Changing existing code** — add the entry to the log; wrap multi-line changes with `*---> BOC <id>` / `*<--- EOC <id>`; single-line changes get an inline `"++Added …` / `"--Commented …` tag with the same details.

## 10. Self-Review Checklist
```
[ ] ABAP 7.40+ syntax; Pretty-Printer; snake_case/English
[ ] No type prefixes (lv_/lt_/lo_/ls_); role prefixes (i_/e_/p_/s_/co_) kept
[ ] No obsolete statements (clean vs SAP/abap-cleaner); no macros
[ ] No magic values; nesting <=3 with early exits; no dead code
[ ] No SELECT in loops; FAE guarded; sy-subrc checked; no SELECT *
[ ] Class-based exceptions (RAISE … NEW); abap_bool/true/false
[ ] RETURN-only exit; thin event blocks; correct table type; field symbols in loops
[ ] No whole-table change in own loop; shared data in DAC
[ ] No hardcoded UI text; &1–&4 placeholders; accessibility met
[ ] SOLID + Law of Demeter; tests on public interface, deps injectable
[ ] BOPF: no direct DB on BO tables; CDS: model-only + DCL + Basic-view hierarchy
[ ] Profiling considered; mod-log header / BOC-EOC added
```
