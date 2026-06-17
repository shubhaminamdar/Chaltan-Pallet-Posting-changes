# Chaltan Pallet — Z_LOG_FIORI_RECLASSIFICATION

**Function Module:** `Z_LOG_FIORI_RECLASSIFICATION`  
**Function Group:** `ZFIO_RI_RPU_DASH`  
**System:** RD2 (source verified via SAP MCP)

## Summary

Add quantity calculation logic for movement types **622**, **321**, and **311**, mirroring the existing logic at line ~700 for **322** and **321**.

| Movement Type | Operation | Used In |
|---------------|-----------|---------|
| 322 | Add (+) | Existing |
| 321 | Subtract (−) | Existing and new 622 block |
| 622 | Add (+) | **New** |
| 311 | Subtract (−) | **New** |

**Formula (new):** `total_qty = Σ(622) − Σ(321) − Σ(311)`

Output is appended as a separate row with `move_type = '622'`.

## Prerequisite — Configuration

Movement types are filtered via `lr_bwart[]`, built from `ZLOG_EXEC_VAR` (`ZSCM_CUST_RET_MVT`).

Add active entries for **622** and **311** in `ZLOG_EXEC_VAR` (322 and 321 should already exist):

| NAME | BWART | ACTIVE |
|------|-------|--------|
| ZSCM_CUST_RET_MVT | 622 | X |
| ZSCM_CUST_RET_MVT | 321 | X |
| ZSCM_CUST_RET_MVT | 311 | X |

Without this, `lt_mseg_cl` will not contain 622/311 lines and the new calculation will remain zero.

---

## CH-01 — Data declarations

**Location:** CD:8088769 declaration section (alongside existing `lv_menge_322`, `lv_menge_321`, etc.)

```abap
" Begin: Chaltan pallet - Qty vars for 622/321/311
DATA: lv_menge_622  TYPE menge_d,
      lv_menge_321n TYPE menge_d,
      lv_menge_311  TYPE menge_d,
      lv_matdoc_622 TYPE mblnr,
      lv_meins_622  TYPE meins,
      lw_recls_622  TYPE zst_fiori_reclass_out.
" End: Chaltan pallet
```

---

## CH-02 — Clear accumulators per lips iteration

**Location:** Immediately after `LOOP AT lt_lips INTO lw_lips.`

```abap
" Begin: Chaltan pallet - Clear 622/321/311 accumulators
CLEAR: lv_menge_622,
       lv_menge_321n,
       lv_menge_311,
       lv_matdoc_622,
       lv_meins_622,
       lw_recls_622.
" End: Chaltan pallet
```

---

## CH-03 — Quantity logic inside inner `lt_mseg_cl` read block

**Location:** After existing 322/321 logic, inside the `IF sy-subrc = 0` block following `READ TABLE lt_mseg_cl`

```abap
**************************************************************
" Begin: Chaltan pallet - Qty calculation for 622/321/311
IF lw_mseg_cl-bwart = '622'.
  lv_matdoc_622 = lw_mkpf_cl-mblnr.
  lv_menge_622  = lv_menge_622 + lw_mseg_cl-menge.
ENDIF.

IF lw_mseg_cl-bwart = '321'.
  lv_menge_321n = lv_menge_321n + lw_mseg_cl-menge.
ENDIF.

IF lw_mseg_cl-bwart = '311'.
  lv_menge_311  = lv_menge_311 + lw_mseg_cl-menge.
ENDIF.

IF lw_mseg_cl-bwart = '622'
OR lw_mseg_cl-bwart = '321'
OR lw_mseg_cl-bwart = '311'.
  lv_meins_622 = lw_mseg_cl-meins.
ENDIF.
" End: Chaltan pallet
**************************************************************
```

---

## CH-04 — Append 622 output record

**Location:** After existing `APPEND lw_recls_data TO et_recls_data` for move type 322, before `CLEAR lw_recls_data`

```abap
lw_recls_data-move_type = '322'.
APPEND lw_recls_data TO et_recls_data.

" Begin: Chaltan pallet - Append 622 reclassification output
lw_recls_622-delivery       = lw_recls_data-delivery.
lw_recls_622-material_code  = lw_recls_data-material_code.
lw_recls_622-material_desc  = lw_recls_data-material_desc.
lw_recls_622-sto_no         = lw_recls_data-sto_no.
lw_recls_622-sto_item       = lw_recls_data-sto_item.
lw_recls_622-req_pallet     = 0.
lw_recls_622-good_pallet    = 0.
lw_recls_622-scraps         = 0.
lw_recls_622-matdoc         = lv_matdoc_622.
lw_recls_622-uom            = lv_meins_622.
lw_recls_622-move_type      = '622'.
lw_recls_622-total_qty      = lv_menge_622 - lv_menge_321n - lv_menge_311.

IF lw_recls_622-total_qty > 0.
  APPEND lw_recls_622 TO et_recls_data.
ENDIF.
" End: Chaltan pallet

CLEAR: lw_lips, lw_recls_data.
```

**Note:** The existing filter `DELETE et_recls_data WHERE total_qty LE '0.000'.` also applies to 622 rows.

---

## Reference — Existing logic (322 / 321)

```abap
IF lw_mseg_cl-bwart = '322'.
  lw_recls_data-matdoc = lw_mkpf_cl-mblnr.
  lw_recls_data-total_qty = ( lw_recls_data-total_qty + lw_mseg_cl-menge ).
ENDIF.

IF lw_mseg_cl-bwart = '321'.
  lw_recls_data-total_qty = ( lw_recls_data-total_qty - lw_mseg_cl-menge ).
ENDIF.
lw_recls_data-uom = lw_mseg_cl-meins.
```

**Formula (existing):** `total_qty = Σ(322) − Σ(321)`
