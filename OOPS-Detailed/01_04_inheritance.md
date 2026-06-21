# Inheritance

**Learning Objective:** After this topic you can use an "is-a" hierarchy to reuse and specialize behaviour, and — just as important — recognize when inheritance is the *wrong* tool and composition is better.

**Difficulty Level:** Foundational
**Time to Master:** 75–90 minutes
**Prerequisites:** `01_02_classes_and_objects.md`, `01_03_encapsulation.md`
**Official Sources:**
- ABAP Keyword Documentation → *INHERITING FROM*, *REDEFINITION*, *SUPER->*, *FINAL*, *ABSTRACT* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Inheritance* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** You price standard products, but service products add a labour surcharge, and consignment products defer revenue. Each shares 90% of the pricing logic and differs in one step. Copy-pasting the whole pricing class three times means three places to fix every future bug.

**What Happens WITHOUT This Concept.** Either you duplicate near-identical classes (drift and divergent bugs), or you cram every variant into one class with a tangle of `IF type = …` branches that grows every time a new product type appears.

**Why This Matters in SAP.** Variation-on-a-theme is everywhere in ERP: document types, partner roles, tax categories. Inheritance lets the common part live once while each variant overrides only what differs.

---

## 3. Core Concept Explanation

**Definition.** **Inheritance** lets a **subclass** (child) derive from a **superclass** (parent), automatically gaining the parent's attributes and methods, and optionally **redefining** (overriding) specific methods or adding new ones. The relationship reads as **"is-a"**: a service product *is a* product.

**Key Principles:**
- A subclass reuses the parent and *specializes* it; it does not copy it.
- Override only what genuinely differs (`REDEFINITION`); inherit the rest.
- `PROTECTED` members are visible to subclasses but not to outside callers.
- Favor inheritance only for true "is-a" relationships; otherwise prefer **composition** ("has-a").

**How It Works.** `CLASS lcl_service DEFINITION INHERITING FROM lcl_product` makes `lcl_service` a `lcl_product`. It can `REDEFINITION` a method to change its behaviour and call `super->method( )` to reuse the parent's version inside the override.

**Why It's Designed This Way.** Shared logic in the parent + targeted overrides in children = no duplication and a single home for the common rule. New variants are added, not retrofitted into a growing `IF` ladder.

---

## 4. Visual Representation

```
                 lcl_product   (parent: common pricing)
                 ┌───────────────────────────────┐
                 │ # base_amount()  (PROTECTED)   │
                 │ + price()        (template)    │
                 └───────────────────────────────┘
                     ▲                 ▲
        INHERITING   │                 │   INHERITING
        FROM         │                 │   FROM
   ┌─────────────────┴───┐     ┌───────┴───────────────┐
   │ lcl_service_product │     │ lcl_consignment_prod  │
   │  REDEFINES price()  │     │  REDEFINES price()    │
   │  adds labour surch. │     │  defers revenue       │
   └─────────────────────┘     └───────────────────────┘
        "is-a product"              "is-a product"
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** A base product computes a price; a service product adds a surcharge but reuses the base calculation.

**The WRONG Way (Anti-Pattern):**
```abap
" Duplicated class per variant — common logic copied, will drift
CLASS lcl_service_product DEFINITION.
  PUBLIC SECTION.
    METHODS price RETURNING VALUE(rv) TYPE p_amount.
  PRIVATE SECTION.
    DATA mv_base TYPE p_amount.
ENDCLASS.
CLASS lcl_service_product IMPLEMENTATION.
  METHOD price.
    rv = mv_base * '1.19'.        " tax logic copied from lcl_product...
    rv = rv + 50.                 " ...plus the surcharge
  ENDMETHOD.
