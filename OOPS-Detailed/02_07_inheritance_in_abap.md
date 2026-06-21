# Inheritance in ABAP (REDEFINITION, SUPER->, FINAL, ABSTRACT)

**Learning Objective:** After this topic you can implement ABAP inheritance correctly — redefining methods, calling `super->`, using `ABSTRACT` and `FINAL` on classes and methods, and chaining constructors — and know when to seal a class.

**Difficulty Level:** Intermediate
**Time to Master:** 75–90 minutes
**Prerequisites:** `01_04_inheritance.md`, `02_01_abap_class_syntax.md`
**Official Sources:**
- ABAP Keyword Documentation → *INHERITING FROM*, *REDEFINITION*, *Pseudo Reference super*, *ABSTRACT*, *FINAL* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Inheritance* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** You have a base `report runner` and several specialized runners that differ only in how they fetch data. You want the common run/format/output flow defined once, with each subclass supplying just the fetch step — and you want to forbid anyone instantiating the abstract base directly.

**What Happens WITHOUT This Concept.** Without `ABSTRACT` you can create a meaningless base instance; without `REDEFINITION` you cannot specialize a step; without `FINAL` you cannot stop unwanted subclassing; without `super->` you cannot reuse the parent's logic while extending it.

**Why This Matters in SAP.** SAP's own class library and exception framework lean heavily on these mechanics. Using them correctly is required to extend SAP classes and to build clean hierarchies of your own.

---

## 3. Core Concept Explanation

**Definition.** ABAP inheritance keywords:
- **`INHERITING FROM`** — declares a subclass of one superclass (ABAP is single-inheritance).
- **`REDEFINITION`** — overrides an inherited (non-final) method.
- **`super->method( )`** — calls the superclass's version from inside an override.
- **`ABSTRACT`** — on a class, it cannot be instantiated; on a method, it has no body and *must* be redefined by concrete subclasses.
- **`FINAL`** — on a class, it cannot be subclassed; on a method, it cannot be redefined.

**Key Principles:**
- Make classes `FINAL` by default; open them for inheritance only intentionally.
- Override only what differs; reuse the rest (and `super->` when extending).
- An abstract method defines a required hook; the template flow lives in the base.

**How It Works.** The subclass inherits all non-private members. A redefined method replaces the inherited body for instances of the subclass (dynamic dispatch, Topic 1.5). `super->` reaches exactly one level up.

**Why It's Designed This Way.** These keywords give precise control over *what* may be extended and *how*, so a hierarchy expresses real design intent rather than accidental coupling.

---

## 4. Visual Representation

```
   zcl_runner (ABSTRACT)                     control flow (Template Method)
   ┌──────────────────────────────┐         run():
   │ + run()        (template)     │           data = fetch()   ← abstract hook
   │ # fetch()      (ABSTRACT)     │           format(data)
   │ # format()     (concrete)     │           output()
   └──────────────────────────────┘
            ▲                  ▲
   INHERITING FROM     INHERITING FROM
   ┌───────────────┐   ┌──────────────────┐
   │ zcl_db_runner │   │ zcl_file_runner  │
   │ fetch()=DB    │   │ fetch()=file     │   each supplies ONLY fetch()
   │ (REDEFINITION)│   │ (REDEFINITION)   │
   └───────────────┘   └──────────────────┘
   FINAL: no further subclassing intended
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Redefine a method and reuse the parent via `super->`.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS zcl_db_runner DEFINITION INHERITING FROM zcl_runner.
  PROTECTED SECTION.
    METHODS fetch.            " wrong: re-declares instead of redefining
ENDCLASS.
```
**Problems with this code:**
- A re-declaration does not override the inherited method; ABAP rejects it.
- The intent (override) is not expressed.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS zcl_db_runner DEFINITION INHERITING FROM zcl_runner FINAL.
  PROTECTED SECTION.
    METHODS fetch REDEFINITION.            " correctly overrides the hook
ENDCLASS.
CLASS zcl_db_runner IMPLEMENTATION.
  METHOD fetch.
    SELECT * FROM sflight INTO TABLE @DATA(lt).   " DB-specific fetch
    " ... map lt into the inherited buffer ...
  ENDMETHOD.
