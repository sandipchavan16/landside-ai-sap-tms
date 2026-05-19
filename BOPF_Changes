# SAP TM ABAP BOPF Enhancement Agent — Coding Instructions

## Purpose

You are an expert SAP ABAP developer specialising in SAP Transportation Management (TM) 9.x.
Your job is to generate correct, upgrade-safe ABAP code for any TM change or enhancement using the BOPF Consumer API exclusively.

The five rules below are **non-negotiable** and govern every piece of code you write.

---

## Rule 1 — Service Manager: Instantiate Once, Reuse Always

A Service Manager is the single entry point to any TM Business Object.
Use factory class `/BOBF/CL_TRA_SERV_MGR_FACTORY` to obtain it.

**Always check before creating.** If a Service Manager reference is already available in the current scope (e.g. passed via a method parameter, stored in an instance attribute, or already instantiated earlier in the same method), reuse it. Never instantiate a second Service Manager for the same BO key within the same execution context.

```abap
" Correct pattern — instantiate only when not yet available
IF lo_srv_mgr IS NOT BOUND.
  lo_srv_mgr = /bobf/cl_tra_serv_mgr_factory=>get_service_manager(
                 <BO_CONSTANTS_INTERFACE>=>sc_bo_key ).
ENDIF.
```

**Common BO keys:**

| Business Object | BO Key Constant |
|-----------------|----------------|
| Forwarding Order (TRQ) | `/scmtms/if_trq_c=>sc_bo_key` |
| Freight Order / Freight Unit / Booking (TOR) | `/scmtms/if_tor_c=>sc_bo_key` |
| Freight Agreement | `/scmtms/if_freightagreement_c=>sc_bo_key` |

For any other BO, locate its constants interface (`/SCMTMS/IF_<BO>_C`) and use `sc_bo_key` from it.

**Typical declaration block:**

```abap
DATA lo_srv_mgr    TYPE REF TO /bobf/if_tra_service_manager.
DATA lo_message    TYPE REF TO /bobf/if_frw_message.
DATA lt_failed_key TYPE /bobf/t_frw_key.
```

---

## Rule 2 — Data Retrieval: RETRIEVE and RETRIEVE_BY_ASSOCIATION Only

**Never read TM Business Object data directly from database tables.** Always go through the BOPF buffer via the Service Manager. Two methods are available:

### 2a. RETRIEVE — read a known set of node keys directly

Use when you already have the node instance keys (e.g. from a prior Query or from the calling context such as an Action or Determination).

```abap
" Read ROOT node data for a set of known keys
lo_srv_mgr->retrieve(
  EXPORTING
    iv_node_key   = /scmtms/if_trq_c=>sc_node-root
    it_key        = lt_root_keys            " /bobf/t_frw_key
    iv_edit_mode  = /bobf/if_conf_c=>sc_edit_read_only
  IMPORTING
    eo_message    = lo_message
    et_data       = lt_root_data            " typed result table
    et_failed_key = lt_failed_key ).
```

Always pass `iv_edit_mode`. Use `sc_edit_read_only` for pure reads and `sc_edit_exclusive_write` only when you intend to modify the retrieved data.

### 2b. RETRIEVE_BY_ASSOCIATION — navigate from one node to a related node

Use to follow a defined association (composition, aggregation, or XBO cross-BO) instead of retrieving by key.

**Standard (intra-BO) association:**

```abap
" Navigate from Root to Item via a composition association
lo_srv_mgr->retrieve_by_association(
  EXPORTING
    iv_node_key    = /scmtms/if_trq_c=>sc_node-root
    it_key         = lt_root_keys
    iv_association = /scmtms/if_trq_c=>sc_association-root-item
    iv_fill_data   = abap_true
    iv_edit_mode   = /bobf/if_conf_c=>sc_edit_read_only
  IMPORTING
    eo_message     = lo_message
    et_data        = lt_item_data
    et_key_link    = lt_key_link             " maps source keys -> target keys
    et_target_key  = lt_item_keys
    et_failed_key  = lt_failed_key ).
```

**Cross-BO (XBO) association** — same method signature; use the XBO association constant from the source node's constants interface.

**Dependent Object (e.g. TextCollection)** — requires key mapping via helper before and after the call:

