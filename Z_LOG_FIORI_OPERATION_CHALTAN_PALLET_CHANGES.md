# Z_LOG_FIORI_OPERATION — Suggested Code Changes

**Function Module:** `Z_LOG_FIORI_OPERATION`  
**Function Group:** `ZFIO_RI_RPU_DASH`  
**System:** RD2

## Summary

| # | Change | Detail |
|---|--------|--------|
| 1 | VBAK check (~line 90) | Pass `I_STOGRN-STO_NO` to `VBAK-VBELN`; new logic runs only when entry exists |
| 2 | Movement 322 | Call `BAPI_GOODSMVT_CREATE` for MT 322 (mirror of existing 321 logic) before 321 |
| 3 | Movement 311 | Replace MT `951` with `311` (sloc-to-sloc) when VBAK entry found; receiving sloc from `ZLOG_EXEC_VAR` |

## Prerequisite — Configuration

Add plant-specific entry in `ZLOG_EXEC_VAR` for receiving storage location (value in **REMARKS**):

| NAME | WERKS | REMARKS | ACTIVE |
|------|-------|---------|--------|
| ZSCM_RECEVING_SLOC | &lt;plant&gt; | &lt;receiving sloc e.g. 7903&gt; | X |

Existing entries `EMPTY_PALL_RCVLOC` and `EMPTY_PALL_UNREST_STK` (LGORT) remain unchanged.

---

## CH-01 — Data declarations

**Location:** Declaration section, after existing `lv_vbeln` and config block variables

```abap
" Begin: Chaltan pallet - VBAK check and 322/311 variables
DATA: lv_sto_vbeln   TYPE vbak-vbeln,
      lv_sto_found   TYPE abap_bool,
      lv_matdoc_322  TYPE bapi2017_gm_head_ret-mat_doc,
      lv_value3_op   TYPE lgort_d,
      lt_return_322  TYPE TABLE OF bapiret2.

CONSTANTS: lc_name_rcv_sloc_op TYPE rvari_vnam VALUE 'ZSCM_RECEVING_SLOC'.
" End: Chaltan pallet
```

---

## CH-02 — VBAK validation and receiving SLOC read

**Location:** After config validation block (~line 90), before `IF i_stogrn IS NOT INITIAL`

```abap
" Begin: Chaltan pallet - Validate STO in VBAK
CLEAR: lv_sto_found, lv_sto_vbeln, lv_value3_op.
lv_sto_found = abap_false.

IF i_stogrn-sto_no IS NOT INITIAL.
  lv_sto_vbeln = i_stogrn-sto_no.
  CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
    EXPORTING
      input  = lv_sto_vbeln
    IMPORTING
      output = lv_sto_vbeln.
  SELECT SINGLE vbeln
    FROM vbak
    INTO lv_sto_vbeln
    WHERE vbeln = lv_sto_vbeln.
  IF sy-subrc = 0.
    lv_sto_found = abap_true.
  ENDIF.
ENDIF.

IF lv_sto_found = abap_true AND lv_werks_op IS NOT INITIAL.
  SELECT SINGLE remarks
    FROM zlog_exec_var
    INTO lv_value3_op
    WHERE name   = lc_name_rcv_sloc_op
      AND werks  = lv_werks_op
      AND active = lc_active_op.
  IF sy-subrc <> 0 OR lv_value3_op IS INITIAL.
    CLEAR lw_return.
    lw_return-type = 'E'.
    CONCATENATE 'Receiving storage location not configured for plant'(007)
                lv_werks_op INTO lw_return-message SEPARATED BY space.
    lw_return-message_v1 = lv_werks_op.
    APPEND lw_return TO et_return.
    RETURN.
  ENDIF.
ENDIF.
" End: Chaltan pallet
```

---

## CH-03 — BAPI_GOODSMVT_CREATE for movement type 322

**Location:** Inside `IF sy-subrc = 0` (delivery found), **before** existing `IF i_stogrn-good_pallet IS NOT INITIAL` block for 321

