# Constructors: Instance and Class Constructor

**Learning Objective:** After this topic you can use the instance constructor to guarantee a valid object at creation, use the class (static) constructor to initialize static state once, and chain constructors correctly through inheritance.

**Difficulty Level:** Intermediate
**Time to Master:** 60–75 minutes
**Prerequisites:** `02_01_abap_class_syntax.md`, `02_02_attributes_instance_static.md`
**Official Sources:**
- ABAP Keyword Documentation → *CONSTRUCTOR*, *CLASS_CONSTRUCTOR* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Constructors* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** An order object must always have a customer and at least one item. If callers create an empty object and then *maybe* set those fields, half the system ends up with half-built orders that blow up later, far from where the mistake was made.

**What Happens WITHOUT This Concept.** Without a constructor that demands mandatory data, objects can exist in an invalid, partially-initialized state. Bugs surface late and far from their cause. Static caches initialize lazily and inconsistently.

**Why This Matters in SAP.** A constructor is the one guaranteed entry point at birth. Using it to enforce "a valid object or none at all" prevents a whole class of late-failing bugs in long-running business processes.

---

## 3. Core Concept Explanation

**Definition.**
- The **instance constructor** (`METHODS constructor`) runs automatically once, when an object is created (`NEW`/`CREATE OBJECT`). It sets up instance state.
- The **class (static) constructor** (`CLASS-METHODS class_constructor`) runs automatically once per session, *before the class is first used* (first instance creation, first static access, etc.). It sets up static state.

**Key Principles:**
- Use the instance constructor to require mandatory data — make invalid objects impossible.
- Keep constructors light: assign and validate; do not do heavy work (DB reads, remote calls) there.
- A subclass constructor must call `super->constructor( )` before using inherited state.
- The class constructor takes no parameters and you never call it explicitly.

**How It Works.** `NEW zcl_x( … )` allocates the object and immediately runs `constructor` with the supplied arguments. The first time the class is touched in a session, the runtime runs `class_constructor` once.

**Why It's Designed This Way.** Guaranteeing the constructor runs at birth gives the object exactly one chance to establish its invariants — the cornerstone of encapsulation (Topic 1.3).

---

## 4. Visual Representation

```
   FIRST use of the class in the session
   ──────────────────────────────────────
   class_constructor( )   ← runs ONCE, no parameters, sets static state
            │
            ▼
   NEW zcl_x( a = 1 b = 2 )   ← each time you create an object
   ──────────────────────────────────────
   constructor( a b )      ← runs per object, sets instance state, validates

   Inheritance:  child constructor ── super->constructor( ) ──► parent constructor
                 (parent part initialized first)
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** An order that cannot exist without a customer.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    METHODS set_customer IMPORTING iv_cust TYPE kunnr.   " optional setter
  PRIVATE SECTION.
    DATA mv_cust TYPE kunnr.
ENDCLASS.
" caller may simply forget:
DATA(lo) = NEW lcl_order( ).      " valid object with NO customer → time bomb
```
**Problems with this code:**
- An order with no customer can exist and propagate.
- The rule "must have a customer" is not enforced anywhere.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    METHODS constructor IMPORTING iv_cust TYPE kunnr.    " customer is MANDATORY
  PRIVATE SECTION.
    DATA mv_cust TYPE kunnr.
ENDCLASS.
CLASS lcl_order IMPLEMENTATION.
  METHOD constructor.
    IF iv_cust IS INITIAL.
      RAISE EXCEPTION TYPE cx_sy_create_object_error.   " no valid customer → no object
    ENDIF.
    mv_cust = iv_cust.
  ENDMETHOD.
ENDCLASS.
" caller cannot skip it:
DATA(lo) = NEW lcl_order( iv_cust = '0000001000' ).
```
**Why this is better:**
- You cannot create an order without a customer; the invalid state is impossible.
- Validation lives at the single point of creation.

**Step-by-Step Explanation:**
- `METHODS constructor IMPORTING iv_cust` — the customer must be supplied to `NEW`.
- The `IF … RAISE` guard rejects invalid input at birth.
- After construction, the object is guaranteed valid.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A class that lazily loads shared configuration once (class constructor) and validates per-object data (instance constructor), with a subclass that chains correctly.

```abap
CLASS zcl_document DEFINITION PUBLIC CREATE PUBLIC.
  PUBLIC SECTION.
    CLASS-DATA gv_default_lang TYPE spras READ-ONLY.
    METHODS constructor IMPORTING iv_id TYPE string.
  PROTECTED SECTION.
    DATA mv_id TYPE string.
ENDCLASS.

CLASS zcl_document IMPLEMENTATION.
  METHOD class_constructor.            " runs ONCE before first use
    gv_default_lang = sy-langu.        " initialize static state
  ENDMETHOD.
  METHOD constructor.                  " runs per object
    IF iv_id IS INITIAL.
      RAISE EXCEPTION TYPE cx_sy_create_object_error.
    ENDIF.
    mv_id = iv_id.
  ENDMETHOD.
ENDCLASS.

CLASS zcl_invoice DEFINITION INHERITING FROM zcl_document.
  PUBLIC SECTION.
    METHODS constructor IMPORTING iv_id  TYPE string
                                  iv_due TYPE d.
  PRIVATE SECTION.
    DATA mv_due TYPE d.
