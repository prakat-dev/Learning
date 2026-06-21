# 03 — Inheritance & Polymorphism

> Single inheritance, overriding, abstract/final, casting, and runtime polymorphism — with complete, compiling examples.

---

## Basic inheritance

```abap
CLASS zcl_document DEFINITION.
  PUBLIC SECTION.
    METHODS: get_type RETURNING VALUE(rv_type) TYPE string.
  PROTECTED SECTION.
    DATA: mv_id TYPE string.
ENDCLASS.

CLASS zcl_document IMPLEMENTATION.
  METHOD get_type.
    rv_type = 'Generic document'.
  ENDMETHOD.
ENDCLASS.

CLASS zcl_invoice DEFINITION INHERITING FROM zcl_document.
  PUBLIC SECTION.
    METHODS: get_type REDEFINITION.   " override the parent's version
ENDCLASS.

CLASS zcl_invoice IMPLEMENTATION.
  METHOD get_type.
    rv_type = 'Invoice'.
  ENDMETHOD.
ENDCLASS.
```

`INHERITING FROM` gives single inheritance. The subclass inherits all PUBLIC and PROTECTED members; PRIVATE members stay hidden from it.

---

## super-> and REDEFINITION

`REDEFINITION` replaces a parent method. Inside the redefined method, `super->method( )` calls the parent's original implementation — useful when you want to extend rather than fully replace it.

```abap
CLASS zcl_audited_invoice DEFINITION INHERITING FROM zcl_invoice.
  PUBLIC SECTION.
    METHODS: get_type REDEFINITION.
ENDCLASS.

CLASS zcl_audited_invoice IMPLEMENTATION.
  METHOD get_type.
    DATA(lv_parent) = super->get_type( ).   " "Invoice"
    rv_type = |{ lv_parent } (audited)|.    " "Invoice (audited)"
  ENDMETHOD.
ENDCLASS.
```

Rules:
- Only PUBLIC and PROTECTED methods can be redefined; PRIVATE cannot.
- The signature stays identical — you cannot add or change parameters.
- A method marked `FINAL` cannot be redefined.

---

## Constructor chaining

When a subclass object is created, the parent constructor runs before the child constructor. The call `super->constructor( )` is mandatory whenever the parent has a constructor, and if the parent constructor has importing parameters you must pass them.

```abap
CLASS zcl_vehicle DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING iv_wheels TYPE i.
  PROTECTED SECTION.
    DATA: mv_wheels TYPE i.
ENDCLASS.

CLASS zcl_vehicle IMPLEMENTATION.
  METHOD constructor.
    mv_wheels = iv_wheels.
  ENDMETHOD.
ENDCLASS.

CLASS zcl_truck DEFINITION INHERITING FROM zcl_vehicle.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING iv_payload TYPE i.
  PRIVATE SECTION.
    DATA: mv_payload TYPE i.
ENDCLASS.

CLASS zcl_truck IMPLEMENTATION.
  METHOD constructor.
    super->constructor( iv_wheels = 6 ).   " init parent first - MANDATORY
    mv_payload = iv_payload.               " then child-specific init
  ENDMETHOD.
ENDCLASS.
```

Full execution order on `NEW zcl_truck( iv_payload = 1000 )`:
1. `zcl_vehicle=>class_constructor` (if not yet run this session)
2. `zcl_truck=>class_constructor` (if not yet run this session)
3. `zcl_vehicle->constructor` (via `super->`)
4. `zcl_truck->constructor`

**Why parent first?** The child often depends on parent attributes (here `mv_wheels`) being initialized before it runs. Building from the foundation up guarantees the child never reads uninitialized parent state.

---

## Abstract classes & methods

An abstract class cannot be instantiated; an abstract method has no body and must be redefined by every concrete subclass.