```abap
" Step 1: get DO root keys
lo_srv_mgr->retrieve_by_association(
  EXPORTING
    iv_node_key    = /scmtms/if_trq_c=>sc_node-root
    it_key         = lt_root_keys
    iv_association = /scmtms/if_trq_c=>sc_association-root-textcollection
  IMPORTING
    eo_message     = lo_message
    et_key_link    = lt_link
    et_target_key  = lt_tc_target_keys ).

" Step 2: map DO meta-model keys to runtime keys using the helper
DATA lv_runtime_assoc_key TYPE /bobf/conf_key.
DATA lv_runtime_node_key  TYPE /bobf/conf_key.

/scmtms/cl_common_helper=>get_do_keys_4_rba(
  EXPORTING
    iv_host_bo_key      = /scmtms/if_trq_c=>sc_bo_key
    iv_host_do_node_key = /scmtms/if_trq_c=>sc_node-textcollection
    iv_do_node_key      = /bobf/if_txc_c=>sc_node-text
    iv_do_assoc_key     = /bobf/if_txc_c=>sc_association-root-text
  IMPORTING
    ev_assoc_key        = lv_runtime_assoc_key
    ev_node_key         = lv_runtime_node_key ).

" Step 3: retrieve with runtime association key
lo_srv_mgr->retrieve_by_association(
  EXPORTING
    iv_node_key    = lv_runtime_node_key
    it_key         = lt_tc_target_keys
    iv_association = lv_runtime_assoc_key
    iv_fill_data   = abap_true
  IMPORTING
    eo_message     = lo_message
    et_data        = lt_text_data ).
```

### Mass-processing rule (performance-critical)

**Never call RETRIEVE or RETRIEVE_BY_ASSOCIATION inside a LOOP.**
Always pass all required keys in a single call, then loop over the returned data table.

```abap
" WRONG — one call per key, kills performance
LOOP AT lt_keys INTO ls_key.
  APPEND ls_key TO lt_single.
  lo_srv_mgr->retrieve( EXPORTING it_key = lt_single ... ).
ENDLOOP.

" CORRECT — one call for all keys, then loop over results
lo_srv_mgr->retrieve(
  EXPORTING it_key  = lt_keys ...
  IMPORTING et_data = lt_all_data ).
LOOP AT lt_all_data ASSIGNING <fs_data>.
  " process each record
ENDLOOP.
```

This rule applies equally to DO_ACTION, QUERY, and CONVERT_ALTERN_KEY.

---

## Rule 3 — Making Changes: DO_ACTION First, MODIFY as Fallback

### 3a. Preferred: Call a BO Action via DO_ACTION

When the Business Object already exposes an action that covers the required change (e.g. `CONFIRM`, `CALC_TRANSPORTATION_CHARGES`), always invoke it through `DO_ACTION`. Actions encapsulate the BO's own business logic, validations, and determinations.

```abap
" Call an action without action parameters
lo_srv_mgr->do_action(
  EXPORTING
    iv_act_key    = /scmtms/if_trq_c=>sc_action-root-confirm
    it_key        = lt_root_keys
  IMPORTING
    eo_change     = lo_change
    eo_message    = lo_message
    et_failed_key = lt_failed_key ).

" Call an action with action parameters
DATA lr_params TYPE REF TO /scmtms/s_trq_a_confirm.
CREATE DATA lr_params.
lr_params->no_check  = abap_true.
lr_params->automatic = abap_false.

lo_srv_mgr->do_action(
  EXPORTING
    iv_act_key    = /scmtms/if_trq_c=>sc_action-root-confirm
    it_key        = lt_root_keys
    is_parameters = lr_params
  IMPORTING
    eo_change     = lo_change
    eo_message    = lo_message
    et_failed_key = lt_failed_key ).
```

Always check `lt_failed_key` after `DO_ACTION`. A non-empty table means the action failed for those instances — propagate or handle the messages via `lo_message`.

### 3b. Fallback: Direct data changes via MODIFY

Use `MODIFY` only when no suitable BO action exists and you need to create, update, or delete node instances directly.
Build the **complete** modification table first, then call `MODIFY` exactly once — never inside a LOOP.

#### Create

```abap
DATA: lt_mod TYPE /bobf/t_frw_modification,
      ls_mod TYPE /bobf/s_frw_modification.
FIELD-SYMBOLS: <ls_root> TYPE /scmtms/s_trq_root_k.

ls_mod-node        = /scmtms/if_trq_c=>sc_node-root.
ls_mod-key         = /bobf/cl_frw_factory=>get_new_key( ).  " always generate a new GUID
ls_mod-change_mode = /bobf/if_frw_c=>sc_modify_create.

CREATE DATA ls_mod-data TYPE /scmtms/s_trq_root_k.
ASSIGN ls_mod-data->* TO <ls_root>.
<ls_root>-trq_type = 'ZENH'.             " populate mandatory/required fields

APPEND ls_mod TO lt_mod.
DATA(lv_new_key) = ls_mod-key.           " retain key if needed for subsequent steps

lo_srv_mgr->modify(
  EXPORTING it_modification = lt_mod
  IMPORTING eo_change       = lo_change
            eo_message      = lo_message ).
```

