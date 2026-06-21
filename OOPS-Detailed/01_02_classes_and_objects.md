# Classes and Objects

**Learning Objective:** After this topic you can clearly distinguish a **class** (a blueprint/type) from an **object** (a runtime instance), and explain why one class can produce many independent objects.

**Difficulty Level:** Foundational
**Time to Master:** 60 minutes
**Prerequisites:** `01_01_why_oop_principles.md`
**Official Sources:**
- ABAP Keyword Documentation → *CLASS*, *CREATE OBJECT*, *NEW* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Classes* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A warehouse program tracks 10,000 material movements at once. Each movement has its own material, quantity, and status, and each must validate and post itself independently. You need 10,000 *independent* bundles of "data + behaviour," not 10,000 copies of code.

**What Happens WITHOUT This Concept.** Without the class/object distinction you either duplicate code per movement or fall back to one shared global structure and a loop — re-introducing the shared-state problems from Topic 1.1. There is no clean way to say "here is one *kind* of thing; now make many of them."

**Why This Matters in SAP.** Nearly every SAP business entity — a document, an item, a partner — is naturally "one type, many instances." The class/object distinction is the vocabulary for that, and it underpins all ABAP OO, RAP business objects included.

---

## 3. Core Concept Explanation

**Definition.**
- A **class** is a *blueprint*: it declares what attributes (data) and methods (behaviour) every object of that kind will have. A class is a type; it occupies no business data itself.
- An **object** (a.k.a. **instance**) is a concrete thing built from that blueprint at runtime, with its *own* copy of the attributes.

**Key Principles:**
- One class → many objects, each with independent state.
- You interact with an object through a **reference variable** that points to it.
- Creating an object is **instantiation**.

**How It Works.** `CLASS lcl_x DEFINITION` declares the blueprint. At runtime, `NEW lcl_x( )` allocates a fresh object on the heap and returns a reference. Calling `ref->method( )` runs that object's behaviour against *its* attributes.

**Why It's Designed This Way.** Separating "the kind of thing" (compile-time type) from "an actual thing" (runtime instance) is what lets you write the logic once and run it over thousands of independent states safely.

---

## 4. Visual Representation

```
        CLASS  (blueprint, written once)
        ┌──────────────────────────────┐
        │ lcl_movement                 │
        │  attributes: material, qty   │
        │  methods   : validate(), post() │
        └──────────────────────────────┘
                     │  NEW lcl_movement( ... )
        ┌────────────┼─────────────┬───────────────┐
        ▼            ▼             ▼
   ┌─────────┐  ┌─────────┐   ┌─────────┐
   │ object1 │  │ object2 │   │ object3 │   ... many instances
   │ M-100,5 │  │ M-200,2 │   │ M-100,9 │   each its OWN data
   └─────────┘  └─────────┘   └─────────┘
        ▲
   lo_ref ──────┘  a reference variable points to one object
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Model a single material movement and create instances of it.

**The WRONG Way (Anti-Pattern):**
```abap
" Trying to represent 'many movements' with one global structure + flags
DATA: gs_movement TYPE ty_movement,
      gv_busy      TYPE abap_bool.
" To handle a second movement you must overwrite gs_movement → you lost the first.
```
**Problems with this code:**
- Only one movement can exist at a time.
- "Behaviour" must be free routines reading the single global structure.
- No way to hold several movements simultaneously.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_movement DEFINITION.
  PUBLIC SECTION.
    METHODS constructor IMPORTING iv_material TYPE matnr
                                  iv_qty      TYPE menge_d.
    METHODS describe RETURNING VALUE(rv_text) TYPE string.
  PRIVATE SECTION.
    DATA mv_material TYPE matnr.    " each object's OWN copy
    DATA mv_qty      TYPE menge_d.
ENDCLASS.

CLASS lcl_movement IMPLEMENTATION.
  METHOD constructor.
    mv_material = iv_material.
    mv_qty      = iv_qty.
  ENDMETHOD.
  METHOD describe.
    rv_text = |{ mv_material } x { mv_qty }|.
  ENDMETHOD.
ENDCLASS.

" One CLASS, several independent OBJECTS:
DATA(lo_a) = NEW lcl_movement( iv_material = 'M-100' iv_qty = 5 ).
DATA(lo_b) = NEW lcl_movement( iv_material = 'M-200' iv_qty = 2 ).
WRITE: / lo_a->describe( ),   " M-100 x 5
       / lo_b->describe( ).   " M-200 x 2  (lo_a is untouched)
```
**Why this is better:**
- Two live objects, two independent states, zero interference.
- The blueprint (`lcl_movement`) is written once; instances are cheap.

**Step-by-Step Explanation:**
- `CLASS … DEFINITION / IMPLEMENTATION` — declares the blueprint, then its behaviour.
- `NEW lcl_movement( … )` — allocates one object and returns a reference; `DATA(lo_a)` infers the reference type.
- `lo_a->describe( )` — runs `describe` against `lo_a`'s own `mv_material`/`mv_qty`.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** Build a collection of movements read from a table and post each independently.

