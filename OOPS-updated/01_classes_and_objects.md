# 01 — Classes & Objects (Foundation)

> The absolute fundamentals. Every other topic builds on this.

---

## Class vs Object

A **class** is a blueprint defined at design time. An **object** is a concrete instance created at runtime with `NEW`. One class produces many objects, each with its own independent state.

```abap
" zcl_sales_order is the class (blueprint)
" lo_order1 and lo_order2 are two independent objects
DATA(lo_order1) = NEW zcl_sales_order( iv_vbeln = '0000001000' ).
DATA(lo_order2) = NEW zcl_sales_order( iv_vbeln = '0000002000' ).

" Each object holds its own data - changing one does not affect the other
```

---

## DEFINITION vs IMPLEMENTATION

Every class is written in two blocks. The `DEFINITION` declares the structure (the "what"); the `IMPLEMENTATION` contains the executable code (the "how").

```abap
CLASS zcl_sales_order DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING iv_vbeln TYPE vbeln,
             get_net_value RETURNING VALUE(rv_netwr) TYPE netwr.
  PRIVATE SECTION.
    DATA: mv_vbeln TYPE vbeln,
          ms_header TYPE vbak.
ENDCLASS.

CLASS zcl_sales_order IMPLEMENTATION.
  METHOD constructor.
    mv_vbeln = iv_vbeln.
    SELECT SINGLE * FROM vbak
      WHERE vbeln = @mv_vbeln
      INTO @ms_header.
  ENDMETHOD.

  METHOD get_net_value.
    rv_netwr = ms_header-netwr.
  ENDMETHOD.
ENDCLASS.
```

- **Global class** — created in SE24 / ADT, stored in the repository, reusable everywhere. Prefix `ZCL_`.
- **Local class** — defined inside a program (report, function group, or class test include), visible only there. Prefix `LCL_`.

---

## Components of a class

| Component | Keyword | Purpose |
|-----------|---------|---------|
| Attributes | `DATA` / `CLASS-DATA` | Store state |
| Methods | `METHODS` / `CLASS-METHODS` | Define behavior |
| Events | `EVENTS` / `CLASS-EVENTS` | Trigger reactions (see File 05) |
| Types | `TYPES` | Type definitions local to the class |
| Constants | `CONSTANTS` | Fixed compile-time values |
| Interfaces | `INTERFACES` | Contracts the class implements (see File 04) |

---

## Visibility sections

| Section | External programs | Subclasses | The class itself |
|---------|:---:|:---:|:---:|
| `PUBLIC` | yes | yes | yes |
| `PROTECTED` | no | yes | yes |
| `PRIVATE` | no | no | yes |

```abap
CLASS zcl_material DEFINITION.
  PUBLIC SECTION.
    METHODS: get_description RETURNING VALUE(rv_maktx) TYPE maktx.
    DATA: mv_matnr TYPE matnr READ-ONLY.   " readable outside, writable only inside

  PROTECTED SECTION.
    METHODS: is_price_valid IMPORTING iv_price TYPE netpr
                            RETURNING VALUE(rv_valid) TYPE abap_bool.
    DATA: mv_price TYPE netpr.              " subclasses may use this

  PRIVATE SECTION.
    METHODS: write_change_log.              " internal only - not even subclasses
    DATA: mv_internal_flag TYPE abap_bool.
ENDCLASS.
```

`READ-ONLY` lets external code read an attribute but not change it; only the class (and subclasses, for protected) can write it.

**Why this matters in SAP:** standard classes deliberately keep critical logic `PRIVATE`. That is why you often cannot override standard behavior by inheritance — the method is locked away on purpose, pushing you toward official extension points (BADIs, events) instead.

---

## Class categories

All classes share the same syntax, but in practice they fall into four categories by purpose. This is a common interview opener ("what kinds of classes exist in ABAP?").

| Category | Recognized by | Purpose | Covered in |
|----------|---------------|---------|------------|
| **Usual (normal) class** | nothing special | Everyday business logic, services, models | this file |
| **Exception class** | inherits from `CX_STATIC_CHECK` / `CX_DYNAMIC_CHECK` / `CX_NO_CHECK` | Typed error raising/handling | File 06 |
| **Persistence class** | flagged *Persistent* (Object Services) | OO mapping of DB rows to objects | File 10 |
| **Test class** | local class marked `FOR TESTING` | Unit tests for a class under test | File 09 |

These are roles, not separate syntax types — under the hood they are all ABAP classes; the difference is what they inherit from or how they are flagged.

---

## Instance vs Static components

- **Instance** components (`DATA`, `METHODS`) belong to each object; accessed with the object reference and `->`.
- **Static** components (`CLASS-DATA`, `CLASS-METHODS`) belong to the class, exist exactly once, are shared by all objects; accessed with the class name and `=>`.

