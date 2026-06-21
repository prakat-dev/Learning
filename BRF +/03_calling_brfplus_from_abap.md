# 03 — Calling BRF+ from ABAP

> How to invoke a BRF+ function from ABAP code, set the context, read the result, test it, and the performance trade-offs.

---

## Two ways to call a function

| API | Class / interface | When to use |
|-----|-------------------|-------------|
| **Instance-based** | `IF_FDT_FUNCTION` (via factory) | General use; full control; works with the function object model |
| **Static process** | `CL_FDT_FUNCTION_PROCESS=>PROCESS` | Performance-critical / many calls; no function instance needed |

Both need the **function ID** — a 32-character GUID that uniquely identifies the BRF+ function. You get it from the workbench (the function's General tab shows its ID), and crucially, **the workbench can generate the calling ABAP code for you** (function context menu → there's an option to show/generate sample ABAP code, including the GUID and the typed context structure). In practice you rarely hand-write this from scratch.

---

## Static call — CL_FDT_FUNCTION_PROCESS (recommended for performance)

The static `PROCESS` method runs a function by ID without instantiating a function object. This is the faster path for high-volume calls.

```abap
DATA: lv_function_id TYPE if_fdt_types=>id VALUE '0050568... (32-char GUID)',
      lr_context     TYPE REF TO data,
      lr_result      TYPE REF TO data,
      lo_message     TYPE REF TO if_fdt_result.

FIELD-SYMBOLS: <ls_context> TYPE any,
               <lv_field>   TYPE any,
               <lv_result>  TYPE any.

" 1) Get a typed reference for the context structure of this function
cl_fdt_function_process=>get_data_object_reference(
  EXPORTING
    iv_function_id        = lv_function_id
    iv_data_object        = '_V_CONTEXT'     " the function's context root
  IMPORTING
    er_data               = lr_context ).
ASSIGN lr_context->* TO <ls_context>.

" 2) Fill the context fields (the input parameters)
ASSIGN COMPONENT 'CUSTOMER_GROUP' OF STRUCTURE <ls_context> TO <lv_field>.
<lv_field> = 'VIP'.
ASSIGN COMPONENT 'PRODUCT_CATEGORY' OF STRUCTURE <ls_context> TO <lv_field>.
<lv_field> = 'ELEC'.
ASSIGN COMPONENT 'ORDER_VALUE' OF STRUCTURE <ls_context> TO <lv_field>.
<lv_field> = '12000.00'.

" 3) Process the function
TRY.
    cl_fdt_function_process=>process(
      EXPORTING
        iv_function_id = lv_function_id
        ia_context     = <ls_context>
      IMPORTING
        ea_result      = lr_result      " result returned here
        eo_message     = lo_message ).

    ASSIGN lr_result->* TO <lv_result>.
    " <lv_result> now holds DISCOUNT_PERCENT, e.g. 15
  CATCH cx_fdt INTO DATA(lx_fdt).
    " handle BRF+ processing error
    MESSAGE lx_fdt->get_text( ) TYPE 'E'.
ENDTRY.
```

> Exact parameter names and the context root name can vary slightly by release and by how the function signature was modeled. Always start from the **workbench-generated code** for your specific function — it produces the correct GUID, context structure, and parameter wiring.

---

## Instance-based call — IF_FDT_FUNCTION

The classic object-model approach: get the function instance from the factory, ask it for its context, fill it, then `process`.

```abap
DATA: lo_function TYPE REF TO if_fdt_function,
      lo_context  TYPE REF TO if_fdt_context,
      lo_result   TYPE REF TO if_fdt_result.

TRY.
    " get the function instance by its ID
    lo_function ?= cl_fdt_factory=>if_fdt_factory~get_instance(
                     )->get_function( lv_function_id ).

    " get and fill the context
    lo_context = lo_function->get_process_context( ).
    lo_context->set_value( iv_name = 'CUSTOMER_GROUP'   ia_value = 'VIP' ).
    lo_context->set_value( iv_name = 'ORDER_VALUE'      ia_value = '12000.00' ).

    " process
    lo_function->process(
      EXPORTING io_context = lo_context
      IMPORTING eo_result  = lo_result ).

    " read the result
    DATA lv_discount TYPE p DECIMALS 2.
    lo_result->get_value( IMPORTING ea_value = lv_discount ).
  CATCH cx_fdt INTO DATA(lx).
    " handle error
ENDTRY.
```

The instance API gives more control (reusable context objects, access to messages and trace) at the cost of slightly more overhead per call.

---

## Performance notes

- For **many calls in a loop or a high-volume scenario**, prefer `CL_FDT_FUNCTION_PROCESS=>PROCESS` — it avoids repeatedly building the function object instance.
- BRF+ has its own **buffering**; the first call in a session is the most expensive. Avoid re-fetching the function inside tight loops.
- Keep decision tables reasonably sized; very large tables evaluated row-by-row can be slow — consider a database-backed approach if a "table" is really master data.

---

## Testing and simulation in the workbench

You don't need ABAP to test a function. In the BRF+ workbench, open the function and use the **simulation** feature: enter context values directly, run, and inspect the result and (optionally) a **processing trace** that shows which rules and rows fired. This is the fastest way to validate rules before wiring up the ABAP call, and the trace is invaluable for debugging why a particular result came out.

---

## Versioning and transport

- BRF+ objects are transported like other repository objects (assigned to a package and transport request). Use `$TMP`/local for throwaway prototypes.
- Optional **versioning** can be switched on per object or application — useful where you must reproduce the decision logic as it was at a past date (e.g. legal/audit).

---

## Quick self-test

1. What two APIs can call a BRF+ function, and when do you choose each?
2. What identifies a BRF+ function to ABAP, and where do you get it?
3. Why is hand-writing the call rarely necessary?
4. Which API is preferred for high-volume calls, and why?
5. How do you test a function without writing any ABAP?
6. What does the processing trace help you do?