```abap
TYPES: tt_movement_ref TYPE STANDARD TABLE OF REF TO lcl_movement WITH EMPTY KEY.

DATA(lt_objects) = VALUE tt_movement_ref( ).

" Read raw rows, turn each into its own object:
SELECT material, qty FROM zmovement_stage
  INTO TABLE @DATA(lt_rows).

LOOP AT lt_rows INTO DATA(ls_row).
  APPEND NEW lcl_movement( iv_material = ls_row-material
                           iv_qty      = ls_row-qty ) TO lt_objects.
ENDLOOP.

" Each object describes (or, in a fuller class, posts) itself:
LOOP AT lt_objects INTO DATA(lo_mov).
  WRITE: / lo_mov->describe( ).
ENDLOOP.
```

**Detailed Walkthrough:**
- **`REF TO lcl_movement`** — a table of *references*; each entry points to a distinct object.
- **The first loop** turns each database row into an independent object (the "one type, many instances" pattern).
- **The second loop** treats them uniformly while each keeps its own data.

**How This Works in Practice.** You converted flat rows into a set of self-contained behavioural objects. Adding logic (validation, posting) means adding a method to the *one* class; every instance gains it.

**Why This Implementation.** Holding objects in an internal table is the idiomatic ABAP way to manage a variable-size set of instances.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Using an uninitialized reference.**
```abap
DATA lo_mov TYPE REF TO lcl_movement.   " declared but points to nothing
WRITE lo_mov->describe( ).               " runtime error: CX_SY_REF_IS_INITIAL
```
**Why this is wrong:** a reference variable is not an object; until you `NEW` something, it is initial (null-like).
**Correct approach:** `DATA(lo_mov) = NEW lcl_movement( … ).` before calling methods, or check `IF lo_mov IS BOUND.`

**Mistake #2: Confusing the class with the object.**
```abap
lcl_movement->describe( ).   " wrong: describe is an INSTANCE method, not static
```
**Why this is wrong:** instance methods need an instance. `lcl_movement` is the type, not a thing.
**Correct approach:** call on an instance: `lo_a->describe( )`. (Type-level behaviour uses static methods — Topic 2.3.)

---

## 8. Comparison With Similar Concepts

**Class vs Object:** a class is the cookie cutter; an object is a cookie. There is one cutter and many cookies, each able to have different decorations (state).

**Object vs Structure (`TYPES … BEGIN OF`):** a structure holds data only; an object holds data *and* the behaviour that operates on it, with the data hidden. A structure is passive; an object is active.

**Reference variable vs Object:** the reference is the remote control; the object is the TV. Several remotes can point at one TV; clearing a remote does not destroy the TV (the garbage collector reclaims an object once nothing references it).

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Database rows:** the row-to-object mapping in §6 is the everyday bridge between persisted data and behaviour.
- **BAPI/RFC:** a class instance often represents one business document being processed via BAPIs.
- **RAP / Fiori:** RAP business objects are class-backed; "one entity type, many instances" is the core model.

**SAP-Specific Considerations:** each object consumes session memory; for very large sets, consider whether you truly need one object per row or can process in batches. Unreferenced objects are reclaimed automatically — you do not `FREE` objects manually as a rule, but you do clear references you no longer need.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: a class that is never instantiated and holds only static everything**, used purely as a namespace for procedures.
```abap
CLASS lcl_utils DEFINITION.
  PUBLIC SECTION.
    CLASS-METHODS do_a. CLASS-METHODS do_b.   " a function group with new syntax
ENDCLASS.
```
**Why this fails:** if nothing has state, you may not need objects at all — be honest about whether OO is buying you anything (revisit Topic 1.1).
**Correct approach:** use instances when state varies per thing; use static helpers sparingly and deliberately.

**Common Gotcha:** copying a reference does **not** copy the object. `lo_b = lo_a` makes both point to the *same* object; changing one changes "both," because there is only one.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you create two objects of one class and show they hold independent state? If they share state, you copied a reference instead of instantiating.

**Unit Test Example:**
```abap
CLASS ltcl_movement DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS instances_are_independent FOR TESTING.
ENDCLASS.

CLASS ltcl_movement IMPLEMENTATION.
  METHOD instances_are_independent.
    DATA(lo_a) = NEW lcl_movement( iv_material = 'M-100' iv_qty = 5 ).
    DATA(lo_b) = NEW lcl_movement( iv_material = 'M-200' iv_qty = 2 ).

    cl_abap_unit_assert=>assert_equals( act = lo_a->describe( ) exp = 'M-100 x 5' ).
    cl_abap_unit_assert=>assert_equals( act = lo_b->describe( ) exp = 'M-200 x 2' ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** two instances, two distinct descriptions — proof that one class yields independent objects.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Class = blueprint/type; object = a runtime instance with its own state.
- `NEW lcl_x( )` creates an object and returns a reference; you act through the reference.
- One class produces many independent objects — that is the whole point.

**When to Apply:** whenever you have "one kind of thing, many of them, each with its own data."

**Red Flags:** calling instance methods on the class name; assuming `lo_b = lo_a` copies the object; using a reference before `NEW`.

---

## 13. Dependency Map

**Depends On:** `01_01_why_oop_principles.md` — establishes *why* bundled, instance-owned state matters.

**Enables:**
- `01_03_encapsulation.md` — controlling access to the instance state introduced here.
- `01_06_abstraction_and_interfaces.md` — typing references by interface rather than class.
- `02_01_abap_class_syntax.md` — the full local/global class syntax.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "CLASS" (declaration), "NEW" (instance operator), and "CREATE OBJECT" (classic instantiation), plus "Reference Variables".

**Design Patterns & Best Practices:** Clean ABAP → *Classes* (prefer objects to static-only classes when state is involved); `github.com/SAP/styleguides`.
