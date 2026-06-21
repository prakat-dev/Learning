# 06 — Exception Handling (Class-Based)

> Modern, typed error handling. The exception hierarchy, TRY/CATCH/CLEANUP, custom exception classes, and resumable exceptions.

---

## Why class-based exceptions

Class-based exceptions replace the old `sy-subrc` / `RAISING`-in-function-module style. They are typed objects that can carry rich context (texts, attributes, the original cause), propagate automatically up the call stack, and be caught broadly or narrowly via a class hierarchy.

---

## The exception class hierarchy

Every exception class ultimately inherits from `CX_ROOT`. Three standard superclasses define how the compiler and runtime treat them.

| Superclass | Declaration / handling | Use for |
|------------|------------------------|---------|
| `CX_STATIC_CHECK` | Compiler enforces a `RAISING` clause and a `CATCH` (or re-raise) | Recoverable, expected errors the caller should handle |
| `CX_DYNAMIC_CHECK` | Checked only at runtime; no compiler enforcement | Errors avoidable by a prior check (e.g. a bad cast) |
| `CX_NO_CHECK` | Never declared; implicitly always possible | Serious system/runtime errors you usually cannot recover from |

```
CX_ROOT
 |- CX_STATIC_CHECK
 |- CX_DYNAMIC_CHECK
 |- CX_NO_CHECK
```

---

## TRY / CATCH / CLEANUP

```abap
TRY.
    DATA(lv_result) = lo_calc->divide( iv_num = 10 iv_den = 0 ).

  CATCH zcx_division_by_zero INTO DATA(lx_div).
    WRITE: / 'Division error:', lx_div->get_text( ).

  CATCH cx_root INTO DATA(lx_any).        " broad catch-all - must come LAST
    WRITE: / 'Unexpected error:', lx_any->get_text( ).

  CLEANUP.
    " runs only if an exception leaves this TRY block uncaught locally;
    " use it to release locks or other resources
    lo_calc->release_lock( ).
ENDTRY.
```

Catch specific exceptions before general ones — `cx_root` always last, otherwise it would swallow the more specific cases.

---

## Raising exceptions

```abap
" Simple raise
RAISE EXCEPTION TYPE zcx_order_not_found.

" Raise with a bound T100 message and context attribute
RAISE EXCEPTION TYPE zcx_order_not_found
  EXPORTING
    textid   = zcx_order_not_found=>order_missing
    mv_vbeln = iv_vbeln.

" Wrap and re-raise, preserving the original cause (exception chaining)
TRY.
    SELECT SINGLE * FROM vbak INTO @DATA(ls) WHERE vbeln = @iv_vbeln.
  CATCH cx_sy_open_sql_db INTO DATA(lx_db).
    RAISE EXCEPTION TYPE zcx_data_access_error
      EXPORTING previous = lx_db.
ENDTRY.
```

The `previous` attribute keeps the original exception, so you can trace the full root cause later with `lx->previous`.

---

## Declaring that a method raises

```abap
METHODS: divide IMPORTING iv_num TYPE i
                          iv_den TYPE i
                RETURNING VALUE(rv_result) TYPE i
                RAISING   zcx_division_by_zero.
```

For `CX_STATIC_CHECK` subclasses, the `RAISING` clause is mandatory and callers must catch or propagate. Mark a method's exception resumable with `RAISING RESUMABLE(zcx_warning)`.

---

## Creating a custom exception class

In SE24/ADT, create a class inheriting from `CX_STATIC_CHECK` (or dynamic/no-check), tick "with message class" if you want T100 message texts, and add attributes for context. A typical hand-written version:

```abap
CLASS zcx_order_not_found DEFINITION
  PUBLIC
  INHERITING FROM cx_static_check
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_t100_message.

    CONSTANTS:
      BEGIN OF order_missing,
        msgid TYPE symsgid VALUE 'ZORDER',
        msgno TYPE symsgno VALUE '001',
        attr1 TYPE scx_attrname VALUE 'MV_VBELN',
        attr2 TYPE scx_attrname VALUE '',
        attr3 TYPE scx_attrname VALUE '',
        attr4 TYPE scx_attrname VALUE '',
      END OF order_missing.

    DATA mv_vbeln TYPE vbeln READ-ONLY.

    METHODS constructor
      IMPORTING textid   LIKE if_t100_message=>t100key OPTIONAL
                previous TYPE REF TO cx_root           OPTIONAL
                mv_vbeln TYPE vbeln                    OPTIONAL.
ENDCLASS.

CLASS zcx_order_not_found IMPLEMENTATION.
  METHOD constructor.
    super->constructor( previous = previous ).
    me->mv_vbeln = mv_vbeln.
    CLEAR me->textid.
    IF textid IS INITIAL.
      if_t100_message~t100key = order_missing.
    ELSE.
      if_t100_message~t100key = textid.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```

`get_text( )` returns the resolved message text; `get_longtext( )` the long version. (When you create the class in SE24 with the exception-class wizard, most of this boilerplate is generated for you.)

---

## Resumable exceptions

A resumable exception lets the handler send control back to just after the `RAISE`, instead of unwinding the stack.

```abap
" Raiser
METHOD process_line.
  IF iv_qty = 0.
    RAISE RESUMABLE EXCEPTION TYPE zcx_zero_quantity.
  ENDIF.
  " ... continues here if the handler RESUMEs ...
ENDMETHOD.

" Handler
TRY.
    lo_proc->process_line( iv_qty = 0 ).
  CATCH BEFORE UNWIND zcx_zero_quantity.
    " BEFORE UNWIND keeps the stack so RESUME can return control
    RESUME.   " continue right after the RAISE in process_line
ENDTRY.
```

---

## Class-based vs classic exceptions

| Classic (function module) | Class-based |
|---------------------------|-------------|
| Declared `EXCEPTIONS`, checked via `sy-subrc` | Typed exception objects |
| Carries only a return code | Rich attributes + chaining via `previous` |
| Easy to ignore silently | Compiler-enforced for static-check types |
| Procedural | Object-oriented; propagates up the stack automatically |

---

## Best practices (Clean ABAP)

Throw specific exception types rather than generic `cx_root`. Use `CX_STATIC_CHECK` for situations the caller can reasonably recover from, and reserve `CX_NO_CHECK` for genuinely unrecoverable system errors. Always chain with `previous` when wrapping a lower-level exception. Do not catch what you cannot handle — let it propagate. Never leave an empty `CATCH` block that swallows the error silently.

---

## Quick self-test

1. Name the three standard exception superclasses and when each is used.
2. What is the difference between `CATCH` and `CLEANUP`?
3. Why must `cx_root` be the last `CATCH`?
4. What does the `previous` attribute give you?
5. What do `RAISE RESUMABLE` and `CATCH BEFORE UNWIND ... RESUME` enable?
6. Why is an empty `CATCH` block a bad idea?