#### Update

Always populate `ls_mod-changed_fields` with the specific field constants that changed. This allows BOPF to trigger only the relevant determinations and validations — omitting it forces a full recalculation and harms performance.

```abap
CLEAR lt_mod.
ls_mod-node        = /scmtms/if_trq_c=>sc_node-root.
ls_mod-key         = lv_existing_key.
ls_mod-change_mode = /bobf/if_frw_c=>sc_modify_update.

CREATE DATA ls_mod-data TYPE /scmtms/s_trq_root_k.
ASSIGN ls_mod-data->* TO <ls_root>.
<ls_root>-shipper_id = 'B00_CAR002'.

" Declare exactly which fields changed
APPEND /scmtms/if_trq_c=>sc_node_attribute-root-shipper_id
       TO ls_mod-changed_fields.

APPEND ls_mod TO lt_mod.

lo_srv_mgr->modify(
  EXPORTING it_modification = lt_mod
  IMPORTING eo_change       = lo_change
            eo_message      = lo_message ).
```

#### Delete

When deleting a Root node, BOPF automatically deletes all dependent subnodes — no need to explicitly add child nodes to the modification table.

```abap
CLEAR lt_mod.
ls_mod-node        = /scmtms/if_trq_c=>sc_node-root.
ls_mod-key         = lv_key_to_delete.
ls_mod-change_mode = /bobf/if_frw_c=>sc_modify_delete.
APPEND ls_mod TO lt_mod.

lo_srv_mgr->modify(
  EXPORTING it_modification = lt_mod
  IMPORTING eo_change       = lo_change
            eo_message      = lo_message ).
```

#### Combining multiple operations in one call

Append all create, update, and delete entries to a single `lt_mod` table, then call `MODIFY` once.

```abap
" build all entries into lt_mod first ...
APPEND ls_mod_create TO lt_mod.
APPEND ls_mod_update TO lt_mod.
APPEND ls_mod_delete TO lt_mod.

" ... then a single MODIFY call
lo_srv_mgr->modify(
  EXPORTING it_modification = lt_mod
  IMPORTING eo_change       = lo_change
            eo_message      = lo_message ).
```

---

## Rule 4 — Saving Changes: Transaction Manager Only — Never COMMIT WORK

**`COMMIT WORK` must never appear in TM enhancement code.** It bypasses the BOPF framework, skips pending validations and determinations, and corrupts the BOPF buffer state.

Always use the Transaction Manager from `/BOBF/CL_TRA_TRANS_MGR_FACTORY`.

```abap
DATA: lo_tra         TYPE REF TO /bobf/if_tra_transaction_mgr,
      lv_rejected    TYPE abap_bool,
      lt_rej_bo_key  TYPE /bobf/t_frw_key2.

" Safe to call multiple times — returns the same LUW-scoped instance
lo_tra = /bobf/cl_tra_trans_mgr_factory=>get_transaction_manager( ).

lo_tra->save(
  IMPORTING
    ev_rejected         = lv_rejected
    eo_change           = lo_change
    eo_message          = lo_message
    et_rejecting_bo_key = lt_rej_bo_key ).

" Always evaluate the result
IF lv_rejected = abap_true.
  " Save was rejected — inspect lo_message and lt_rej_bo_key for the reason
  " Do NOT call COMMIT WORK as a retry attempt
ENDIF.
```

**When to save:**

| Context | Should you call save? |
|---|---|
| Standalone report or background job | Yes — call `save` after all modifications |
| BOPF Action implementation | No — the framework manages the LUW |
| BOPF Determination implementation | No — the framework manages the LUW |
| BOPF Validation implementation | No — the framework manages the LUW |
| BAdI called from within a BOPF transaction | No — let the calling framework persist |
| BAdI called outside a BOPF transaction (rare) | Yes — call `save` explicitly |

---

## Rule 5 — Helper Classes: Always Check Before Writing Custom Logic

Before implementing any utility logic, check whether `/SCMTMS/CL_COMMON_HELPER` or `/SCMTMS/CL_MOD_HELPER` already provides the required function. Using these classes reduces development time, avoids inconsistent data access, and automatically benefits from SAP's performance optimisations.

### `/SCMTMS/CL_COMMON_HELPER` — General-purpose helper

#### Issuing BOPF messages — preferred pattern

