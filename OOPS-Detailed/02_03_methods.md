# Methods: Instance, Static, and Functional

**Learning Objective:** After this topic you can declare and call instance, static, and functional methods correctly, choose the right parameter kinds (`IMPORTING`/`EXPORTING`/`CHANGING`/`RETURNING`/`RAISING`), and write methods that read as expressions.

**Difficulty Level:** Intermediate
**Time to Master:** 75 minutes
**Prerequisites:** `02_01_abap_class_syntax.md`
**Official Sources:**
- ABAP Keyword Documentation → *METHODS*, *CLASS-METHODS*, *Functional Methods*, *Parameter Interface* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Methods* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A pricing routine needs the net amount. Written with an `EXPORTING` parameter you must declare a variable, call the method on its own line, then use the result. Written as a *functional* method returning a value, it reads `total = lo->net_amount( )` — one line, composable.

**What Happens WITHOUT This Concept.** Overusing `EXPORTING`/`CHANGING` produces verbose, statement-heavy code that cannot be composed in expressions and is harder to test. Confusing instance vs static methods causes "method is unknown / instance required" errors.

**Why This Matters in SAP.** Modern ABAP (7.4+) is expression-oriented. Functional methods plug into `COND`, `VALUE`, string templates, and chaining — readable, testable code that Clean ABAP recommends.

---

## 3. Core Concept Explanation

**Definition.**
- An **instance method** (`METHODS`) operates on a specific object and can use its instance state; call it via `ref->method( )`.
- A **static method** (`CLASS-METHODS`) belongs to the class, needs no instance, and can use only static state; call it via `class=>method( )`.
- A **functional method** is any method with a single `RETURNING` parameter (and otherwise only input parameters), usable directly in expressions.

**Parameter kinds:**
- `IMPORTING` — input (read-only inside the method).
- `EXPORTING` — output passed back via a variable.
- `CHANGING` — in-and-out.
- `RETURNING VALUE(r)` — a single return value (makes the method functional).
- `RAISING` — declares the (class-based) exceptions it may throw.

**Key Principles:**
- Prefer `RETURNING` (functional) over `EXPORTING` for a single result.
- Keep parameter lists short; few inputs, one output.
- Use static methods only when there is genuinely no instance state involved.

**How It Works.** The compiler resolves `ref->m( )` against the object's type and `class=>m( )` against the class. A functional method can appear wherever a value of its return type is expected.

**Why It's Designed This Way.** A single return value turns a method into a composable expression; that composability is what makes 7.4+ ABAP concise and testable.

---

## 4. Visual Representation

```
   PARAMETER FLOW                         CALL STYLES

   IMPORTING  ──►┌──────────┐             instance:  lo_obj->net_amount( )
   CHANGING  ◄──►│  METHOD  │──► RETURNING static:    zcl_x=>parse( iv = '...' )
   EXPORTING ◄── │  body    │                          ▲ no instance needed
                 └──────────┘──► RAISING    functional: total = lo->net_amount( )
                                  (exceptions)           ▲ usable inside an expression
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Return a net amount.

**The WRONG Way (Anti-Pattern):**
```abap
METHODS net_amount EXPORTING ev_net TYPE p LENGTH 13 DECIMALS 2.
" caller:
DATA lv_net TYPE p LENGTH 13 DECIMALS 2.
lo->net_amount( IMPORTING ev_net = lv_net ).   " extra variable + extra statement
result = lv_net * '1.19'.
```
**Problems with this code:**
- Forces a temporary variable and a separate statement; not composable.
- Reads as a procedure call, not a value.

**The RIGHT Way (Following Best Practice):**
```abap
METHODS net_amount RETURNING VALUE(rv_net) TYPE p LENGTH 13 DECIMALS 2.
" caller:
result = lo->net_amount( ) * '1.19'.            " one composable expression
```
**Why this is better:**
- Functional: the call *is* a value; no temporary, no extra line.
- Easy to test (`assert_equals( act = lo->net_amount( ) … )`) and easy to chain.

**Step-by-Step Explanation:**
- `RETURNING VALUE(rv_net)` — one output by value; this makes the method functional.
- `lo->net_amount( )` — used directly in arithmetic, like any value.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A converter class offering a static parser (no state) and instance methods that use state, including a functional method and one that raises an exception.

```abap
CLASS zcl_amount DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    " static: pure transformation, no instance state needed
    CLASS-METHODS from_string IMPORTING iv_text TYPE string
                              RETURNING VALUE(ro) TYPE REF TO zcl_amount
                              RAISING   cx_sy_conversion_no_number.
    " instance functional method
    METHODS value RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
    METHODS add IMPORTING io_other TYPE REF TO zcl_amount
                RETURNING VALUE(ro) TYPE REF TO zcl_amount.
  PRIVATE SECTION.
    DATA mv_value TYPE p LENGTH 13 DECIMALS 2.
    METHODS constructor IMPORTING iv_value TYPE p LENGTH 13 DECIMALS 2.
ENDCLASS.

CLASS zcl_amount IMPLEMENTATION.
  METHOD from_string.
    DATA lv_p TYPE p LENGTH 13 DECIMALS 2.
    lv_p = iv_text.                          " may raise on bad input
    ro = NEW zcl_amount( lv_p ).
  ENDMETHOD.
  METHOD constructor.
    mv_value = iv_value.
  ENDMETHOD.
  METHOD value.
    rv = mv_value.
  ENDMETHOD.
  METHOD add.
    ro = NEW zcl_amount( mv_value + io_other->value( ) ).   " returns a new amount
  ENDMETHOD.