ENDCLASS.
```
**Why this is better:**
- `REDEFINITION` correctly replaces the inherited body for this subclass.
- `FINAL` signals the hierarchy ends here.

**Step-by-Step Explanation:**
- `INHERITING FROM zcl_runner` — establishes the subclass.
- `METHODS fetch REDEFINITION` — the addition that turns re-declaration into override.
- `FINAL` — prevents accidental deeper subclassing.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** An abstract base defines the run flow (Template Method); subclasses supply the data step; one override extends the parent's formatting via `super->`.

```abap
CLASS zcl_runner DEFINITION PUBLIC ABSTRACT CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS run FINAL.                         " the fixed flow; cannot be overridden
  PROTECTED SECTION.
    METHODS fetch  ABSTRACT.                    " required hook: subclass must supply
    METHODS format.                             " default, overridable
    DATA mt_buffer TYPE string_table.
ENDCLASS.

CLASS zcl_runner IMPLEMENTATION.
  METHOD run.
    fetch( ).        " polymorphic call to the subclass's fetch
    format( ).
    cl_demo_output=>write( mt_buffer ).
  ENDMETHOD.
  METHOD format.
    " default formatting (uppercase each line)
    mt_buffer = VALUE #( FOR line IN mt_buffer ( to_upper( line ) ) ).
  ENDMETHOD.
ENDCLASS.

CLASS zcl_file_runner DEFINITION INHERITING FROM zcl_runner FINAL.
  PROTECTED SECTION.
    METHODS fetch  REDEFINITION.
    METHODS format REDEFINITION.
ENDCLASS.
CLASS zcl_file_runner IMPLEMENTATION.
  METHOD fetch.
    APPEND 'line from file' TO mt_buffer.       " file-specific fetch
  ENDMETHOD.
  METHOD format.
    super->format( ).                            " reuse parent formatting...
    APPEND '--- end of file ---' TO mt_buffer.   " ...then extend it
  ENDMETHOD.
ENDCLASS.

" DATA(lo) = NEW zcl_runner( ).  ← rejected: class is ABSTRACT
DATA lo TYPE REF TO zcl_runner.
lo = NEW zcl_file_runner( ).
lo->run( ).        " runs the template, dispatching fetch/format to the subclass
```

**Detailed Walkthrough:**
- **`ABSTRACT` class + `ABSTRACT` `fetch`** — the base is incomplete on purpose; only concrete subclasses can be created.
- **`run FINAL`** — the *flow* is fixed; subclasses customize steps, not the orchestration.
- **`super->format( )`** — the override reuses then extends the parent behaviour.

**How This Works in Practice.** This is the **Template Method** pattern: invariant flow in the base, variable steps in subclasses. Adding a new runner means one subclass implementing `fetch`.

**Why This Implementation.** `ABSTRACT` enforces the contract (you must supply `fetch`); `FINAL` on `run` protects the flow; `super->` enables extend-not-replace.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Subclass constructor that skips `super->constructor( )`.**
```abap
CLASS zcl_child DEFINITION INHERITING FROM zcl_parent_with_ctor.
  PUBLIC SECTION. METHODS constructor IMPORTING iv_x TYPE i.
ENDCLASS.
CLASS zcl_child IMPLEMENTATION.
  METHOD constructor.
    mv_x = iv_x.        " wrong: parent constructor never called
  ENDMETHOD.