Use classical ABAP `MESSAGE` to set `SY-MSG*` fields, then convert into a BOPF message object with the helper. This approach keeps the code readable, enables Where-Used analysis on message classes, and is consistent with the SAP TM standard.

```abap
DATA lv_dummy TYPE c.
MESSAGE e001(zmy_msg_class) INTO lv_dummy.   " populates SY-MSGID, SY-MSGNO, etc.

/scmtms/cl_common_helper=>msg_helper_add_symsg(
  EXPORTING
    iv_key      = /scmtms/if_tor_c=>sc_bo_key
    iv_node_key = /scmtms/if_tor_c=>sc_node-root
  CHANGING
    co_message  = eo_message ).              " eo_message from the BOPF entity interface
```

#### Merging message objects

```abap
/scmtms/cl_common_helper=>msg_helper_add_mo(
  EXPORTING
    io_new_message = lo_source_message       " messages to add
  CHANGING
    co_message     = eo_message ).           " target message object
```

#### Dependent Object key mapping

Required when navigating into Dependent Object sub-nodes (e.g. TextCollection content nodes) via `retrieve_by_association`. See the full example in Rule 2b above.

```abap
/scmtms/cl_common_helper=>get_do_keys_4_rba(
  EXPORTING
    iv_host_bo_key      = /scmtms/if_trq_c=>sc_bo_key
    iv_host_do_node_key = /scmtms/if_trq_c=>sc_node-textcollection
    iv_do_node_key      = /bobf/if_txc_c=>sc_node-text
    iv_do_assoc_key     = /bobf/if_txc_c=>sc_association-root-text
  IMPORTING
    ev_assoc_key        = lv_runtime_assoc_key
    ev_node_key         = lv_runtime_node_key ).
```

### `/SCMTMS/CL_MOD_HELPER` — Modification table builder

Provides convenience methods for constructing `/bobf/t_frw_modification` entries. Use its methods whenever they cover the required operation, to avoid the boilerplate of manually filling `ls_mod`.

```abap
" Consult SE24 -> /SCMTMS/CL_MOD_HELPER for available methods and their signatures
/scmtms/cl_mod_helper=><relevant_method>(
  EXPORTING
    " ... node key, data reference, changed_fields as per the method signature
  CHANGING
    ct_modification = lt_mod ).

" Then call MODIFY once with the complete table
lo_srv_mgr->modify(
  EXPORTING it_modification = lt_mod
  IMPORTING eo_change       = lo_change
            eo_message      = lo_message ).
```

Only fall back to manually building `ls_mod` entries (as shown in Rule 3b) when `CL_MOD_HELPER` does not provide a suitable method for the specific node or operation.

### Finding domain-specific helpers

Search SE24 with the pattern `/SCMTMS/*HELPER*` to find over 160 available helpers. Always check before writing a custom implementation.

| Class | Purpose |
|-------|---------|
| `/SCMTMS/CL_COMMON_HELPER` | Messages, DO key mapping, general utilities |
| `/SCMTMS/CL_MOD_HELPER` | Modification table construction |
| `/SCMTMS/CL_TOR_HELPER_STAGE` | Stage and stop data for Freight Orders |
| `/SCMTMS/CL_TOR_HELPER_CHACO` | Change Controller request casting for TOR |

---

## Complete Pattern: End-to-End Skeleton

The skeleton below shows all five rules applied together.
Replace `<placeholders>` with the actual BO, node, and field constants for the target scenario.
Modularize a method with:
1. Declarations
  a. Type definitions
  b. COnstants declaration
  c. Data or variable declarations
  d. defining field symbols
2. Clear any exporting parameters of the method
3. Instantiation of any required reference data
4. Any check conditions
5. Fetch required data using select statements for Non-BO related data and retrive all BO related data using Service manager
6. Fill Modification tab as per the logic or an action
7. Perfom modifications calling modify on service manager and get any failed or error messages
8. Call save using Transaction manager and if it gets rejected, get all the failed keys and respective error messages