ENDCLASS.

CLASS zcl_invoice IMPLEMENTATION.
  METHOD constructor.
    super->constructor( iv_id ).       " MUST initialize the parent part first
    mv_due = iv_due.                   " then the child's own state
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **`class_constructor`** initializes `gv_default_lang` exactly once; you never call it.
- **`super->constructor( iv_id )`** runs the parent's validation and setup before the child touches inherited state — required when the parent has a non-default constructor.
- **Order matters:** parent first, then child, so inherited state is ready when the child uses it.

**How This Works in Practice.** The first `zcl_document`/`zcl_invoice` touched in the session triggers `class_constructor`; every `NEW` triggers the instance constructor chain parent→child.

**Why This Implementation.** Splitting one-time static setup (class constructor) from per-object setup (instance constructor) keeps each concern in its correct lifecycle slot.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Forgetting `super->constructor( )` in a subclass.**
```abap
METHOD constructor.
  mv_due = iv_due.        " wrong: parent constructor not called → "instance constructor of superclass"
ENDMETHOD.
```
**Why this is wrong:** when the parent has an explicit constructor, the child *must* call it (as the first relevant action); otherwise the parent part is uninitialized and the compiler/runtime objects.
**Correct approach:** `super->constructor( … ).` before using inherited members.

**Mistake #2: Heavy work in the constructor.**
```abap
METHOD constructor.
  SELECT * FROM huge_table INTO TABLE mt_data.   " slow, untestable, surprising
ENDMETHOD.
```
**Why this is wrong:** constructors that hit the database or remote systems are slow, hard to test, and create objects with hidden side effects.
**Correct approach:** keep construction cheap; inject data or load it explicitly via a method/factory (Topics 3.3, 3.5).

---

## 8. Comparison With Similar Concepts

**Instance constructor vs Class constructor:** instance runs per object with parameters to set up instance state; class runs once per session with no parameters to set up static state. Different lifecycles, different jobs.

**Constructor vs Factory method (Topic 3.3):** a constructor always returns an instance of *this* class; a factory method can choose a subtype, return a cached instance, or fail cleanly. Use a factory when creation needs logic beyond "make one of me."

**Constructor vs setters:** a constructor enforces mandatory data up front; scattered setters allow half-built objects. Prefer the constructor for required fields.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Singletons (Topic 3.2):** `class_constructor` or a static factory creates the single instance.
- **Dependency injection (Topic 3.5):** dependencies are passed *into* the constructor.
- **RAP:** managed objects are created by the framework; your validations run at defined points rather than in a constructor.

**SAP-Specific Considerations:** the class constructor must not raise checked exceptions and takes no parameters; keep it minimal because failures there are hard to handle. Avoid DB/RFC calls in constructors for testability. `CREATE PRIVATE` + a factory is the idiomatic way to control instantiation (Topic 2.11, 3.3).

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the do-nothing constructor plus mandatory setters.**
```abap
DATA(lo) = NEW lcl_order( ).
lo->set_customer( '1000' ).   " everyone must remember this, every time
lo->set_items( lt ).
```
**Why this fails:** the object is invalid between `NEW` and the setters; someone will forget a setter.
**Correct approach:** require the mandatory data in the constructor.

**Common Gotcha:** the class constructor runs the *first time the class is touched in a session* — which can be earlier or later than you expect, and only once. Do not rely on it re-running; reset static state explicitly in tests (Topic 2.2).

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you create an *invalid* object? If yes, the constructor isn't enforcing the invariants it should.

**Unit Test Example:**
```abap
CLASS ltcl_order DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS rejects_empty_customer FOR TESTING.
ENDCLASS.

CLASS ltcl_order IMPLEMENTATION.
  METHOD rejects_empty_customer.
    TRY.
        DATA(lo) = NEW lcl_order( iv_cust = space ).
        cl_abap_unit_assert=>fail( 'empty customer should be rejected' ).
      CATCH cx_sy_create_object_error.
        " expected: construction refused → invariant protected
    ENDTRY.
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the test proves an order cannot be born invalid — the constructor enforces the rule.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Instance constructor: per object, with parameters, enforces a valid object at birth.
- Class constructor: once per session, no parameters, initializes static state.
- Subclass constructors must call `super->constructor( )` first; keep all constructors cheap.

**When to Apply:** put mandatory data and validation in the instance constructor; put one-time static setup in the class constructor.

**Red Flags:** missing `super->constructor( )`; DB/RFC work in a constructor; do-nothing constructor with required setters.

---

## 13. Dependency Map

**Depends On:**
- `02_01_abap_class_syntax.md` — constructors are methods of the class.
- `02_02_attributes_instance_static.md` — the class constructor initializes static attributes.

**Enables:**
- `02_07_inheritance_in_abap.md` — constructor chaining mechanics.
- `03_02_singleton_pattern.md`, `03_05_dependency_injection.md` — both center on controlled construction.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "CONSTRUCTOR", "CLASS_CONSTRUCTOR", "Constructor - Instance Constructor", "Static Constructor".

**Design Patterns & Best Practices:** Clean ABAP → *Constructors* (prefer `NEW` to `CREATE OBJECT`; keep constructors simple; prefer multiple static creation methods to optional parameters) (`github.com/SAP/styleguides`).