ENDCLASS.
```
**Why this is wrong:** when the parent has an explicit constructor, the child must call `super->constructor( … )`; otherwise inherited state is uninitialized and the compiler/runtime objects.
**Correct approach:** call `super->constructor( … )` before using inherited members (Topic 2.4).

**Mistake #2: Trying to redefine a `FINAL` method.**
```abap
METHODS run REDEFINITION.   " run was declared FINAL in the base → not allowed
```
**Why this is wrong:** `FINAL` methods are explicitly closed to redefinition.
**Correct approach:** override a non-final hook (`fetch`/`format`), not the sealed flow.

---

## 8. Comparison With Similar Concepts

**`ABSTRACT` class vs Interface (Topic 2.6):** an abstract class can mix concrete shared code with abstract hooks and hold state, but you inherit only one; an interface is pure contract and you can implement many. Use abstract base for a true "is-a" family sharing code; interface for cross-cutting capabilities.

**`REDEFINITION` vs new method:** redefinition overrides an inherited method (same signature); a new method adds behaviour the parent never had. Mismatched signatures cannot be a redefinition.

**`FINAL` vs open:** `FINAL` seals against subclassing/redefinition (the Clean ABAP default); open classes are an explicit invitation to extend. Seal unless extension is intended.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Exception classes (Topic 2.10):** a deep hierarchy under `CX_ROOT`; you subclass and sometimes redefine `get_text`.
- **SAP class library / BAdIs:** extension often means subclassing a provided base and redefining specific methods.
- **RAP:** behaviour implementations use handler classes; inheritance is used judiciously.

**SAP-Specific Considerations:** single inheritance only — model cross-cutting concerns with interfaces, not a second parent. `CREATE PRIVATE`/`PROTECTED` (with `FRIENDS`, Topic 2.11) restricts who may instantiate within a hierarchy. Keep hierarchies shallow to avoid the yo-yo problem (Topic 1.4).

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: a non-final class with everything overridable, "just in case."**
```abap
CLASS zcl_base DEFINITION PUBLIC CREATE PUBLIC.   " not FINAL, no abstract intent
  PUBLIC SECTION. METHODS a. METHODS b. METHODS c.   " all silently overridable
ENDCLASS.
```
**Why this fails:** unconstrained extension makes behaviour unpredictable and refactoring unsafe.
**Correct approach:** `FINAL` by default; expose specific protected hooks when extension is a real requirement.

**Common Gotcha:** redefining a method changes it for the subclass only; the base and sibling classes keep their own versions. Dynamic dispatch picks the *actual* object's version at runtime — make sure your override fulfils the same contract (Liskov, Topic 3.1).

---

## 11. Testing & Validation

**How to Verify Understanding:** Can a subclass be created where the base cannot (because the base is abstract), and does `run( )` call the subclass's `fetch`? If the base instantiates, it isn't abstract.

**Unit Test Example:**
```abap
CLASS ltcl_runner DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS file_runner_overrides_steps FOR TESTING.
ENDCLASS.

CLASS ltcl_runner IMPLEMENTATION.
  METHOD file_runner_overrides_steps.
    DATA lo TYPE REF TO zcl_runner.
    lo = NEW zcl_file_runner( ).      " base is abstract; subclass is concrete
    lo->run( ).
    " 'run' dispatched fetch()+format() to the subclass; here we just assert it executed:
    cl_abap_unit_assert=>assert_bound( lo ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the test creates the concrete subclass through a base-typed reference and runs the template — confirming polymorphic dispatch of the overridden steps.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- `INHERITING FROM` (single inheritance), `REDEFINITION` to override, `super->` to reuse the parent.
- `ABSTRACT` (class = not instantiable; method = must be redefined); `FINAL` (class = not subclassable; method = not redefinable).
- Subclass constructors call `super->constructor( )` first; default classes to `FINAL`.

**When to Apply:** a genuine "is-a" family with shared flow and varying steps (Template Method); seal everything else.

**Red Flags:** re-declaring instead of `REDEFINITION`; missing `super->constructor( )`; instantiable base that should be abstract; non-final classes with no extension intent.

---

## 13. Dependency Map

**Depends On:**
- `01_04_inheritance.md` — the *why* of "is-a" hierarchies.
- `02_01_abap_class_syntax.md`, `02_04_constructors.md` — class structure and constructor chaining.

**Enables:**
- `02_08_casting_and_rtti.md` — up/down casts along the hierarchy.
- `02_10_class_based_exceptions.md` — the exception hierarchy uses these mechanics.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "INHERITING FROM", "REDEFINITION", "Pseudo Reference super", "ABSTRACT", "FINAL".

**Design Patterns & Best Practices:** Clean ABAP → *Inheritance* (prefer composition; make classes `FINAL`; redefine sparingly) (`github.com/SAP/styleguides`). Pattern reference: Template Method (Gang of Four).
