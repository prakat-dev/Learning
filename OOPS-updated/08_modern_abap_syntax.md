# 08 — Modern ABAP Syntax (7.4 / 7.5+) for OOP

> The constructor operators and expressions you will see constantly in modern OO code. Fluency here is expected in any current ABAP role.

---

## Inline declarations

Declare a variable at the point of first use; the type is inferred.

```abap
DATA(lv_sum) = 1 + 2.                                   " typed as i
SELECT * FROM vbak INTO TABLE @DATA(lt_vbak) UP TO 10 ROWS.
LOOP AT lt_vbak INTO DATA(ls_vbak).                     " work area inline
  ASSIGN COMPONENT 'VBELN' OF STRUCTURE ls_vbak TO FIELD-SYMBOL(<lv>).
ENDLOOP.
```

---

## NEW — instance operator

Replaces `CREATE OBJECT`. Also creates data objects.

```abap
DATA(lo_order) = NEW zcl_sales_order( iv_vbeln = '0000001000' ).

DATA: li_sender TYPE REF TO zif_sender.
li_sender = NEW zcl_channel_rest( ).

DATA(lr_int) = NEW i( 42 ).   " anonymous data object, lr_int is a data reference
```

---

## VALUE — build structures and internal tables

```abap
" Structure
DATA(ls_addr) = VALUE zaddr( name = 'ACME' city = 'Berlin' ).

" Internal table of scalars
DATA(lt_ids) = VALUE int4_tab( ( 1 ) ( 2 ) ( 3 ) ).

" Extend an existing table, keeping current rows
lt_ids = VALUE #( BASE lt_ids ( 4 ) ( 5 ) ).

" Table of structures, including a nested table
DATA(lt_orders) = VALUE ztt_order(
  ( vbeln = '0000000001'
    items = VALUE ztt_item( ( posnr = '000010' ) ( posnr = '000020' ) ) ) ).
```

---

## CONV — convert between types

```abap
DATA(lv_str)    = CONV string( 42 ).
DATA(lv_packed) = CONV dmbtr( '100.50' ).
```

---

## CORRESPONDING — map by field name

```abap
" Same-named fields copied automatically
DATA(ls_target) = CORRESPONDING zts_target( ls_source ).

" Rename with MAPPING, skip fields with EXCEPT
DATA(lt_target) = CORRESPONDING ztt_target( lt_source
                    MAPPING target_field = source_field
                    EXCEPT  unwanted_field ).
```

---

## COND — multi-branch conditional (if / elseif / else)

```abap
DATA(lv_text) = COND string(
  WHEN sy-subrc = 0 THEN 'Success'
  WHEN sy-subrc = 4 THEN 'No data found'
  ELSE                   'Error' ).
```

## SWITCH — branch on a single value (case)

```abap
DATA(lv_day) = SWITCH string( iv_weekday
  WHEN 1 THEN 'Monday'
  WHEN 2 THEN 'Tuesday'
  ELSE        'Other' ).
```

---

## REDUCE — collapse a set into one value

```abap
" Sum a column
DATA(lv_total) = REDUCE dmbtr(
  INIT sum = 0
  FOR  ls IN lt_items
  NEXT sum = sum + ls-netwr ).

" Conditional accumulation (debit minus credit)
DATA(lv_balance) = REDUCE dmbtr(
  INIT bal = 0
  FOR  wa IN lt_lines WHERE ( belnr = '0000000100' )
  NEXT bal = COND #( WHEN wa-shkzg = 'S' THEN bal + wa-dmbtr
                     ELSE                     bal - wa-dmbtr ) ).
```

---

## FOR — iterate inside VALUE / REDUCE

```abap
" Project one table into another (extract a single column)
DATA(lt_names) = VALUE string_table(
  FOR ls IN lt_customers ( CONV string( ls-name1 ) ) ).

" Copy only rows matching a condition
DATA(lt_big_orders) = VALUE ztt_order(
  FOR ls IN lt_orders WHERE ( netwr > 1000 ) ( ls ) ).
```

---

## FILTER — subset an internal table

`FILTER` needs the table to have a sorted or hashed key (or a secondary key) on the filtered field.

```abap
" Table must have an appropriate key on SPRAS
DATA lt_msg TYPE SORTED TABLE OF zmsg WITH NON-UNIQUE KEY spras.
" ... fill lt_msg ...

DATA(lt_en)     = FILTER #( lt_msg WHERE spras = 'E' ).            " keep matches
DATA(lt_non_en) = FILTER #( lt_msg EXCEPT WHERE spras = 'E' ).     " drop matches
```

---

## Table expressions and existence checks

```abap
" Direct row read - raises CX_SY_ITAB_LINE_NOT_FOUND if the row is missing
DATA(ls_row) = lt_orders[ vbeln = '0000001000' ].

" Guard against a missing row
IF line_exists( lt_orders[ vbeln = '0000001000' ] ).
  DATA(lv_index) = line_index( lt_orders[ vbeln = '0000001000' ] ).
ENDIF.

" Or read with OPTIONAL to get an initial structure instead of a dump
DATA(ls_safe) = VALUE #( lt_orders[ vbeln = '0000009999' ] OPTIONAL ).
```

> A missing row in an inline table expression short-dumps. Use `line_exists( )` first, or `VALUE #( ... OPTIONAL )`, or read into a field symbol and check `sy-subrc`.

---

## REF — obtain a reference

```abap
DATA(lr_struct) = REF #( ls_addr ).   " data reference to an existing object
```

---

## LET — local helper inside an expression

```abap
DATA(lv_msg) = COND string(
  LET prefix = 'Order ' IN
  WHEN iv_ok = abap_true THEN prefix && 'succeeded'
  ELSE                        prefix && 'failed' ).
```

---

## String templates

```abap
DATA(lv_line)  = |Order { lv_vbeln } total { lv_total }|.
DATA(lv_matnr) = |{ lv_matnr ALPHA = OUT }|.   " strip leading zeros
DATA(lv_date)  = |{ sy-datum DATE = USER }|.   " format per user settings
```

---

## Why this matters for OOP

Constructor operators make object and data setup concise, and functional (returning) methods combine with inline declarations into clean, chainable calls:

```abap
DATA(lt_open) = zcl_order_service=>get_instance( )->read_open_orders( iv_kunnr = '0001000001' ).
```

Clean ABAP encourages these expressions over verbose classic statements — but always favor readability over cleverness; deeply nested `REDUCE`/`FOR` can become hard to follow.

---

## Quick self-test

1. What happens if a table expression `lt[ key = x ]` finds no matching row, and what are three safe alternatives?
2. `COND` vs `SWITCH` — when do you use each?
3. What does `VALUE #( BASE lt ... )` do?
4. What key requirement does `FILTER` impose on the internal table?
5. How does `CORRESPONDING` handle differently-named fields?
6. Write a `REDUCE` that sums the `netwr` column of `lt_items`.
