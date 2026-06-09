# SAP TM (BOPF) Enhancement — Agent Instructions

Generate correct, upgrade-safe ABAP for SAP TM 9.x / S/4HANA TM using the **BOPF Consumer API only**. The five rules below are non-negotiable. They sit on top of `ABAP_CodingPractices.instructions.md` (Clean ABAP, 7.40+, abap-cleaner) and `NamingConventions.instructions.md`.

> **Naming**: local vars use Clean ABAP names (no `lv_`/`lt_`/`lo_`/`ls_`). The `iv_`/`it_`/`et_`/`eo_`/`es_`/`is_` prefixes below are **fixed `/BOBF/` API parameter names** — write them exactly.

## Rule 1 — Service Manager: instantiate once, reuse
Obtain via `/BOBF/CL_TRA_SERV_MGR_FACTORY`. Reuse any reference already in scope (parameter/attribute/earlier in method); never create a second one for the same BO key in one execution context.
```abap
IF service_manager IS NOT BOUND.
  service_manager = /bobf/cl_tra_serv_mgr_factory=>get_service_manager( <BO_C>=>sc_bo_key ).
ENDIF.
```
**BO keys**: TRQ (Forwarding Order) `/scmtms/if_trq_c=>sc_bo_key`; TOR (Freight Order/Unit/Booking) `/scmtms/if_tor_c=>sc_bo_key`; Freight Agreement `/scmtms/if_freightagreement_c=>sc_bo_key`. For others use `sc_bo_key` from `/SCMTMS/IF_<BO>_C`.

## Rule 2 — Data retrieval: QUERY / RETRIEVE / RETRIEVE_BY_ASSOCIATION only
**Never SELECT on TM BO tables** — always go through the buffer. Flow: QUERY → RETRIEVE / RBA. Inside Actions/Determinations/Validations the framework passes `it_key`, so QUERY is usually not needed.

| Method | Use when |
|---|---|
| `QUERY` | you have search criteria, need matching keys |
| `RETRIEVE` | you have keys, need node data |
| `RETRIEVE_BY_ASSOCIATION` | you have keys, need a related node via association |