```abap
CLASS zcl_shape DEFINITION ABSTRACT.
  PUBLIC SECTION.
    METHODS: area     ABSTRACT RETURNING VALUE(rv_area) TYPE f,   " no body here
             describe RETURNING VALUE(rv_text) TYPE string.       " concrete, shared
ENDCLASS.

CLASS zcl_shape IMPLEMENTATION.
  METHOD describe.
    rv_text = |This shape has area { area( ) }|.   " calls the subclass's area( )
  ENDMETHOD.
  " note: no implementation for area - it is ABSTRACT
ENDCLASS.

CLASS zcl_circle DEFINITION INHERITING FROM zcl_shape FINAL.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING iv_radius TYPE f,
             area REDEFINITION.
  PRIVATE SECTION.
    DATA: mv_radius TYPE f.
ENDCLASS.

CLASS zcl_circle IMPLEMENTATION.
  METHOD constructor.
    super->constructor( ).
    mv_radius = iv_radius.
  ENDMETHOD.
  METHOD area.
    rv_area = mv_radius * mv_radius * '3.14159265'.
  ENDMETHOD.
ENDCLASS.
```

- `NEW zcl_shape( )` → compile error (abstract).
- If `zcl_circle` omitted the `area` redefinition → compile error (abstract method not implemented).
- `describe( )` is concrete and shared, yet it can call the abstract `area( )` because at runtime the object is always a concrete subclass.

**Abstract vs regular superclass:** a regular superclass can be instantiated and its methods need not be redefined, so a forgotten override silently runs a meaningless default. Abstract makes both mistakes compile errors — the compiler is your safety net.

---

## Final classes & methods

```abap
CLASS zcl_circle DEFINITION INHERITING FROM zcl_shape FINAL.
  " FINAL class: nothing can inherit from zcl_circle
ENDCLASS.

CLASS zcl_calculator DEFINITION.
  PUBLIC SECTION.
    METHODS: calc FINAL RETURNING VALUE(rv) TYPE i.   " cannot be redefined
ENDCLASS.
```

Clean ABAP recommends making classes `FINAL` by default and opening them for inheritance only when you intend it. This prevents accidental, unplanned subclassing.

---

## Casting

| Direction | Operator(s) | Risk |
|-----------|-------------|------|
| **Upcast** (subclass → superclass/interface) | `=` (implicit) | Always safe |
| **Downcast** (superclass → subclass) | `CAST` or `?=` | Can fail at runtime |

```abap
" Upcast - safe, implicit. A circle IS a shape.
DATA: lo_shape TYPE REF TO zcl_shape.
lo_shape = NEW zcl_circle( iv_radius = 5 ).

" Downcast - risky, explicit. Must guard against the wrong actual type.
DATA: lo_rect TYPE REF TO zcl_rectangle.
TRY.
    lo_rect = CAST zcl_rectangle( lo_shape ).   " modern syntax
    " classic equivalent: lo_rect ?= lo_shape.
  CATCH cx_sy_move_cast_error.
    " lo_shape was a circle, not a rectangle
ENDTRY.
```

**Memory aid:** moving **up** the hierarchy is always safe (`=`); moving **down** is risky and needs `CAST`/`?=` with error handling.

---

## Polymorphism in practice

```abap
DATA: lt_shapes TYPE STANDARD TABLE OF REF TO zcl_shape.

APPEND NEW zcl_circle( iv_radius = 5 )                TO lt_shapes.
APPEND NEW zcl_rectangle( iv_width = 4 iv_height = 6 ) TO lt_shapes.

LOOP AT lt_shapes INTO DATA(lo_shape).
  WRITE: / lo_shape->area( ).   " each object runs its own area( ) implementation
ENDLOOP.
```

No `CASE` on type — each object knows its own behavior. Adding a new shape subclass requires no change to this loop.

---

## Common SAP use cases

A base class holds shared framework logic; subclasses provide the specifics. Typical examples include output handlers (shared spool/logging in the base, content per document type in subclasses), IDoc processors (base reads the control and data records, subclass maps and posts per message type), and posting frameworks (base orchestrates validate/build/post/commit, subclass builds the specific BAPI call).

---

## Quick self-test

1. What does `super->constructor( )` do, and why is it mandatory?
2. Can you redefine a PRIVATE method? A FINAL method?
3. Difference between an abstract class and a final class?
4. Why does the abstract `zcl_shape` example work even though `area( )` has no body in the base?
5. `=` vs `CAST`/`?=` — which is risky and why?
6. Why does Clean ABAP recommend `FINAL` by default?
