# ABAP Coding Practice Instructions for AI Agent

> Based on: [ABAP Best Practices by ilyakaznacheev](https://ilyakaznacheev.github.io/abap-best-practice/)
>
> **Purpose:** This instruction file governs how you, as an AI coding agent, must write, review, and suggest ABAP code. Always apply every rule in this document. When generating code, flag any violation you deliberately omit and explain why.

---

## 0. Agent Behaviour Contract

- Apply **all** rules below to every piece of ABAP code you produce or review.
- When a rule cannot be followed (e.g., legacy system constraint), **call it out explicitly** and suggest the best available alternative.
- Default to the most modern ABAP syntax available (ABAP 7.40+ unless the user specifies an older release).
- Provide short inline comments in generated code only where business logic is non-obvious. Never comment *how* — only *why*.
- After generating any non-trivial code, append a **"Best Practices Checklist"** section confirming which rules were applied.

---

## 1. Style and Formatting

### 1.1 Pretty-Printer
- Always format code as if Pretty-Printer has been run: consistent indentation (2 spaces per level), keywords in **UPPER CASE**, operands in **lower_case**.
- Never produce unformatted or inconsistently indented code.

### 1.2 snake_case
- Use underscores between words for all identifiers: variables, methods, classes, interfaces, constants.
- ✅ `run_accounting_document_processing()`
- ❌ `runAccountingDocumentProcessing()`, `RunAccDoc()`

### 1.3 Consistent Spelling
- Pick **one** set of comparison operators and use it throughout: prefer symbolic operators (`=`, `<>`, `>=`, `<=`, `>`, `<`) over alphabetic ones (`EQ`, `NE`, `GE`, `LE`, `GT`, `LT`).
- Never mix both styles in the same code unit.

### 1.4 No Obsolete Statements
Avoid the following deprecated constructs. Always use the modern alternative:

| ❌ Obsolete                    | ✅ Modern Alternative                         |
|-------------------------------|----------------------------------------------|
| `MOVE a TO b`                 | `b = a`                                      |
| `COMPUTE x = y`               | `x = y`                                      |
| `ADD 1 TO lv_count`           | `lv_count = lv_count + 1` or `lv_count += 1`|
| `WRITE TO`                    | String expressions / `|...|`                 |
| `DATA: ... TYPE TABLE OF ... WITH HEADER LINE` | Separate `DATA` + `DATA` structure |
| `CALL FUNCTION ... EXCEPTIONS ... OTHERS = n` (without checking) | Class-based exceptions |
| `SELECT ... INTO TABLE ... BYPASSING BUFFER` overuse | Targeted OpenSQL |
| `FIELD-SYMBOLS: <fs> TYPE ANY`| Typed field symbols where possible            |

---

## 2. General Coding Rules

### 2.1 Separation of Concerns
- Split programs into layers: **UI / Controller / Business Logic / Data Access**.
- Dialog modules and event blocks (`AT SELECTION-SCREEN`, `START-OF-SELECTION`, PAI/PBO modules) must contain **no business logic** — delegate immediately to a class method.

```abap
" ✅ Good: event block delegates to class
START-OF-SELECTION.
  lo_processor = NEW zcl_report_processor( ).
  lo_processor->run( ).
```

### 2.2 Self-Describing Code
- Name variables, methods, and classes so the code reads like prose.
- ❌ `lv_i`, `lt_data`, `cl_util`, `start()`
- ✅ `lv_index`, `lt_sale_orders`, `zcl_file_io_helper`, `run_accounting_document_processing()`
- Decompose large methods into a sequence of smaller, well-named methods.
  - ❌ `process_document()`
  - ✅ `prepare_document_data()` → `is_doc_creation_possible()` → `lock_tables()` → `create_document()` → `unlock_tables()`

### 2.3 Commenting
- **Do NOT** comment on *how* code works — the code itself should explain that.
- **DO** comment the *business reason* for a decision, constraint, or special case.
- A short description of business context in the method header is the ideal comment.

```abap
" ✅ Good: explains business reason
" Clearing documents must use company-specific document type per FI policy §4.2
lo_doc_processor->change_document(
  iv_blart = lc_clearing_document
  iv_bukrs = lc_main_company
).

" ❌ Bad: explains the obvious
" Call change_document with blart and bukrs
lo_doc_processor->change_document( ... ).
```

### 2.4 Scope — Be as Local as Possible
- Declare variables, constants, and types at the **lowest possible scope**.
- Prefer local variables inside methods over class attributes; prefer class attributes over global variables.
- Never use global variables to pass data between methods — use parameters.

### 2.5 No Magic Numbers or Literals
Replace all hard-coded values with named constants that describe their **business meaning**.

```abap
" ❌ Bad
lo_doc_processor->change_document( iv_blart = 'AB'  iv_bukrs = 'C123' ).

" ✅ Good
CONSTANTS:
  lc_clearing_document TYPE blart VALUE 'AB',
  lc_main_company      TYPE bukrs VALUE 'C123'.

lo_doc_processor->change_document(
  iv_blart = lc_clearing_document
  iv_bukrs = lc_main_company
).
```

### 2.6 Avoid Deep Nesting
- Maximum nesting depth: **3 levels**. Refactor anything deeper.
- Use early exits (`RETURN`, `CONTINUE`, `CHECK`) to invert conditions and flatten nesting.

```abap
" ❌ Bad — 4 levels deep
LOOP AT lt_data ASSIGNING <ls_data>.
  IF a = b.
    IF c = d.
      IF <ls_data> IS NOT INITIAL.
        <ls_data>-field = 'aaa'.
      ENDIF.
    ELSE.
      <ls_data>-field = 'bbb'.
    ENDIF.
  ELSE.
    <ls_data>-field = 'ccc'.
  ENDIF.
ENDLOOP.

" ✅ Good — early exits, flat structure
LOOP AT lt_data ASSIGNING <ls_data>.
  IF a <> b.
    <ls_data>-field = 'ccc'.
    CONTINUE.
  ENDIF.

  IF c <> d.
    <ls_data>-field = 'bbb'.
    CONTINUE.
  ENDIF.

  CHECK <ls_data> IS NOT INITIAL.
  <ls_data>-field = 'aaa'.
ENDLOOP.
```

### 2.7 Delete Dead Code
- Never leave commented-out code blocks, unused variables, unreachable branches, or old implementations in place.
- Use Extended Program Check (`SLIN`) and Code Inspector (`SCI`/`ATC`) to detect dead code before delivery.

### 2.8 Error Handling — Never Ignore Errors
- Every operation that can fail **must** have an explicit error reaction: raise an exception, log the error, or show a message.
- Silently swallowing errors is forbidden.
- Always check `sy-subrc` after database operations — even if no action is required, document that explicitly.

```abap
" ✅ Explicit no-action comment
IF sy-subrc <> 0.
  " No action needed: missing entry is a valid state per business rule
ENDIF.
```

### 2.9 Class-Based Exceptions
- Use **only** class-based exceptions (`CX_*` / `ZCX_*`). Never use classic `EXCEPTIONS` in new code.
- One exception class per **type of problem**; use different message texts for different root causes.
- Handle exceptions as close to the raising point as the context allows.

```abap
" ✅ Good
TRY.
    lo_file_reader->read( iv_path = lv_file_path ).
  CATCH zcx_io_error INTO lo_error.
    " Handle: log, message, or re-raise with context
    MESSAGE lo_error->get_text( ) TYPE 'E'.
ENDTRY.
```

### 2.10 Boolean Fields
- Use `abap_bool` type — never `c LENGTH 1` or `char1` for boolean values.
- Use constants `abap_true` and `abap_false` — never literals `'X'` or `' '`.
- Avoid `IS INITIAL` / `IS NOT INITIAL` for boolean semantics.

```abap
" ❌ Bad
DATA: lv_flag TYPE c LENGTH 1.
lv_flag = 'X'.

" ✅ Good
DATA: lv_show_popup TYPE abap_bool.
lv_show_popup = abap_true.
```

### 2.11 Implicit Data Declarations — Avoid
- Never use `TABLES`, `NODES`, or `TABLE ... WITH HEADER LINE`.
- Always use explicit `DATA` declarations with fully qualified types.

### 2.12 Exit Methods with RETURN Only
- Use only `RETURN` to exit a method, function module, or subroutine.
- Do not use `CHECK` or `EXIT` as method exits.

### 2.13 No Logic in Dialog Modules / Event Blocks
- All PAI/PBO modules and event blocks must be **thin wrappers** that call class methods.

### 2.14 Avoid Macros
- Do not write new macros. They cannot be debugged, have no syntax check, and hide interfaces.
- Replace existing macros with methods during refactoring.

### 2.15 Internal Table Selection — Type Matters

| Access Pattern              | Table Type      |
|-----------------------------|-----------------|
| Index access only           | `STANDARD TABLE`|
| Index + key access          | `SORTED TABLE`  |
| Key access only             | `HASHED TABLE`  |

### 2.16 Table Row Access — Method Matters

| Scenario                                  | Use              |
|-------------------------------------------|------------------|
| Narrow row type, read-only                | Work area (`INTO`)|
| Wide/deep row type, to be modified        | Field symbol (`ASSIGNING`) |
| Wide/deep row type, pass reference        | Reference (`REFERENCE INTO`) |

- In loops, **always prefer field symbol** over work area copy to avoid row-copy overhead.

### 2.17 Do Not Modify an Entire Table Inside a Loop
- Never call `DELETE lt_table WHERE ...` or `LOOP AT lt_table ... MODIFY lt_table` on the outer loop's own table.
- Modify only the currently assigned row via field symbol.

### 2.18 Shared Data Access
- Wrap all shared memory / shared object / application buffer access in dedicated **Data Access Classes (DAC)** with getter/setter methods.
- Never access shared resources directly from business logic.

### 2.19 Accessibility
- Every UI element must meet accessibility standards:
  - Input/output fields: meaningful labels.
  - Icons: tooltip text.
  - Table columns: header text.
  - Information must not rely on color alone.
  - Fields grouped in frames with titles.

### 2.20 System Fields in UI
- Never display `sy-*` fields directly in the UI.
- Map to a dedicated program variable with a meaningful name and label first.

---

## 3. Language and Translation (i18n / l10n)

| Rule | Detail |
|------|--------|
| No hardcoded text literals | Use message classes (`MESSAGE`) or text symbols (`TEXT-xxx`) |
| No text constants | Constants cannot be translated; use messages or text symbols |
| Text tables for DB text storage | Assign text tables via foreign key; supports translation and SE63 |
| One original language per project | Pick English; apply uniformly across all objects |
| English-only naming | All variable, method, class, and dictionary object names in English |
| Translatable texts in UI only | Messages, OTR, text symbols — no hard-coded strings |
| Numbered placeholders | Use `&1`–`&4` in messages, never anonymous `&` |

---

## 4. Object-Oriented Programming

### 4.1 Prefer Classes
- Use ABAP classes for all new development.
- Function Modules are acceptable **only** where required by the framework: RFC, update modules, BAdI implementations that mandate FM.
- `FORM` routines are obsolete — do not write new ones.

### 4.2 SOLID Principles

| Principle | What to enforce |
|-----------|-----------------|
| **S**ingle Responsibility | Each class/method does exactly one thing |
| **O**pen/Closed | Extend behavior via inheritance or composition; do not modify existing classes |
| **L**iskov Substitution | Subclasses must be usable wherever the parent/interface is expected |
| **I**nterface Segregation | Keep interfaces focused; do not force clients to depend on methods they don't use |
| **D**ependency Inversion | Depend on interfaces, not concrete classes; inject dependencies |

### 4.3 Law of Demeter
- A method may call methods on: `self`, its own attributes, parameters passed in, objects it creates.
- Never chain calls across unrelated objects: ❌ `lo_order->get_header( )->get_partner( )->get_address( )->get_city( )`
- ✅ Provide a direct delegation method: `lo_order->get_ship_to_city( )`

### 4.4 No Utility/Helper Classes with Static Methods
- Instead of `cl_file_util=>upload_file_into_str_tab( lv_path )` create a proper object:
  `NEW zcl_file( iv_path = lv_path )->get_str_tab( )`
- Design objects around *responsibilities*, not functions.

### 4.5 Design Patterns
Apply well-known patterns where appropriate. Common ABAP use cases:
- **Factory** — object creation abstraction
- **Strategy** — interchangeable algorithms behind an interface
- **Observer/Event** — decoupled event notification
- **Decorator** — extend behavior without subclassing
- **Template Method** — define skeleton, let subclasses fill in steps

---

## 5. Database Usage

### 5.1 Use OpenSQL (ABAP SQL)
- Always use OpenSQL / ABAP SQL. Use Native SQL only when a DB-specific function is unavoidable and document why.

### 5.2 Always Check sy-subrc After DB Operations
```abap
SELECT SINGLE * FROM zorders INTO ls_order WHERE order_id = lv_id.
IF sy-subrc <> 0.
  RAISE EXCEPTION TYPE zcx_order_not_found.
ENDIF.
```

### 5.3 Select Only Required Fields
- ❌ `SELECT * FROM mara INTO TABLE lt_mara`
- ✅ `SELECT matnr mtart mbrsh FROM mara INTO TABLE lt_mara WHERE ...`

### 5.4 FOR ALL ENTRIES — Always Check Empty Table First
```abap
" ❌ Dangerous: if lt_orders is empty, returns entire DB table!
SELECT * FROM vbap INTO TABLE lt_items
  FOR ALL ENTRIES IN lt_orders
  WHERE vbeln = lt_orders-vbeln.

" ✅ Safe
IF lt_orders IS NOT INITIAL.
  SELECT vbeln matnr kwmeng FROM vbap INTO TABLE lt_items
    FOR ALL ENTRIES IN lt_orders
    WHERE vbeln = lt_orders-vbeln.
ENDIF.
```

---

## 6. Performance

### 6.1 No SELECT Inside Loops
- Never execute a `SELECT` (or any DB call) inside `LOOP`, `DO`, `WHILE`, or `SELECT ... ENDSELECT`.
- Collect all keys before the loop; execute one bulk query; use an internal table in the loop.

```abap
" ❌ Bad
LOOP AT lt_orders ASSIGNING <ls_order>.
  SELECT SINGLE * FROM vbap INTO ls_item WHERE vbeln = <ls_order>-vbeln.
ENDLOOP.

" ✅ Good
IF lt_orders IS NOT INITIAL.
  SELECT vbeln matnr kwmeng FROM vbap INTO TABLE lt_items
    FOR ALL ENTRIES IN lt_orders
    WHERE vbeln = lt_orders-vbeln.
ENDIF.
LOOP AT lt_orders ASSIGNING <ls_order>.
  READ TABLE lt_items ASSIGNING <ls_item>
    WITH KEY vbeln = <ls_order>-vbeln.
ENDLOOP.
```

### 6.2 Prefer JOIN over FOR ALL ENTRIES + RANGE
- Use `JOIN` when joining two DB tables — it is executed entirely inside the database and avoids the application/DB round-trip overhead of FAE.

### 6.3 Use FAE on HANA with FDA
- On SAP HANA systems, FAE with Fast Data Access (FDA) can outperform JOINs for specific scenarios. Ensure the latest patch level is applied.

### 6.4 Reduce Total Number of DB Requests
- Batch-fetch rows once with a `WHERE ... IN (...)` range rather than one request per key.
- Pre-calculate all keys before the fetch section of a method.

### 6.5 Profile Non-Trivial Code
- For any process beyond a simple list report, run `SAT` (Runtime Analysis) or `ST05` (SQL Trace) and verify there are no unexpected DB hits or loops.
- Document profiling findings in the technical design note.

---

## 7. Unit Testing (ABAP Unit)

### 7.1 Test Only Public Interface
- Never test private or protected methods directly.
- Tests verify **behavior** (what), not **implementation** (how). Internal refactoring must not break tests if behavior is unchanged.

### 7.2 Isolate Tests
- Each test method must be fully independent: set up all state in `SETUP`, tear down in `TEARDOWN`.
- Tests must never depend on each other's execution order.

### 7.3 Tests Must Be Repeatable
- Same inputs → same outputs, every time, in any environment.
- Mock all external dependencies (DB, RFC, file system) using test doubles / injection.

### 7.4 Tests as Living Documentation
- Test class names: `zcl_<component>_test`
- Test method names describe the scenario: `test_order_creation_fails_when_company_blocked`
- A new developer should be able to understand expected behavior by reading test names and bodies.

### 7.5 Design for Testability
- Apply Dependency Injection: pass collaborators via constructor or setter, not hardcoded `NEW` inside methods.
- Hide external dependencies behind interfaces so they can be mocked.
- Low coupling + high cohesion = easy to test.

```abap
" ✅ Testable design: DB access injected
CLASS zcl_order_processor DEFINITION.
  PUBLIC SECTION.
    METHODS:
      constructor
        IMPORTING io_db_access TYPE REF TO zif_order_db_access,
      process_orders.
  PRIVATE SECTION.
    DATA: mo_db_access TYPE REF TO zif_order_db_access.
ENDCLASS.
```

---

## 8. S/4HANA — BOPF & CDS

### 8.1 BOPF — No Direct DB Access
- Never use `SELECT`, `UPDATE`, `INSERT`, or `DELETE` directly on tables managed by a BOPF Business Object.
- Always go through the BOPF API (`/BOBF/IF_TRA_SERVICE_MANAGER`) to ensure buffering, validations, and determinations fire correctly.

### 8.2 CDS Views — No Business Logic
- CDS Views are **data models only**: associations, joins, annotations, field aliases.
- Business rules, filtering by status, or derived calculations belong in ABAP classes or BOs — not in CDS view `WHERE` clauses.

### 8.3 CDS Hierarchy — Use Basic Views
- For every database table, create a **Basic/Interface CDS View** (e.g., `I_Material` for `MARA`).
- Consumption views and upper-layer views must reference the Basic View — never the raw DB table directly.

---

## 9. Modification logs

### 9.1 Creating a Mod log header - When creating a method/ Program/ Function Module

- Mod logs are comments which provide the context of What the code does, Who edited it and when it was edited and under which DC or TR
```abap
*& 1/ Object Overview
*&---------------------------------------------------------------------*
*&    @Object Name       : <Name of the Method/program/ FUnction Module>
*&    @Initial WRICEF ID : <Work Item Id> | <Feature ID>
*&    @Transport #       : <TR > eg: NS1K959820
*&    @Release           : <Release> eg:July Minor 2025
*&    @Author            : <Author User ID> eg: SSH811
*&---------------------------------------------------------------------*
*& 2/ Description of Object’s Purpose & Functionality
*&---------------------------------------------------------------------*
*& Purpose: some Descrption about what the below code does
*&---------------------------------------------------------------------*
*& 3/ Modification Log
*&---------------------------------------------------------------------*
*& CR/Defect ID               | Transport  | Date       | Rel | Author |
*&---------------------------------------------------------------------*
*& <CR/Defect ID>-Description | NSxKXXXXXX | DD.MM.YYYY | 3.2 | SSH811 |
eg: 
*& DC 6000017454: LTMS_SDE_   | NS1K964849 | 30.07.2025 | July| SSH811 |
*& SAP-2526_FRO Mass Trigger Fix
*&---------------------------------------------------------------------*
```

### 9.2 Updating the existing code
- When Updating a specific piece of code add the Current change details in the modification log
- Add the Begin of change and End of change with the Modification log details specified in the header
- For any single line changes like addition or commenting add inline comments with the modification details
```abap
 IF tor_service_manager IS NOT BOUND.
      tor_service_manager = /bobf/cl_tra_serv_mgr_factory=>get_service_manager( iv_bo_key = /scmtms/if_tor_c=>sc_bo_key ).
    ENDIF.

    "Fetch Tor data
    tor_service_manager->retrieve(
      EXPORTING
        iv_node_key             = /scmtms/if_tor_c=>sc_node-root
        it_key                  = i_key
        iv_edit_mode            = i_root_edit_mode  "++Added for DC 6000017454| NS1K964849 | By SSH811
      IMPORTING
        et_data                 = e_tor_root
        et_failed_key           = e_failed_keys "++Added for DC 6000017454| NS1K964849 | By SSH811
        "eo_message              = e_message "--Commented for DC 6000017454| NS1K964849 | By SSH811
    ).
*---> BOC for DC 6000017454| NS1K964849 | By SSH811
    "if there are any locked keys return
    if e_message is BOUND.
      /scmtms/cl_common_helper=>analyze_messages(
        EXPORTING
          it_lock_key               = i_key
          io_message                = e_message
        IMPORTING
          et_locked_key             = e_failed_keys
      ).
    endif.
*<--- EOC for DC 6000017454| NS1K964849 | By SSH811
```

---
## 10. Code Quality Gates (Checklist for Agent)

Before finalising any generated code, confirm each item:

```
ABAP Best Practices — Self-Review Checklist
============================================
[ ] Pretty-Printer formatting applied
[ ] Hungarian notation used correctly for all identifiers
[ ] snake_case throughout
[ ] No obsolete statements
[ ] No magic numbers/literals — all constants named
[ ] Nesting depth ≤ 3
[ ] No SELECT inside loops
[ ] FOR ALL ENTRIES guarded by IS NOT INITIAL check
[ ] sy-subrc checked after every DB operation
[ ] SELECT * avoided — only required fields selected
[ ] Class-based exceptions used — no classic EXCEPTIONS
[ ] abap_bool / abap_true / abap_false used for Boolean fields
[ ] No TABLE WITH HEADER LINE
[ ] RETURN used for method exit (not CHECK/EXIT)
[ ] No logic in dialog modules / event blocks
[ ] No macros
[ ] Internal table type matches access pattern (standard/sorted/hashed)
[ ] Field symbols used in loops (no unnecessary row copies)
[ ] Shared data wrapped in Data Access Class
[ ] No hardcoded UI texts
[ ] Numbered message placeholders (&1–&4)
[ ] SOLID principles respected
[ ] Law of Demeter respected
[ ] Unit tests designed for public interface only
[ ] Dependencies injectable / mockable
[ ] Profiling considered for complex processes
```

---

## 11. Quick Reference — Common Patterns

### Safe Internal Table Loop with Modification
```abap
LOOP AT lt_sales_orders ASSIGNING <ls_order>.
  CHECK <ls_order>-status = lc_status_open.
  <ls_order>-processed = abap_true.
ENDLOOP.
```

### Bulk DB Read with FAE
```abap
IF lt_order_keys IS NOT INITIAL.
  SELECT vbeln erdat auart
    FROM vbak
    INTO TABLE @DATA(lt_order_headers)
    FOR ALL ENTRIES IN @lt_order_keys
    WHERE vbeln = @lt_order_keys-vbeln.
ENDIF.
```

### Class-Based Exception Raise and Catch
```abap
" Raise
RAISE EXCEPTION TYPE zcx_validation_error
  MESSAGE e001(zvalidation) WITH lv_field_name.

" Catch
TRY.
    lo_validator->validate( ls_input ).
  CATCH zcx_validation_error INTO DATA(lo_error).
    lv_message = lo_error->get_text( ).
    " Handle appropriately
ENDTRY.
```

### Dependency Injection Constructor
```abap
METHOD constructor.
  " Assign injected dependency or default
  mo_db_access = COND #(
    WHEN io_db_access IS BOUND
    THEN io_db_access
    ELSE NEW zcl_order_db_access( )
  ).
ENDMETHOD.
```

---