```abap
CLASS zcl_counter DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor.
    DATA: mv_id TYPE i READ-ONLY.            " unique per object
    CLASS-DATA: gv_total TYPE i READ-ONLY.   " shared across all objects
    CLASS-METHODS: get_total RETURNING VALUE(rv_total) TYPE i.
ENDCLASS.

CLASS zcl_counter IMPLEMENTATION.
  METHOD constructor.
    gv_total = gv_total + 1.   " increment shared counter
    mv_id    = gv_total.       " assign this object's unique id
  ENDMETHOD.

  METHOD get_total.
    rv_total = gv_total.
  ENDMETHOD.
ENDCLASS.

" Usage
DATA(lo_a) = NEW zcl_counter( ).   " mv_id = 1, gv_total = 1
DATA(lo_b) = NEW zcl_counter( ).   " mv_id = 2, gv_total = 2
DATA(lv)   = zcl_counter=>get_total( ).   " 2 - called via class name, no object
```

**Critical rule:** a static method cannot access instance attributes — there is no `me->` context inside it. It can only use `CLASS-DATA` and call other static methods.

---

## Instantiation

```abap
" Modern - preferred
DATA(lo_order) = NEW zcl_sales_order( iv_vbeln = '0000001000' ).

" Classic - still valid, identical effect
DATA: lo_order2 TYPE REF TO zcl_sales_order.
CREATE OBJECT lo_order2 EXPORTING iv_vbeln = '0000001000'.
```

You control who may instantiate with the `CREATE` addition in the definition:

```abap
CLASS zcl_logger DEFINITION CREATE PRIVATE.
  " only zcl_logger itself can run NEW zcl_logger( )
  " this is the first building block of the Singleton pattern (see File 07)
ENDCLASS.
```

`CREATE PUBLIC` (the default), `CREATE PROTECTED`, and `CREATE PRIVATE` restrict instantiation to everyone, subclasses, or the class itself respectively.

---

## Instance constructor

The method named `constructor` runs on every `NEW`. It may have `IMPORTING` parameters and a `RAISING` clause. If it raises an exception the object is not created and the reference stays unbound.

```abap
CLASS zcl_sales_order DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING iv_vbeln TYPE vbeln
                         RAISING   zcx_order_not_found.
  PRIVATE SECTION.
    DATA: ms_header TYPE vbak.
ENDCLASS.

CLASS zcl_sales_order IMPLEMENTATION.
  METHOD constructor.
    SELECT SINGLE * FROM vbak
      WHERE vbeln = @iv_vbeln
      INTO @ms_header.
    IF sy-subrc <> 0.
      RAISE EXCEPTION TYPE zcx_order_not_found.
    ENDIF.
  ENDMETHOD.
ENDCLASS.

" Caller must handle the possible failure
TRY.
    DATA(lo_order) = NEW zcl_sales_order( iv_vbeln = '0000001000' ).
  CATCH zcx_order_not_found.
    " lo_order is NOT bound here
ENDTRY.
```

---

## Static constructor

The method named `class_constructor` runs **once per program session**, automatically, the first time the class is touched in any way (creating an instance, calling a static method, or reading a static attribute). It takes no parameters and cannot raise exceptions. Use it for one-time setup.

```abap
CLASS zcl_config DEFINITION.
  PUBLIC SECTION.
    CLASS-METHODS: class_constructor,
                   get IMPORTING iv_key TYPE zconfig-key
                       RETURNING VALUE(rv_value) TYPE zconfig-value.
  PRIVATE SECTION.
    CLASS-DATA: gt_config TYPE SORTED TABLE OF zconfig WITH UNIQUE KEY key.
ENDCLASS.

CLASS zcl_config IMPLEMENTATION.
  METHOD class_constructor.
    SELECT * FROM zconfig INTO TABLE @gt_config.   " loaded once, reused by all calls
  ENDMETHOD.

  METHOD get.
    rv_value = VALUE #( gt_config[ key = iv_key ]-value OPTIONAL ).
  ENDMETHOD.
ENDCLASS.
```

---

## Destructor / object lifetime

ABAP has no explicit destructor. An object is removed by **garbage collection** once no reference points to it. You can drop a reference manually:

```abap
CLEAR lo_order.   " release this reference
FREE  lo_order.   " release reference and any bound internal data
```

---

## me-> self reference

Inside an instance method, `me->` refers to the current object. It is optional unless you need to disambiguate an attribute from a parameter or local variable of the same name.

```abap
METHOD set_name.
  me->mv_name = mv_name.   " me->mv_name is the attribute; mv_name is the parameter
ENDMETHOD.
```

---

## Quick self-test

1. What is the difference between a class and an object?
2. Where do global vs local classes live, and what are their prefixes?
3. Can a static method read an instance attribute? Why or why not?
4. When does `class_constructor` run, and how many times?
5. What does `CREATE PRIVATE` do, and which pattern does it enable?
6. How are objects destroyed in ABAP?
7. What does `READ-ONLY` allow and prevent?
8. Name the four class categories and what distinguishes each.
