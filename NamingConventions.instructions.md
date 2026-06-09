# SAP ABAP Naming Conventions — Agent Instructions

Always apply these names when generating/reviewing ABAP. Flag deviations and propose the correct name. Governs **names**; coding → `ABAP_CodingPractices.instructions.md`; SAP TM/BOPF → `BOPF_Changes.instructions.md`.

## Principles
- Clean ABAP, clean vs [SAP/abap-cleaner](https://github.com/SAP/abap-cleaner): no global data, no macros, enums over standalone constants.
- Singular for variables/structures; plural for internal tables. Every CDS view needs a DCL; prefer `DEFINE VIEW ENTITY`.
- Methods: ≤3 import params; only one of Export/Changing/Returning.
- `Z` = transportable, `Y` = local/sandbox.
- **No type-encoding prefixes** on locals (`lv_`/`lt_`/`ls_`/`lo_`/`lr_`/`lc_`/`gv_`) — name for what it is (`order`, `orders`, `order_count`). **Keep role prefixes** (they encode role, not type): params `I_`/`E_`/`C_`/`R_`/`T_`, screen `p_`/`c_`/`r_`/`s_`/`b_`, constants/types `CO_`/`TY_`/`TT_`. `/BOBF/` API prefixes (`iv_`/`it_`/`et_`/`eo_`/`es_`/`is_`) are fixed SAP signatures — write as-is.

## Codes
- **Module `<MD>`**: Cross `CMN` · Accounts-to-Report `A2R` · Procure-to-Pay `P2P` · Order-to-Cash `O2C` · Service Delivery `SD` · Lead-to-Agreement `L2A`.
- **Criticality `<C>`**: `C` critical · `N` not critical.

## Repository Objects
| Object | Convention | Example |
|---|---|---|
| Class | `ZCL_<type>_<name>` | `ZCL_TVARVC_CONFIGURATION` |
| Exception Class | `ZCX_<name>` | `ZCX_INVALID_CONFIGURATION` |
| Interface | `ZIF_<name>` | `ZIF_CONFIGURABLE` |
| Report | `Z<MD><C>_<name>` | `ZATRC_CREATE_BOOKINGS_XLS` |
| Include | `Z<name>` / `MZ<name>` | `ZDEPO_CHANGES_F01` |
| Module Pool | `SAPMZ<name>` | |
| Function Group / Module | `Z<MD>_<name>` / `Z<MD><C>_<name>` | `ZSD_OPTIMIZER` / `ZSDC_OPTIMIZER` |
| Transaction Code | `Z<MD><C>_<name>` | `ZSDC_ASSIGN_RECEPIENT` |
| Area Menu | `ZTM_<MD>_<name>` | `ZTM_O2C_Admin_Support` |
| Message Class | `Z<name>` | `ZFREIGHT_ORDER` |
| Package | `Z<MD>` · UI `Z<MD>_UI` · Interface `Z<MD>_INF` | `ZO2C` / `ZO2C_UI` / `ZO2C_INF` |

## Internal Declarations
| Object | Convention | Example |
|---|---|---|
| Constant | `CO_<name>` (or enum) | `co_gcss_identifier` |
| Type / Table type | `TY_<name>` / `TT_<name>` | `ty_transportation_order` |
| Attribute (inst/static/global) | `<name>` (no prefix) | `transportation_order_id` |
| Local class / interface | `lcl_<name>` / `lif_<name>` | `lcl_shipper` |
| Local test / exception class | `ltc_<name>` / `lcx_<name>` | `ltc_shipper` |
| Field symbol | `<name>` | `<shipper>` |
| Macros | avoid | |

## Selection Screen
`p_<name>` parameter · `c_<name>` checkbox · `r_<name>` radiobutton (group `<name>` ≤4 chars) · `s_<name>` select-option · `b_<name>` block.

## Methods & Parameters
- Patterns: `GET_`/`SET_<attr>`, `ON_<event>`, `AS_<type>`, `IS_<adj>`, `CHECK_<objective>`.
- Params: Import `I_` · Export `E_` · Table (FM) `T_` · Changing `C_` · Returning `R_`.

## DDIC
| Object | Convention | Example |
|---|---|---|
| DB Table | `ZD<DT>_<name>[T]` | `ZDD_TRQPTY` |
| View | `ZV<VT>_<name>` | `ZVP_TRQPTY` |
| Structure / Table Type | `ZST_<name>` / `ZT_<name>` | `ZST_TRQPTY` / `ZT_TRQPTY` |
| Domain | `Z<name>` | `ZTRQPTY` |
| Search Help | `ZSH_<name>` | `ZSH_TRQPTY` |
| Lock Object | `EZ_<name>` | `EZAOD_TAGLIB` |
| DDL/DCL (CDS) | `Z<VT>_<name>` | `ZI_TransportationOrder` |
| SQL DB View | `ZSQL<VT>_<name>` | `ZSQLITRANSPORTORDER` |
| Append Structure / Field | `ZZ<name>` | `ZZFWO_INTREF` |

`<DT>`: `C` customizing · `D` transaction · `M` master (+`T` = text table). `<VT>`: `D` database · `P` projection · `H` help · `M` maintenance · `I` interface/CDS · `R` root/base · `C` projection-CDS · `A` abstract · `CE` custom entity.

## RAP
| Object | Convention | Example |
|---|---|---|
| Behaviour Definition | `Z<VT>_<name>` | `ZI_TransportationOrder` |
| Behaviour Pool | `ZCL_BP_<name>` | `ZCL_BP_TransportationOrder` |
| Local Handler / Saver | `LHC_<name>` / `LSC_<name>` | `lhc_transportation_order` |
| Metadata Extension | `Z<VT>_<name>` | `ZC_TransportationOrder` |
| Service Definition | `Z<ST>_<name>` | `ZUI_TransportationOrder` |
| Service Binding | `Z<ST>_<name>_<PC>` | `ZUI_TransportationOrder_O4` |

`<ST>`: `UI` / `API`. `<PC>`: `O2` (OData V2) / `O4` (OData V4).

## Enhancements
Spot `ZES_` · Composite Spot `ZCES_` · BAdI Def `ZBD_` · BAdI Impl `ZBI_` · Enh Impl `ZEI_` · Composite Enh Impl `ZCEI_` · CMOD Project `Z<name>` (≤8). Example: `ZES_CARRIER_FACTOR`.

## BOBF / Business Objects
| Object | Convention | Example |
|---|---|---|
| BO Enhancement / Node | `ZENH_<name>` / `Z<name>` | `ZENH_TOR` / `ZBENEFIT_METRIC` |
| Data Structure (+ transient `_TR`, combined `_K`) | `ZST_<name>[_TR\|_K]` | `ZST_TOR_BENEFIT_METRIC` |
| Combined Node Table Type | `ZT_<name>_K` | `ZT_TOR_BENEFIT_METRIC_K` |
| Node Extension Include | `ZST_EEW_<name>` | `ZST_EEW_TOR_BENEFIT_METRIC` |
| Action / Determination / Validation / Query | `Z<name>` | `ZOPTIMIZE` |
| Class: Action / Determ / Valid / Query | `ZCL_A_<BO>_<name>` / `ZCL_D_<BO>_<node>_<name>` / `ZCL_V_<name>` / `ZCL_Q_<name>` | `ZCL_A_TOR_BENEFIT_METRIC_OPTIMIZE` |

**Special classes**: Controller `ZCL_UI_<name>_CONTROLLER` · View Exit `ZCL_UI_VIEWEXIT_<name>` · UI Helper `ZCL_UI_HELPER_<name>` · Conversion `ZCL_UI_CONVERSION_<name>` · Common Helper `ZCL_<name>_HELPER` · Bootstrap `ZCL_<name>_BOOTSTRAP` · Exception `ZCX_<name>` · Field Control `ZCL_<name>_FC` · Cross BO/Node `ZCL_C_<name>`.
**Conditions**: Condition `ZCOND_<name>[_<var>]` · Condition Type / Data Access Def `Z_<object>_<attribute>`.

## BRF+
- Main: App `ZA_` · Function `ZF_` · Ruleset `ZRS_` · Rule `ZR_` · Catalog `ZC_` · Object Filter `ZOF_`.
- Data: Data Element `ZE_` · Structure `ZS_` · Table `ZT_`.
- Expressions: Boolean `ZB_` · Constant `ZCO_` · Decision Table `ZDT_` · Decision Tree `ZDR_` · Formula `ZFO_` · Function Call `ZFC_` · Step Sequence `ZSS_`.
- Actions: Procedure Call `ZPC_` · Log Message `ZLM_` · Send Email `ZSE_` · Start Workflow `ZSW_`.

## FIORI / UI5
Header pkg `ZUI` · Common `ZUI_COMMON` · Sub `ZUI_<App>` · Namespace `mycompany.<MD>.<App>` · Freestyle app `ZUI_<App>` · Adaptation `ZADP_<App>` · Semantic Object `#Z<Entity>` · Git repo `ZUI_<App>`. Example: `maersk.sd.socreateapp`, `ZUI_SOCREATE`.

## WebDynpro
App `Z<MD>_<name>` · Component `Z_<name>` · App Config `<App>_<variant>` · Comp Config `ZWDCC_<name>` · Custom Controller `CC_<name>` · Window `W_<name>` · View `V_<name>` · Comp Usage `U_<comp>` · Inbound/Outbound Plug `IP_<name>`/`OP_<name>`.
**View elements**: Button `BTN_` · Checkbox `CHK_` · DropDownByIndex `DDI_` · DropDownByKey `DDK_` · Group `GRP_` · InputField `INP_` · Label `LBL_` · Table `TBL_` · TextView `TXV_` · TextEdit `TXE_` · RadioButton `RBT_` · TransparentContainer `TCO_`.

## Authorisation
Class `Z<MD>` · Object `Z<MD>_<name>` · Auth Group: tables `Z<MD><C>_<name>` (14), client-specific `Z<M><name>` (4), programs `Z<MD>_<name>` (8). Example: `ZSD_LAST_MILE`.

## IDoc
Extension `Z<basic_type>EXT` · Segment/IDoc/Message Type `Z<name>` · Partner Profile `<Logical System>`. Example: `ZARTMAS05EXT`, `ZARTMAS`.

## Launchpad
Tile field `<MD><C>_<T>_<name>` · Catalogue `ZCAT_<MD>_<#>_<name>` · Group `ZGRP_<MD>_<#>_<name>`. `<T>`: `C` custom / `S` standard. Example: `ZCAT_ATR_1005_MACCRUALRPTUT`.

## Agent Behaviour
1. Validate names before generating; flag & correct deviations first.
2. Ask for `<MD>` when required and not given.
3. Assume Clean ABAP unless told it's a legacy system.
4. Don't invent conventions; if uncovered, say so and suggest a consistent pattern.
5. Briefly note which convention each generated name follows.
6. Prioritise correctness over brevity.
7. Flag type-encoding prefixes (`lv_`/`lt_`/`lo_`/`ls_`) on locals; preserve role prefixes (`I_`/`E_`/`p_`/`s_`/`CO_`/…).