```abap
" Begin: Chaltan pallet - 322 goods movement (before 321)
IF lv_sto_found = abap_true AND i_stogrn-good_pallet IS NOT INITIAL.
  CLEAR: lt_item, lt_return_322, lv_matdoc_322.
  lw_goodsmvt_code-gm_code = '04'.
  lv_qty = i_stogrn-good_pallet.
  lw_item-material   = i_stogrn-material_code.
  lw_item-plant      = i_stogrn-plant.
  lw_item-stge_loc   = lv_value2_op.
  lw_item-move_stloc = lv_value1_op.
  lw_item-stck_type  = 'X'.
  lw_item-entry_qnt  = lv_qty.
  lw_item-entry_uom  = i_stogrn-uom.
  lw_item-deliv_numb = lw_del.
  lw_item-deliv_item = lw_delitm.
  lw_item-move_type  = '322'.
  APPEND lw_item TO lt_item.
  CLEAR lw_item.

  CALL FUNCTION 'BAPI_GOODSMVT_CREATE'
    EXPORTING
      goodsmvt_header  = lw_goodsmvt_header
      goodsmvt_code    = lw_goodsmvt_code
    IMPORTING
      materialdocument = lv_matdoc_322
    TABLES
      goodsmvt_item    = lt_item
      return           = lt_return_322.

  CLEAR lw_return.
  READ TABLE lt_return_322 INTO lw_return WITH KEY type = gc_e.
  IF sy-subrc = 0.
    gw_return-type    = lw_return-type.
    gw_return-message = lw_return-message.
    gw_return-id      = lw_return-id.
    gw_return-number  = lw_return-number.
    APPEND gw_return TO et_return.
    CLEAR gw_return.
    CALL FUNCTION 'BAPI_TRANSACTION_ROLLBACK'.
    RETURN.
  ELSE.
    CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'
      EXPORTING
        wait = abap_true.
    IF lv_matdoc_322 IS NOT INITIAL.
      UPDATE mseg CLIENT SPECIFIED SET xblnr_mkpf = lw_del
        WHERE mandt = sy-mandt
          AND mblnr = lv_matdoc_322.
      IF sy-subrc = 0.
        COMMIT WORK.
        UPDATE mkpf CLIENT SPECIFIED SET xblnr = lw_del
          WHERE mandt = sy-mandt
            AND mblnr = lv_matdoc_322.
        IF sy-subrc = 0.
          COMMIT WORK.
        ELSE.
          ROLLBACK WORK.
        ENDIF.
      ELSE.
        ROLLBACK WORK.
      ENDIF.
    ENDIF.
  ENDIF.
ENDIF.
" End: Chaltan pallet
```

**322 vs 321 sloc mapping**

| Field | 321 (existing) | 322 (new) |
|-------|----------------|-----------|
| `stge_loc` | `lv_value1_op` (rcv/blocked) | `lv_value2_op` (unrestricted) |
| `move_stloc` | `lv_value2_op` (unrestricted) | `lv_value1_op` (rcv/blocked) |
| `stck_type` | `X` | `X` |
| `gm_code` | `04` | `04` |

---

## CH-04 — Replace movement 951 with 311 (when VBAK found)

**Location:** `IF i_stogrn-req_pallet IS NOT INITIAL` block (~line 197)

Replace the block that sets `move_type = '951'` with conditional logic:

```abap
IF i_stogrn-req_pallet IS NOT INITIAL.
  CLEAR lt_item.

  IF lv_sto_found = abap_true.
    " Begin: Chaltan pallet - 311 sloc-to-sloc transfer
    lw_goodsmvt_code-gm_code = '04'.
    lw_item-material   = i_stogrn-material_code.
    lw_item-plant      = i_stogrn-plant.
    lw_item-stge_loc   = lv_value1_op.
    lw_item-move_stloc = lv_value3_op.
    lw_item-entry_qnt  = i_stogrn-req_pallet.
    lw_item-entry_uom  = i_stogrn-uom.
    lw_item-deliv_numb = lw_del.
    lw_item-deliv_item = lw_delitm.
    lw_item-move_type  = '311'.
    " End: Chaltan pallet
  ELSE.
    lw_goodsmvt_code-gm_code = '03'.
    lw_item-material   = i_stogrn-material_code.
    lw_item-plant      = i_stogrn-plant.
    lw_item-stge_loc   = lv_value1_op.
    lw_item-stck_type  = 'X'.
    lw_item-entry_qnt  = i_stogrn-req_pallet.
    lw_item-entry_uom  = i_stogrn-uom.
    lw_item-deliv_numb = lw_del.
    lw_item-deliv_item = lw_delitm.
    lw_item-move_type  = '951'.
  ENDIF.

  APPEND lw_item TO lt_item.
  CLEAR lw_item.

  CALL FUNCTION 'BAPI_GOODSMVT_CREATE'
    EXPORTING
      goodsmvt_header  = lw_goodsmvt_header
      goodsmvt_code    = lw_goodsmvt_code
    IMPORTING
      materialdocument = lv_matdoc1
    TABLES
      goodsmvt_item    = lt_item
      return           = lt_return1.

  " ... existing error handling and MSEG/MKPF update unchanged ...
ENDIF.
```

**311 sloc mapping**

| Field | Value |
|-------|-------|
| `stge_loc` | `lv_value1_op` (source — EMPTY_PALL_RCVLOC) |
| `move_stloc` | `lv_value3_op` (destination — ZSCM_RECEVING_SLOC remarks) |
| `gm_code` | `04` |
| `move_type` | `311` |

---

## Behaviour Matrix

| VBAK entry | good_pallet | req_pallet | Movements posted |
|------------|-------------|------------|------------------|
| Found | &gt; 0 | &gt; 0 | 322 → 321 → 311 |
| Found | &gt; 0 | 0 | 322 → 321 |
| Found | 0 | &gt; 0 | 311 only |
| Not found | &gt; 0 | &gt; 0 | 321 → 951 (unchanged) |
| Not found | &gt; 0 | 0 | 321 only |
| Not found | 0 | &gt; 0 | 951 only |