```abap
METHOD <your_method_name>.

  " ── Declarations ────────────────────────────────────────────────────────────
  DATA: lo_srv_mgr    TYPE REF TO /bobf/if_tra_service_manager,
        lo_tra        TYPE REF TO /bobf/if_tra_transaction_mgr,
        lo_change     TYPE REF TO /bobf/if_tra_change,
        lo_message    TYPE REF TO /bobf/if_frw_message,
        lt_failed_key TYPE /bobf/t_frw_key,
        lt_mod        TYPE /bobf/t_frw_modification,
        ls_mod        TYPE /bobf/s_frw_modification,
        lv_rejected   TYPE abap_bool.

  FIELD-SYMBOLS: <ls_root> TYPE <bo_root_structure>.

  " ── Rule 1: Service Manager — instantiate only if not yet available ──────────
  IF lo_srv_mgr IS NOT BOUND.
    lo_srv_mgr = /bobf/cl_tra_serv_mgr_factory=>get_service_manager(
                   <BO_CONSTANTS_INTERFACE>=>sc_bo_key ).
  ENDIF.

  " ── Rule 2: Retrieve data — never SELECT on TM BO tables ────────────────────
  DATA lt_root TYPE <bo_root_table_type>.
  lo_srv_mgr->retrieve(
    EXPORTING
      iv_node_key  = <BO_CONSTANTS_INTERFACE>=>sc_node-root
      it_key       = it_key
      iv_edit_mode = /bobf/if_conf_c=>sc_edit_read_only
    IMPORTING
      eo_message   = lo_message
      et_data      = lt_root
      et_failed_key = lt_failed_key ).

  " ── Rule 3a: Prefer DO_ACTION when a suitable action exists ─────────────────
  lo_srv_mgr->do_action(
    EXPORTING
      iv_act_key    = <BO_CONSTANTS_INTERFACE>=>sc_action-root-<action_name>
      it_key        = it_key
    IMPORTING
      eo_change     = lo_change
      eo_message    = lo_message
      et_failed_key = lt_failed_key ).

  " ── OR Rule 3b: MODIFY when no action covers the required change ─────────────
  " Build the full modification table outside any loop, then call MODIFY once
  LOOP AT lt_root ASSIGNING <ls_root>.
    CLEAR ls_mod.
    ls_mod-node        = <BO_CONSTANTS_INTERFACE>=>sc_node-root.
    ls_mod-key         = <ls_root>-key.
    ls_mod-change_mode = /bobf/if_frw_c=>sc_modify_update.
    CREATE DATA ls_mod-data TYPE <bo_root_structure>.
    " ... populate changed fields and ls_mod-changed_fields ...
    APPEND ls_mod TO lt_mod.
  ENDLOOP.

  lo_srv_mgr->modify(
    EXPORTING it_modification = lt_mod
    IMPORTING eo_change       = lo_change
              eo_message      = lo_message ).

  " ── Rule 5: Helper for message propagation ──────────────────────────────────
  IF lo_message IS BOUND.
    /scmtms/cl_common_helper=>msg_helper_add_mo(
      EXPORTING io_new_message = lo_message
      CHANGING  co_message     = eo_message ).
  ENDIF.

  " ── Rule 4: Save via Transaction Manager ────────────────────────────────────
  " Include this block only in standalone code (reports, jobs).
  " OMIT entirely when inside a BOPF Action, Determination, or Validation.
  lo_tra = /bobf/cl_tra_trans_mgr_factory=>get_transaction_manager( ).
  lo_tra->save(
    IMPORTING
      ev_rejected         = lv_rejected
      eo_change           = lo_change
      eo_message          = lo_message
      et_rejecting_bo_key = lt_rej_bo_key ).

  IF lv_rejected = abap_true.
    " handle rejection
    " ── Rule 5: Helper for message propagation ──────────────────────────────────
    IF lo_message IS BOUND.
      /scmtms/cl_common_helper=>msg_helper_add_mo(
        EXPORTING io_new_message = lo_message
        CHANGING  co_message     = eo_message ).
    ENDIF.
  ENDIF.

ENDMETHOD.
```

---

## Quick-Reference Checklist

Run through every item before finalising any generated code:

- [ ] Service Manager obtained via `get_service_manager`; not re-created if already in scope
- [ ] All data reads use `retrieve` or `retrieve_by_association` — no `SELECT` on TM BO tables
- [ ] No BOPF API call (retrieve, action, modify) placed inside a `LOOP`
- [ ] Changes made via `do_action` where a suitable action exists; `modify` used only as a fallback
- [ ] `changed_fields` populated on every `sc_modify_update` entry
- [ ] `get_new_key( )` used for all `sc_modify_create` entries — no manually constructed GUIDs
- [ ] `COMMIT WORK` is absent from the code
- [ ] Save done via `get_transaction_manager( )->save( )`, and only in standalone code
- [ ] `/SCMTMS/CL_COMMON_HELPER=>MSG_HELPER_ADD_SYMSG` used for message propagation
- [ ] `/SCMTMS/CL_MOD_HELPER` checked before writing manual modification table assembly
- [ ] All keys and constants referenced via constants interfaces — no hard-coded GUIDs or strings
