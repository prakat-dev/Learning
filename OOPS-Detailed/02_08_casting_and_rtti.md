# Casting and RTTI (Narrowing / Widening, Run-Time Type Information)

**Learning Objective:** After this topic you can safely perform up-casts (widening) and down-casts (narrowing), handle `CX_SY_MOVE_CAST_ERROR`, use `IS INSTANCE OF`, and query types at runtime with RTTI — while recognizing that frequent casting often signals a missing polymorphic method.

**Difficulty Level:** Advanced
**Time to Master:** 75 minutes
**Prerequisites:** `02_06_interfaces_in_abap.md`, `02_07_inheritance_in_abap.md`
**Official Sources:**
- ABAP Keyword Documentation → *Casting*, *CAST*, *Move Cast*, *IS INSTANCE OF*, *RTTI* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Object Orientation* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A method receives a general `REF TO if_payment` but, for a logging feature, needs to know whether the concrete object also supports an `if_auditable` capability. You must *safely* test and convert the reference — and fail gracefully when it does not.

**What Happens WITHOUT This Concept.** A naive down-cast on the wrong type raises `CX_SY_MOVE_CAST_ERROR` at runtime and dumps the program. Without RTTI you cannot inspect a generic reference (e.g. a `data` of unknown type) at all.

**Why This Matters in SAP.** Generic frameworks (serialization, dynamic table handling, RAP generic helpers) routinely receive references of broad types and must reason about the actual type at runtime. Casting and RTTI are the safe tools for that.

---

## 3. Core Concept Explanation

**Definition.**
- **Up-cast (widening)**: assign a more specific reference to a more general one (subclass → superclass, class → interface). Always safe; implicit.
- **Down-cast (narrowing)**: assign a general reference to a more specific one. Only valid if the object *actually* is that type; checked at runtime. Use `CAST` (expression) or `?=` (Move-Cast operator).
- **RTTI (Run-Time Type Information)**: the `CL_ABAP_*DESCR` class hierarchy that describes types at runtime (`CL_ABAP_TYPEDESCR`, `CL_ABAP_CLASSDESCR`, `CL_ABAP_STRUCTDESCR`, `CL_ABAP_TABLEDESCR`, …).

**Key Principles:**
- Up-casts need no check; down-casts must be guarded.
- Guard a down-cast with `IS INSTANCE OF` or a `TRY … CATCH cx_sy_move_cast_error`.
- Prefer adding a polymorphic method over casting to a concrete type.

**How It Works.** `CAST if_auditable( lo_ref )` attempts the conversion; if `lo_ref`'s object does not implement `if_auditable`, it raises `CX_SY_MOVE_CAST_ERROR`. `IS INSTANCE OF` returns a boolean without raising. RTTI's `…=>describe_by_object_ref( )` / `describe_by_data( )` return descriptor objects you can interrogate.

**Why It's Designed This Way.** Down-casting is inherently risky (you assert a type the compiler can't verify), so ABAP forces a runtime check; RTTI exists for the legitimate cases where type must be discovered dynamically.

---

## 4. Visual Representation

```
   WIDENING (up-cast, safe, implicit)        NARROWING (down-cast, checked)

   lcl_card  ──►  if_payment  ──►  object     if_payment ──?──► if_auditable
   (specific)     (general)                   (general)        (specific)
        always allowed                         allowed ONLY if the actual
                                               object implements if_auditable;
                                               else CX_SY_MOVE_CAST_ERROR

   GUARD:  IF lo_ref IS INSTANCE OF if_auditable.   " test first
             DATA(lo_aud) = CAST if_auditable( lo_ref ).
           ENDIF.
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Convert an interface reference back to a capability it may or may not have.

**The WRONG Way (Anti-Pattern):**
```abap
DATA(lo_aud) = CAST lif_auditable( io_payment ).   " unchecked down-cast
WRITE lo_aud->audit_text( ).                        " dumps if io_payment isn't auditable
```
**Problems with this code:**
- If the object does not implement `lif_auditable`, the program short-dumps with `CX_SY_MOVE_CAST_ERROR`.
- No graceful fallback.

**The RIGHT Way (Following Best Practice):**
```abap
IF io_payment IS INSTANCE OF lif_auditable.          " safe test, no exception
  DATA(lo_aud) = CAST lif_auditable( io_payment ).   " now guaranteed valid
  cl_demo_output=>write( lo_aud->audit_text( ) ).