ENDCLASS.

" Usage — note the composition enabled by functional methods:
TRY.
    DATA(lo_sum) = zcl_amount=>from_string( '100.00' )->add( zcl_amount=>from_string( '50.00' ) ).
    WRITE lo_sum->value( ).      " 150.00
  CATCH cx_sy_conversion_no_number.
    WRITE 'bad number'.
ENDTRY.
```

**Detailed Walkthrough:**
- **`CLASS-METHODS from_string`** — static because it needs no existing instance; it *creates* one (a tiny factory).
- **`add` returns a new `zcl_amount`** — functional and chainable: `from_string(...)->add(...)`.
- **`RAISING cx_…`** — the parser declares its failure mode; callers must handle it (Topic 2.10).

**How This Works in Practice.** Static for stateless transformations, instance for stateful behaviour, functional for composability — the everyday method-design decision.

**Why This Implementation.** Returning new objects (immutability) plus functional methods yields fluent, testable code, consistent with Clean ABAP.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Calling an instance method statically.**
```abap
zcl_amount=>value( ).     " wrong: value() needs an instance
```
**Why this is wrong:** instance methods require an object; only `CLASS-METHODS` are callable on the class.
**Correct approach:** `lo->value( )`, or declare it `CLASS-METHODS` if it truly needs no state.

**Mistake #2: Many `EXPORTING` parameters instead of a result structure/object.**
```abap
METHODS calc EXPORTING ev_net TYPE p ev_tax TYPE p ev_gross TYPE p ev_currency TYPE waers.
```
**Why this is wrong:** long output lists are hard to read, call, and test; the method probably does too much.
**Correct approach:** return one object/structure, or split responsibilities; prefer a single `RETURNING`.

---

## 8. Comparison With Similar Concepts

**Instance vs Static method:** instance methods can read/write instance state and need an object; static methods cannot touch instance state and need none. Choose static only when no instance data is involved.

**Functional vs procedural method:** a functional method has one `RETURNING` and is usable in expressions; a procedural-style method uses `EXPORTING`/`CHANGING` and reads as a statement. Prefer functional for single results.

**`EXPORTING` vs `RETURNING`:** `RETURNING` gives one composable value; `EXPORTING` is for multiple outputs (which often signals the method should be split). `CHANGING` mutates a passed variable — use rarely.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Constructor expressions (`VALUE`, `COND`, `REDUCE`):** functional methods slot directly into them.
- **String templates:** `|Total: { lo->value( ) }|`.
- **RAP / BAPIs:** wrapping a BAPI typically exposes clean functional methods over its parameter tables.

**SAP-Specific Considerations:** `RETURNING` is passed by value (a copy) — fine for scalars and small structures; for very large internal tables consider the cost, though clarity usually wins. Methods can declare `RAISING` for class-based exceptions and `RAISING RESUMABLE` for resumable ones (Topic 2.10).

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the multi-purpose method controlled by a flag.**
```abap
METHODS process IMPORTING iv_mode TYPE c.   " mode = 'C' calc, 'P' print, 'S' save...
```
**Why this fails:** one method, several responsibilities, selected by a flag — hard to name, test, and reuse.
**Correct approach:** one method per intent (`calculate`, `print`, `save`).

**Common Gotcha:** a method with a single `RETURNING` *and* extra `EXPORTING`/`CHANGING` is **not** functional and cannot be used in expressions. To stay composable, use only `IMPORTING` + one `RETURNING`.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can each method be called as one expression and asserted on its return value? If a method needs setup of outputs and multiple statements to test, reconsider its shape.

**Unit Test Example:**
```abap
CLASS ltcl_amount DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS add_returns_sum FOR TESTING RAISING cx_static_check.
ENDCLASS.

CLASS ltcl_amount IMPLEMENTATION.
  METHOD add_returns_sum.
    DATA(lo) = zcl_amount=>from_string( '100.00' )->add( zcl_amount=>from_string( '50.00' ) ).
    cl_abap_unit_assert=>assert_equals( act = lo->value( ) exp = CONV p( '150.00' ) ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** functional methods let the whole behaviour be exercised and asserted in a single expression — the testability payoff.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Instance method needs an object (`ref->m( )`); static method does not (`class=>m( )`).
- A method with one `RETURNING` (and only inputs) is *functional* and composable — prefer it.
- Parameter kinds: `IMPORTING` (in), `EXPORTING` (out), `CHANGING` (in/out), `RETURNING` (single value), `RAISING` (exceptions).

**When to Apply:** functional + instance by default; static only for stateless operations; `EXPORTING` only for genuine multiple outputs.

**Red Flags:** calling instance methods on the class; long `EXPORTING` lists; a `RETURNING` method that also exports (not functional); flag-driven multi-purpose methods.

---

## 13. Dependency Map

**Depends On:** `02_01_abap_class_syntax.md` — methods are declared in class sections.

**Enables:**
- `02_04_constructors.md` — constructors are special methods.
- `02_06_interfaces_in_abap.md` — interface methods follow the same parameter rules.
- `02_10_class_based_exceptions.md` — `RAISING` connects methods to exceptions.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "METHODS", "CLASS-METHODS", "Functional Methods", "RETURNING", "RAISING".

**Design Patterns & Best Practices:** Clean ABAP → *Methods* (prefer `RETURNING`; few parameters; do one thing) (`github.com/SAP/styleguides`).