### 2a. QUERY
Query keys/attributes live in the BO constants interface: `<BO_C>=>sc_query-<node>-<query>` and `sc_query_attribute-<node>-<query>-<attr>`. Three types: **Node Attribute** (modelled only), **Custom** and **Generic Result/QDB** (both implement `/BOBF/IF_FRW_QUERY`; QDB result's first field must be `DB_KEY TYPE /BOBF/CONF_KEY`).

Selection rows (`/bobf/t_frw_query_selparam`) mirror SELECT-OPTIONS: `attribute_name`, `sign` (`I`/`E`), `option` (`EQ`/`NE`/`BT`/`GT`/`LT`/`GE`/`LE`/`CP`), `low`, `high`. Same attr = OR; different attrs = AND.
```abap
DATA(selection_parameters) = VALUE /bobf/t_frw_query_selparam(
  ( attribute_name = /scmtms/if_trq_c=>sc_query_attribute-root-query_by_attributes-created_by
    sign = 'I' option = 'EQ' low = 'JDOE' )
  ( attribute_name = /scmtms/if_trq_c=>sc_query_attribute-root-query_by_attributes-trq_type
    sign = 'I' option = 'EQ' low = 'ZTYPE1' ) ).

service_manager->query(
  EXPORTING iv_query_key            = /scmtms/if_trq_c=>sc_query-root-query_by_attributes
            it_selection_parameters = selection_parameters
            iv_fill_data            = abap_false        " abap_true also returns et_data (costs more)
  IMPORTING eo_message              = message
            es_query_info           = query_info        " es_query_info-result_truncated => narrow criteria
            et_key                  = keys ).
IF keys IS INITIAL. RETURN. ENDIF.
```
With `iv_fill_data = abap_true`, also receive `et_data` (typed to the query result, e.g. `/scmtms/t_trq_root_k`) and skip a separate RETRIEVE.

**CONVERT_ALTERN_KEY** — faster than QUERY when you already hold the business document ID:
```abap
service_manager->convert_altern_key(
  EXPORTING iv_node_key   = /scmtms/if_trq_c=>sc_node-root
            iv_altkey_key = /scmtms/if_trq_c=>sc_alternative_key-root-trq_id
            it_key        = doc_ids
  IMPORTING et_key        = bopf_keys ).
```
**Query rules**: never inside a LOOP; prefer CONVERT_ALTERN_KEY over QUERY when a doc ID exists; avoid `iv_fill_data` unless data is needed immediately; restrict criteria tightly; check `result_truncated` when unbounded.

**Custom query skeleton** — inherit `/SCMTMS/CL_Q_SUPERCLASS`, implement `/BOBF/IF_FRW_QUERY`; inside `~retrieve` use `io_read->retrieve`/`retrieve_by_association`, fill `et_key` (and `et_data` if `iv_fill_data`); **no SELECT, MODIFY, or save** (queries are read-only).

### 2b. RETRIEVE — read known keys
```abap
service_manager->retrieve(
  EXPORTING iv_node_key   = /scmtms/if_trq_c=>sc_node-root
            it_key        = root_keys
            iv_edit_mode  = /bobf/if_conf_c=>sc_edit_read_only   " ..._exclusive_write only to modify
  IMPORTING eo_message    = message
            et_data       = root_data
            et_failed_key = failed_keys ).
```

### 2c. RETRIEVE_BY_ASSOCIATION — navigate to a related node
```abap
service_manager->retrieve_by_association(
  EXPORTING iv_node_key    = /scmtms/if_trq_c=>sc_node-root
            it_key         = root_keys
            iv_association = /scmtms/if_trq_c=>sc_association-root-item
            iv_fill_data   = abap_true
            iv_edit_mode   = /bobf/if_conf_c=>sc_edit_read_only
  IMPORTING eo_message     = message
            et_data        = item_data
            et_key_link    = key_links       " source->target map
            et_target_key  = item_keys
            et_failed_key  = failed_keys ).
```
XBO (cross-BO): same signature, XBO association constant. **Dependent Object** (e.g. TextCollection): map keys first with `/scmtms/cl_common_helper=>get_do_keys_4_rba` (→ `ev_assoc_key`, `ev_node_key`), then RBA with those runtime keys.

### Mass-processing rule (critical)
**Never call RETRIEVE/RBA/DO_ACTION/QUERY/CONVERT_ALTERN_KEY inside a LOOP.** Pass all keys in one call, then loop over results.

## Rule 3 — Changes: DO_ACTION first, MODIFY as fallback
### 3a. DO_ACTION (preferred) — encapsulates BO logic/validations/determinations
```abap
DATA(action_parameters) = NEW /scmtms/s_trq_a_confirm( no_check = abap_true automatic = abap_false ).
service_manager->do_action(
  EXPORTING iv_act_key    = /scmtms/if_trq_c=>sc_action-root-confirm
            it_key        = root_keys
            is_parameters = action_parameters    " omit if the action takes none
  IMPORTING eo_change     = change
            eo_message    = message
            et_failed_key = failed_keys ).
```
Always check `failed_keys` (non-empty = failed for those instances; handle via `message`).

### 3b. MODIFY (fallback) — only when no action fits
Build the **whole** `modifications` table first, then call MODIFY **once** (never in a LOOP). One table can mix create/update/delete.
```abap
DATA modifications TYPE /bobf/t_frw_modification.

" CREATE — always get_new_key( ); populate mandatory fields
modifications = VALUE #( ( node        = /scmtms/if_trq_c=>sc_node-root
                           key         = /bobf/cl_frw_factory=>get_new_key( )
                           change_mode = /bobf/if_frw_c=>sc_modify_create
                           data        = NEW /scmtms/s_trq_root_k( trq_type = 'ZENH' ) ) ).

" UPDATE — set changed_fields so only relevant determinations/validations fire
APPEND VALUE #( node           = /scmtms/if_trq_c=>sc_node-root
                key            = existing_key
                change_mode    = /bobf/if_frw_c=>sc_modify_update
                data           = NEW /scmtms/s_trq_root_k( shipper_id = 'B00_CAR002' )
                changed_fields = VALUE #( ( /scmtms/if_trq_c=>sc_node_attribute-root-shipper_id ) )
              ) TO modifications.

" DELETE root — subnodes are deleted automatically
APPEND VALUE #( node = /scmtms/if_trq_c=>sc_node-root
                key  = key_to_delete
                change_mode = /bobf/if_frw_c=>sc_modify_delete ) TO modifications.

service_manager->modify( EXPORTING it_modification = modifications
                         IMPORTING eo_change = change  eo_message = message ).
```

## Rule 4 — Save: Transaction Manager only — never `COMMIT WORK`
`COMMIT WORK` bypasses the framework and corrupts the buffer. Use `/BOBF/CL_TRA_TRANS_MGR_FACTORY`.
```abap
DATA(transaction_manager) = /bobf/cl_tra_trans_mgr_factory=>get_transaction_manager( ).
transaction_manager->save( IMPORTING ev_rejected         = rejected
                                     eo_message          = message
                                     et_rejecting_bo_key = rejecting_bo_keys ).
IF rejected = abap_true.
  " inspect message + rejecting_bo_keys; do NOT retry with COMMIT WORK
ENDIF.
```
**Save** in standalone reports/jobs and in BAdIs outside a BOPF transaction. **Do NOT save** inside Actions/Determinations/Validations or BAdIs within a BOPF transaction — the framework owns the LUW.

## Rule 5 — Helpers: check before writing custom logic
Search SE24 `/SCMTMS/*HELPER*` (160+ helpers) first.
- **Messages**: set `SY-MSG*` via classic `MESSAGE … INTO`, then `/scmtms/cl_common_helper=>msg_helper_add_symsg` (or `…_add_mo` to merge a message object) into `co_message`.
- **DO key mapping**: `…=>get_do_keys_4_rba` (Rule 2c).
- **Modification table**: `/SCMTMS/CL_MOD_HELPER` methods build `ct_modification` — prefer over manual `VALUE #( … )`.
- Domain helpers: `…_TOR_HELPER_STAGE` (stages/stops), `…_TOR_HELPER_CHACO` (change-controller casting).

## End-to-end skeleton
1. Declarations → 2. clear exports → 3. instantiate refs → 4. checks → 5. fetch (SELECT only non-BO data; BO data via Service Manager) → 6. build modifications / pick action → 7. modify/do_action, capture failed keys & messages → 8. save via Transaction Manager (standalone only); on reject capture keys & messages.
```abap
METHOD <name>.
  DATA modifications TYPE /bobf/t_frw_modification.

  IF service_manager IS NOT BOUND.
    service_manager = /bobf/cl_tra_serv_mgr_factory=>get_service_manager( <BO_C>=>sc_bo_key ).
  ENDIF.

  service_manager->retrieve(
    EXPORTING iv_node_key = <BO_C>=>sc_node-root  it_key = it_key
              iv_edit_mode = /bobf/if_conf_c=>sc_edit_read_only
    IMPORTING eo_message = DATA(message)  et_data = DATA(root_data)  et_failed_key = DATA(failed_keys) ).

  " Prefer DO_ACTION; else build modifications (outside any loop) and MODIFY once:
  LOOP AT root_data ASSIGNING FIELD-SYMBOL(<root>).
    APPEND VALUE #( node = <BO_C>=>sc_node-root  key = <root>-key
                    change_mode = /bobf/if_frw_c=>sc_modify_update
                    data = NEW <bo_root_structure>( )
                    changed_fields = VALUE #( ( ... ) ) ) TO modifications.
  ENDLOOP.
  service_manager->modify( EXPORTING it_modification = modifications
                           IMPORTING eo_change = DATA(change)  eo_message = message ).

  IF message IS BOUND.
    /scmtms/cl_common_helper=>msg_helper_add_mo( EXPORTING io_new_message = message
                                                 CHANGING  co_message = eo_message ).
  ENDIF.

  " Standalone code only — OMIT inside Action/Determination/Validation:
  DATA(transaction_manager) = /bobf/cl_tra_trans_mgr_factory=>get_transaction_manager( ).
  transaction_manager->save( IMPORTING ev_rejected = DATA(rejected)  eo_message = message ).
ENDMETHOD.
```

## Checklist
```
[ ] Service Manager reused, not re-created
[ ] QUERY (not SELECT) to find instances; CONVERT_ALTERN_KEY when doc ID known
[ ] Constants from sc_query/sc_node/sc_action/... — no hard-coded strings/GUIDs
[ ] iv_fill_data only when data needed now; result_truncated checked; empty keys handled
[ ] All reads via retrieve/RBA; no BOPF call inside a LOOP
[ ] do_action preferred; modify is fallback, called once; changed_fields set; get_new_key for creates
[ ] No COMMIT WORK; save via Transaction Manager, standalone only
[ ] Helpers checked (/SCMTMS/*HELPER*) before custom code
[ ] Clean ABAP + abap-cleaner; mod-log header / BOC-EOC per coding-practices §9
```