ELSE.
  cl_demo_output=>write( 'not auditable' ).          " graceful fallback
ENDIF.
```
**Why this is better:**
- The cast happens only when it is known to succeed.
- A non-auditable object is handled, not crashed.

**Step-by-Step Explanation:**
- `IS INSTANCE OF lif_auditable` — boolean test of the *actual* object's type.
- `CAST lif_auditable( … )` — the guarded narrowing conversion.
- `ELSE` — the explicit, safe alternative path.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A generic logger receives any object and uses RTTI to report its class name and attributes — without knowing the type at compile time.

```abap
CLASS zcl_obj_logger DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    CLASS-METHODS describe IMPORTING io_obj TYPE REF TO object
                           RETURNING VALUE(rv_text) TYPE string.
ENDCLASS.

CLASS zcl_obj_logger IMPLEMENTATION.
  METHOD describe.
    " RTTI: get a class descriptor for the actual object behind the generic ref
    DATA(lo_descr) = CAST cl_abap_classdescr(
                       cl_abap_typedescr=>describe_by_object_ref( io_obj ) ).

    rv_text = |Class: { lo_descr->get_relative_name( ) }|.

    " enumerate the object's attributes via RTTI metadata
    LOOP AT lo_descr->attributes INTO DATA(ls_attr).
      rv_text = |{ rv_text }\n  attr: { ls_attr-name } visibility { ls_attr-visibility }|.
    ENDLOOP.
  ENDMETHOD.
ENDCLASS.

" Works for ANY object:
DATA(lv_info) = zcl_obj_logger=>describe( NEW lcl_card( ) ).
cl_demo_output=>write( lv_info ).
```

**Detailed Walkthrough:**
- **`TYPE REF TO object`** — `object` is the root of all classes; this accepts any instance (a widening on the caller's side).
- **`describe_by_object_ref( )`** — RTTI entry point returning a `cl_abap_typedescr`.
- **`CAST cl_abap_classdescr( … )`** — narrows the descriptor to the class-descriptor subtype to read `attributes`.

**How This Works in Practice.** Generic utilities (serializers, dynamic UIs, test helpers) use RTTI to adapt to whatever type arrives at runtime — the legitimate home of casting.

**Why This Implementation.** RTTI is the *intended* mechanism for runtime type inspection; here the down-cast of the descriptor is safe because `describe_by_object_ref` always returns a class descriptor for an object reference.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Down-casting to drive behaviour instead of adding a method.**
```abap
IF io_shape IS INSTANCE OF lcl_circle.
  DATA(lo_c) = CAST lcl_circle( io_shape ).
  rv = lo_c->radius( ) ** 2 * '3.14159'.     " caller knows circle math
ELSEIF io_shape IS INSTANCE OF lcl_square.
  " ...