ENDCLASS.
" If the tax rule changes, you must find and fix EVERY copied class.
```
**Problems with this code:**
- The shared tax rule is duplicated; copies drift apart over time.
- Each new product type means another full copy.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_product DEFINITION.
  PUBLIC SECTION.
    METHODS constructor IMPORTING iv_base TYPE p_amount.
    METHODS price RETURNING VALUE(rv_price) TYPE p_amount.
  PROTECTED SECTION.
    DATA mv_base TYPE p_amount.                 " visible to subclasses
    METHODS base_amount RETURNING VALUE(rv) TYPE p_amount.
ENDCLASS.

CLASS lcl_product IMPLEMENTATION.
  METHOD constructor.
    mv_base = iv_base.
  ENDMETHOD.
  METHOD base_amount.
    rv = mv_base * '1.19'.                       " the COMMON rule, defined ONCE
  ENDMETHOD.
  METHOD price.
    rv_price = base_amount( ).
  ENDMETHOD.
ENDCLASS.

CLASS lcl_service_product DEFINITION INHERITING FROM lcl_product.
  PUBLIC SECTION.
    METHODS price REDEFINITION.                  " specialize only price
ENDCLASS.

CLASS lcl_service_product IMPLEMENTATION.
  METHOD price.
    rv_price = base_amount( ) + 50.              " reuse parent rule, add surcharge
  ENDMETHOD.
ENDCLASS.
```
**Why this is better:**
- The tax rule (`base_amount`) lives once; a change there fixes every product type.
- `lcl_service_product` writes only the one line that differs.

**Step-by-Step Explanation:**
- `INHERITING FROM lcl_product` — establishes the "is-a" link; the child gains the parent's members.
- `PROTECTED … base_amount` — shared logic visible to children, hidden from outside callers.
- `price REDEFINITION` — the child replaces only `price`; everything else is inherited.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** Multiple product types must price themselves, reusing the common base and calling the parent implementation explicitly where useful (`super->`).

```abap
CLASS lcl_consignment_product DEFINITION INHERITING FROM lcl_product.
  PUBLIC SECTION.
    METHODS price REDEFINITION.
ENDCLASS.

CLASS lcl_consignment_product IMPLEMENTATION.
  METHOD price.
    " reuse the FULL parent price, then apply a consignment adjustment:
    rv_price = super->price( ) * '0.90'.   " 10% deferred-revenue discount on the parent result
  ENDMETHOD.
ENDCLASS.

" Common driver treats them by the parent type (see Polymorphism, Topic 1.5):
DATA lt_products TYPE STANDARD TABLE OF REF TO lcl_product WITH EMPTY KEY.
APPEND NEW lcl_product( 100 )              TO lt_products.
APPEND NEW lcl_service_product( 100 )      TO lt_products.
APPEND NEW lcl_consignment_product( 100 )  TO lt_products.

LOOP AT lt_products INTO DATA(lo_prod).
  WRITE: / lo_prod->price( ).    " each runs ITS OWN price(), common base reused
ENDLOOP.
```

**Detailed Walkthrough:**
- **`super->price( )`** — calls the parent's `price` from inside the child's override, so the child *extends* rather than replaces the base behaviour.
- **A table typed `REF TO lcl_product`** — holds any subclass instance, because each *is a* product.
- **The loop** runs each object's own `price`; the shared rule is never duplicated.

**How This Works in Practice.** Adding a fourth product type means one new subclass that overrides only `price`. No existing code changes — the closest thing to the Open–Closed principle (Topic 3.1) at this level.

**Why This Implementation.** `super->` is used when the variant *builds on* the parent result; a plain override (Example 1) is used when the variant *replaces* a step.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Inheriting for code reuse when the relationship is not "is-a".**
```abap
CLASS lcl_invoice DEFINITION INHERITING FROM lcl_string_utils.   " an invoice is NOT a string utility
```
**Why this is wrong:** inheritance ties the child to the parent's whole interface and lifecycle; a non-"is-a" link creates a brittle, confusing hierarchy.
**Correct approach:** use **composition** — give `lcl_invoice` a `lcl_string_utils` *member* and call it.

**Mistake #2: Forgetting `REDEFINITION` in the definition.**
```abap
CLASS lcl_service_product DEFINITION INHERITING FROM lcl_product.
  PUBLIC SECTION.
    METHODS price.                 " wrong: re-declares instead of redefining → syntax error
ENDCLASS.
```
**Why this is wrong:** to override an inherited method you must declare it with the `REDEFINITION` addition, not re-declare its signature.
**Correct approach:** `METHODS price REDEFINITION.`

---

