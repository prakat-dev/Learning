# 10 — Advanced Topics

> Friendship, persistence classes, shared objects, and OOP ALV with CL_SALV_TABLE.

---

## Friendship

A class grants another class access to its PRIVATE and PROTECTED members with `FRIENDS`. This deliberately breaks encapsulation, so use it sparingly.

```abap
CLASS zcl_engine DEFINITION FRIENDS zcl_mechanic.
  PRIVATE SECTION.
    DATA: mv_oil_level TYPE i.
ENDCLASS.

" zcl_mechanic can now read and write mv_oil_level directly
```

Friendship is **granted**, not taken, and is one-directional unless reciprocated. Subclasses of a friend inherit the friendship. The most common legitimate use is letting a local test class reach the private state of the class under test:

```abap
CLASS zcl_calc DEFINITION
  PUBLIC
  CREATE PUBLIC
  GLOBAL FRIENDS ltc_calc.   " the local test class may inspect private members
  ...
ENDCLASS.
```

---

## Persistence classes (Object Services)

A persistence class is SAP's built-in object-relational mapper. Instead of writing SELECT/INSERT/UPDATE/DELETE, you map a class to a database table and let the Persistence Service load rows into objects and save objects back.

**Setup:** in SE24 you create a class of type *Persistent class*, then use the Persistence Representation editor to map attributes to the fields of a table (say `ZEMPLOYEE` with key `PERNR`, plus `ENAME` and `SALARY`). SAP generates two classes:
- `ZCL_EMPLOYEE` — the persistent class, with generated `GET_*` / `SET_*` methods per mapped field.
- `ZCA_EMPLOYEE` — the **class agent** (a generated singleton accessed via `ZCA_EMPLOYEE=>AGENT`) used to create, read, and delete persistent objects.

```abap
" CREATE (insert)
DATA(lo_agent) = zca_employee=>agent.
TRY.
    DATA(lo_emp) = lo_agent->create_persistent( i_pernr = '00000001' ).
    lo_emp->set_ename( 'Alice Smith' ).
    lo_emp->set_salary( '50000.00' ).
  CATCH cx_os_object_existing.
    " a row with that key already exists
ENDTRY.
COMMIT WORK.

" READ (select by key)
TRY.
    DATA(lo_read) = zca_employee=>agent->get_persistent( i_pernr = '00000001' ).
    DATA(lv_name) = lo_read->get_ename( ).
  CATCH cx_os_object_not_found.
    " no such row
ENDTRY.

" UPDATE - just call a setter on a loaded object, then commit; no UPDATE statement
lo_read->set_salary( '55000.00' ).
COMMIT WORK.

" DELETE
zca_employee=>agent->delete_persistent( lo_read ).
COMMIT WORK.
```

You never write SQL. The Transaction Service tracks which objects changed and flushes them on `COMMIT WORK`. For explicit control you can drive the Transaction Service yourself:

```abap
DATA(lo_txn) = cl_os_system=>get_transaction_manager( )->create_transaction( ).
lo_txn->start( ).
" ... create / modify persistent objects ...
lo_txn->end( ).   " commits the unit of work
```

**Reality check:** persistence classes never became popular — they add overhead and give little control over the generated SQL, which hurts on large datasets. In S/4HANA the equivalent role is filled by **CDS views** (reading) and **RAP** managed behavior (full create/update/delete lifecycle). Recognize a persistence class and its `ZCA_*` agent if you see one in legacy code, but you would rarely build a new one. A good interview line: "it is the old Object Services ORM, superseded by CDS and RAP."

---

## Shared Objects (shared memory)

Shared objects store an object in shared memory so many sessions share one in-memory copy — ideal for caching read-mostly data such as configuration or large lookup tables.

Building blocks: a **shared memory area** is defined in transaction SHMA, which generates an area class (e.g. `ZCL_MY_AREA`); a **root class** is the entry-point object stored in the area. You attach for read (many concurrent readers) or for write/update (one exclusive writer).

```abap
" WRITE - build the cache once
DATA(lo_area) = zcl_my_area=>attach_for_write( ).
DATA(lo_root) = NEW zcl_my_root( ).
lo_root->load_config( ).             " expensive load, done once
lo_area->set_root( lo_root ).
lo_area->detach_commit( ).

" READ - fast access from shared memory, from any session
DATA(lo_read_area) = zcl_my_area=>attach_for_read( ).
DATA(lt_config)    = lo_read_area->root->get_config( ).
lo_read_area->detach( ).
```

Only one writer may attach at a time; readers see the last committed version. The payoff is loading expensive data once and serving many sessions from memory.

---

## OOP ALV with CL_SALV_TABLE

The modern, fully object-oriented ALV. You create it with a static `factory` method, then configure it through getter objects.

```abap
DATA: lt_flights TYPE STANDARD TABLE OF sflight.
SELECT * FROM sflight INTO TABLE @lt_flights UP TO 100 ROWS.

TRY.
    cl_salv_table=>factory(
      IMPORTING r_salv_table = DATA(lo_alv)
      CHANGING  t_table      = lt_flights ).

    lo_alv->get_functions( )->set_all( abap_true ).            " full toolbar
    lo_alv->get_display_settings( )->set_striped_pattern( abap_true ).

    DATA(lo_cols) = lo_alv->get_columns( ).
    lo_cols->set_optimize( abap_true ).                        " optimize widths
    DATA(lo_col) = CAST cl_salv_column_table( lo_cols->get_column( 'PRICE' ) ).
    lo_col->set_short_text( 'Price' ).

    lo_alv->display( ).
  CATCH cx_salv_msg cx_salv_not_found INTO DATA(lx_salv).
    MESSAGE lx_salv->get_text( ) TYPE 'I'.
ENDTRY.
```

Handling a double-click event:

```abap
" In a handler class:
CLASS lcl_handler DEFINITION.
  PUBLIC SECTION.
    METHODS: on_double_click
               FOR EVENT double_click OF cl_salv_events_table
               IMPORTING row column.
ENDCLASS.

" Registration:
DATA(lo_events) = lo_alv->get_event( ).
DATA(lo_handler) = NEW lcl_handler( ).
SET HANDLER lo_handler->on_double_click FOR lo_events.
```

Key classes in the SALV framework: `CL_SALV_TABLE` (the ALV), `CL_SALV_COLUMNS_TABLE` (all columns) and `CL_SALV_COLUMN_TABLE` (one column), `CL_SALV_FUNCTIONS_LIST` (toolbar), `CL_SALV_DISPLAY_SETTINGS` (title/striping), `CL_SALV_SORTS` and `CL_SALV_FILTERS`, and `CL_SALV_EVENTS_TABLE` (events).

> `CL_SALV_TABLE` is display-oriented (read-only grids) for full-screen, list, or popup output. For editable grids you still use the older `CL_GUI_ALV_GRID`.

---

## Quick self-test

1. Is friendship taken or granted? Is it bidirectional?
2. What is the main legitimate use of `FRIENDS`?
3. What does the persistence class agent (`ZCA_*`) provide, and how do you persist changes?
4. Why use shared objects, and what is the concurrency rule?
5. How do you create a `CL_SALV_TABLE` instance?
6. Can `CL_SALV_TABLE` produce editable grids? If not, what do you use instead?