ENDIF.
```
**Why this is wrong:** this is the `CASE`-on-type anti-pattern (Topic 1.5) wearing a cast; the caller is coupled to every concrete type.
**Correct approach:** give the interface an `area( )` method and call `io_shape->area( )` polymorphically.

**Mistake #2: Using `?=` without expecting the exception.**
```abap
DATA lo_aud TYPE REF TO lif_auditable.
lo_aud ?= io_payment.        " raises CX_SY_MOVE_CAST_ERROR if incompatible
```
**Why this is wrong:** the Move-Cast operator `?=` raises on mismatch; unguarded, it dumps.
**Correct approach:** guard with `IS INSTANCE OF`, or wrap in `TRY … CATCH cx_sy_move_cast_error`.

---

## 8. Comparison With Similar Concepts

**Up-cast vs Down-cast:** widening (specific→general) is always safe and implicit; narrowing (general→specific) asserts a type the compiler can't verify and is runtime-checked. Up-casts enable polymorphism; down-casts partially undo it.

**`CAST` vs `?=`:** both perform a down-cast; `CAST(...)` is an inline expression (modern, composable), `?=` is the classic statement form. Both raise `CX_SY_MOVE_CAST_ERROR` on mismatch.

**`IS INSTANCE OF` vs `TRY/CATCH`:** `IS INSTANCE OF` tests without raising (preferred for a simple branch); a `TRY/CATCH cx_sy_move_cast_error` handles the failure after attempting — use when the cast is expected to usually succeed.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Serialization / `CL_ABAP_*DESCR`:** RTTI underlies generic serialization, RTTS (run-time type *services*) for building types dynamically, and dynamic table handling.
- **Generic frameworks / ALV / RAP helpers:** receive broad references and inspect them via RTTI.
- **Factories (Topic 3.3):** typically return interface references; consumers rarely need to cast back.

**SAP-Specific Considerations:** down-casting has a small runtime cost and, more importantly, a design cost (coupling). Reserve RTTI for genuinely generic code. `describe_by_data( )` inspects data objects; `describe_by_object_ref( )` inspects instances; `describe_by_name( )` inspects named types.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: "cast soup" to reach concrete features.**
```abap
DATA(lo) = CAST lcl_special( CAST if_base( factory->get( ) ) ).   " layered casting
```
**Why this fails:** each cast couples the caller to a concretion and can dump; layered casts signal a broken abstraction.
**Correct approach:** expose the needed behaviour on the interface, or have the factory return the right type.

**Common Gotcha:** `IS INSTANCE OF` checks the *actual* runtime object, not the declared reference type. An initial (unbound) reference is *not* an instance of anything, so the guard correctly skips it — but calling a method on an unbound reference still raises `CX_SY_REF_IS_INITIAL`.

---

## 11. Testing & Validation

**How to Verify Understanding:** Before any down-cast, can you state how it is guarded and what happens if the type doesn't match? An unguarded down-cast is a latent dump.

**Unit Test Example:**
```abap
CLASS ltcl_cast DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS guarded_cast_is_safe FOR TESTING.
ENDCLASS.

CLASS ltcl_cast IMPLEMENTATION.
  METHOD guarded_cast_is_safe.
    DATA lo_pay TYPE REF TO lif_payment.
    lo_pay = NEW lcl_card( ).               " card is NOT auditable

    " The guard must prevent a dump and choose the fallback:
    DATA lv_is_aud TYPE abap_bool.
    lv_is_aud = xsdbool( lo_pay IS INSTANCE OF lif_auditable ).

    cl_abap_unit_assert=>assert_false( lv_is_aud ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the test confirms `IS INSTANCE OF` returns false for a non-implementer, so the guarded cast path is never entered — no runtime dump.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Up-cast (widening) is safe and implicit; down-cast (narrowing) is runtime-checked and must be guarded.
- Guard with `IS INSTANCE OF` (test) or `TRY/CATCH cx_sy_move_cast_error`; `CAST(...)` and `?=` both perform the down-cast.
- RTTI (`CL_ABAP_*DESCR`) inspects types at runtime for genuinely generic code.

**When to Apply:** generic/framework code that must reason about runtime types; almost never to drive ordinary business behaviour (use polymorphism).

**Red Flags:** unguarded down-casts; `IS INSTANCE OF`/`CAST` ladders replacing a polymorphic method; layered "cast soup."

---

## 13. Dependency Map

**Depends On:**
- `02_06_interfaces_in_abap.md` — casting between class and interface references.
- `02_07_inheritance_in_abap.md` — up/down the class hierarchy.

**Enables:**
- `03_03_factory_pattern.md` — factories return references that consumers use without casting.
- Generic utility design across the system.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "Casting", "CAST", "Move Cast" (`?=`), "IS INSTANCE OF", "RTTI", "CL_ABAP_TYPEDESCR".

**Design Patterns & Best Practices:** Clean ABAP → prefer polymorphism to casting / type checks (`github.com/SAP/styleguides`).