## 8. Comparison With Similar Concepts

**Inheritance ("is-a") vs Composition ("has-a"):** inheritance says a service product *is a* product; composition says an invoice *has a* formatter. Composition is more flexible and the modern default — Clean ABAP advises preferring composition and using inheritance sparingly. Use inheritance only for genuine specialization.

**Inheritance vs Interfaces (Topic 1.6):** inheritance shares *implementation* down a single parent chain (ABAP has single inheritance); an interface shares only a *contract* and a class can implement many. When you want polymorphism without forced shared code, prefer an interface.

**Override (`REDEFINITION`) vs `super->`:** a plain override replaces the parent method; `super->method( )` lets the override reuse the parent's logic and add to it.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Exception classes:** ABAP's exception framework is a deep inheritance tree rooted at `CX_ROOT` — you subclass `CX_STATIC_CHECK` etc. (Topic 2.10).
- **SAP class library:** many SAP classes are designed for extension; you subclass and redefine specific methods.
- **BAdI / enhancement:** classic extensibility often relies on redefining methods of a provided base class.

**SAP-Specific Considerations:** ABAP supports **single inheritance only** — a class has at most one parent. Mark classes/methods `FINAL` to prevent unintended subclassing/overriding. Deep hierarchies hurt readability; keep them shallow.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: deep inheritance for incidental sharing** ("yo-yo problem"), where understanding one method means jumping up and down 5 levels.
```abap
" lcl_a -> lcl_b -> lcl_c -> lcl_d -> lcl_e ... where is price() actually decided?
```
**Why this fails:** behaviour is scattered across levels; readers cannot follow the flow.
**Correct approach:** flatten to one or two levels; prefer composition/interfaces for the rest.

**Common Gotcha:** the subclass constructor must call `super->constructor( )` (when the parent has a non-trivial constructor) before using inherited state — otherwise the parent part is uninitialized.

---

## 11. Testing & Validation

**How to Verify Understanding:** Does the subclass change *only* what differs, with the common rule defined once in the parent? If the child re-implements shared logic, inheritance isn't buying you reuse.

**Unit Test Example:**
```abap
CLASS ltcl_pricing DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS subclass_reuses_base FOR TESTING.
ENDCLASS.

CLASS ltcl_pricing IMPLEMENTATION.
  METHOD subclass_reuses_base.
    DATA(lo_base)    = NEW lcl_product( 100 ).               " 100 * 1.19 = 119
    DATA(lo_service) = NEW lcl_service_product( 100 ).        " 119 + 50  = 169

    cl_abap_unit_assert=>assert_equals( act = lo_base->price( )    exp = CONV p_amount( 119 ) ).
    cl_abap_unit_assert=>assert_equals( act = lo_service->price( ) exp = CONV p_amount( 169 ) ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the service price equals the *base* price plus the surcharge — proving the subclass reused the parent's tax rule rather than copying it.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Inheritance = reuse + specialize via an "is-a" hierarchy; override only what differs.
- `PROTECTED` shares with subclasses; `super->` reuses parent logic inside an override.
- ABAP is single-inheritance; prefer composition/interfaces unless the relationship is truly "is-a".

**When to Apply:** genuine specialization of a common base where most behaviour is shared.

**Red Flags:** inheriting just to reuse a utility; deep hierarchies; subclasses re-implementing shared logic; missing `super->constructor( )`.

---

## 13. Dependency Map

**Depends On:**
- `01_02_classes_and_objects.md` — needs the class/instance model.
- `01_03_encapsulation.md` — `PROTECTED` only makes sense after visibility is understood.

**Enables:**
- `01_05_polymorphism.md` — overriding is one of the two routes to polymorphism.
- `02_07_inheritance_in_abap.md` — full ABAP mechanics (FINAL, ABSTRACT, constructor chaining).

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "INHERITING FROM", "REDEFINITION", "Pseudo Reference super", "FINAL", "ABSTRACT".

**Design Patterns & Best Practices:** Clean ABAP → *Inheritance* (prefer composition over inheritance) (`github.com/SAP/styleguides`). Conceptual basis: the Liskov Substitution Principle (Topic 3.1) constrains correct subclassing.
