# Strategy Pattern

**Learning Objective:** After this topic you can encapsulate interchangeable algorithms behind an interface and select them at runtime via composition — replacing sprawling `CASE`/`IF` ladders with pluggable strategy objects.

**Difficulty Level:** Advanced
**Time to Master:** 60 minutes
**Prerequisites:** `01_05_polymorphism.md`, `02_06_interfaces_in_abap.md`, `03_01_solid_principles.md`
**Official Sources:**
- Clean ABAP → *Object Orientation* (prefer composition; avoid type checks) (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)
- ABAP Keyword Documentation → *Interfaces*, *Polymorphism* (`help.sap.com/doc/abapdocu_latest_index_htm`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** Shipping cost depends on the carrier: flat rate, weight-based, or zone-based. A single method grows a long `CASE carrier … ` with the three formulas inline. A fourth carrier means editing that method and retesting all branches.

**What Happens WITHOUT This Concept.** Algorithms that vary are baked into one method as conditional branches. The method grows without bound, every change risks unrelated branches, and you cannot reuse or test one algorithm in isolation.

**Why This Matters in SAP.** Business rules that vary by country, document type, customer class, or carrier are everywhere. Strategy turns "a branch per variant" into "a class per variant," making rules pluggable, individually testable, and table-drivable.

---

## 3. Core Concept Explanation

**Definition.** The **Strategy** pattern defines a family of interchangeable algorithms behind a common interface and lets the algorithm be selected and swapped independently of the code that uses it. The using object (context) holds a reference to a strategy and delegates to it.

**Key Principles:**
- Each algorithm becomes a class implementing a shared interface (e.g. `lif_shipping`).
- The context depends on the interface and is given a concrete strategy (composition/injection).
- Selecting/replacing the strategy is decoupled from running it.

**How It Works.** Instead of branching on a type code, the context calls `mo_strategy->calculate( … )`. Polymorphism (Topic 1.5) dispatches to whichever strategy was plugged in. A factory (Topic 3.3) usually chooses the concrete strategy.

**Why It's Designed This Way.** It is OCP (Topic 3.1) for algorithms: add a new behaviour by adding a class, never by editing the context. The varying part is isolated; the stable part stays closed.

---

## 4. Visual Representation

```
   CONTEXT (zcl_shipment)                 STRATEGIES (interchangeable)
   ┌───────────────────────────┐         lif_shipping
   │ mo_strategy : lif_shipping │──────►  ├── zcl_flat_rate    calculate()
   │ cost():                    │ delegates├── zcl_by_weight    calculate()
   │   = mo_strategy->calculate │         └── zcl_by_zone      calculate()
   └───────────────────────────┘
        swap strategy at runtime ⇄        add a carrier = add a class
        (no change to the context)        (context never edited → OCP)
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Compute shipping cost with a pluggable rule.

**The WRONG Way (Anti-Pattern):**
```abap
METHOD cost.
  CASE mv_carrier.
    WHEN 'FLAT'.   rv = 5.
    WHEN 'WEIGHT'. rv = mv_weight * '0.50'.
    WHEN 'ZONE'.   rv = zone_table[ mv_zone ].
    " add a carrier → edit here, retest everything
  ENDCASE.
ENDMETHOD.
```
**Problems with this code:**
- All algorithms entangled in one method; the `CASE` grows forever.
- Cannot test or reuse one rule alone.

**The RIGHT Way (Following Best Practice):**
```abap
INTERFACE lif_shipping.
  METHODS calculate IMPORTING iv_weight TYPE p LENGTH 7 DECIMALS 3
                    RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
ENDINTERFACE.

CLASS zcl_flat_rate DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION. INTERFACES lif_shipping.
ENDCLASS.
CLASS zcl_flat_rate IMPLEMENTATION.
  METHOD lif_shipping~calculate. rv = 5. ENDMETHOD.
ENDCLASS.

CLASS zcl_by_weight DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION. INTERFACES lif_shipping.
ENDCLASS.
CLASS zcl_by_weight IMPLEMENTATION.
  METHOD lif_shipping~calculate. rv = iv_weight * '0.50'. ENDMETHOD.
ENDCLASS.

" Context delegates to whatever strategy it holds:
CLASS zcl_shipment DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING io_strategy TYPE REF TO lif_shipping.
    METHODS cost IMPORTING iv_weight TYPE p LENGTH 7 DECIMALS 3
                 RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
  PRIVATE SECTION.
    DATA mo_strategy TYPE REF TO lif_shipping.
ENDCLASS.
CLASS zcl_shipment IMPLEMENTATION.
  METHOD constructor. mo_strategy = io_strategy. ENDMETHOD.
  METHOD cost.        rv = mo_strategy->calculate( iv_weight ). ENDMETHOD.
ENDCLASS.

DATA(lo) = NEW zcl_shipment( io_strategy = NEW zcl_by_weight( ) ).
WRITE lo->cost( iv_weight = '10.000' ).   " 5.00
```
**Why this is better:**
- Each rule is its own class — testable and reusable alone.
- A new carrier is a new class; `zcl_shipment` never changes.

**Step-by-Step Explanation:**
- `INTERFACE lif_shipping` — the common contract for all algorithms.
- Each strategy class implements `calculate` its own way.
- `zcl_shipment` holds `lif_shipping` and delegates — no branching.

---

## 6. Code Example 2: Real-World Application (strategy chosen by a factory, swappable at runtime)

**Business Scenario:** The carrier is chosen from configuration via a factory, and a premium customer can switch to a different strategy mid-process without rebuilding the shipment.

```abap
CLASS zcl_shipping_factory DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    CLASS-METHODS for_carrier IMPORTING iv_carrier TYPE string
                              RETURNING VALUE(ro)  TYPE REF TO lif_shipping
                              RAISING   cx_sy_create_object_error.
ENDCLASS.
CLASS zcl_shipping_factory IMPLEMENTATION.
  METHOD for_carrier.
    ro = SWITCH #( iv_carrier
      WHEN 'FLAT'   THEN NEW zcl_flat_rate( )
      WHEN 'WEIGHT' THEN NEW zcl_by_weight( )
      ELSE THROW cx_sy_create_object_error( ) ).
  ENDMETHOD.
ENDCLASS.

" Context gains a setter so the strategy can change at runtime:
CLASS zcl_shipment DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING io_strategy TYPE REF TO lif_shipping.
    METHODS set_strategy IMPORTING io_strategy TYPE REF TO lif_shipping.
    METHODS cost IMPORTING iv_weight TYPE p LENGTH 7 DECIMALS 3
                 RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
  PRIVATE SECTION.
    DATA mo_strategy TYPE REF TO lif_shipping.
ENDCLASS.
CLASS zcl_shipment IMPLEMENTATION.
  METHOD constructor.  mo_strategy = io_strategy. ENDMETHOD.
  METHOD set_strategy. mo_strategy = io_strategy. ENDMETHOD.
  METHOD cost.         rv = mo_strategy->calculate( iv_weight ). ENDMETHOD.
ENDCLASS.

" Usage — config-driven selection, then a runtime swap:
DATA(lo_ship) = NEW zcl_shipment(
                  io_strategy = zcl_shipping_factory=>for_carrier( 'FLAT' ) ).
DATA(lv1) = lo_ship->cost( '10.000' ).                 " flat: 5.00

lo_ship->set_strategy( zcl_shipping_factory=>for_carrier( 'WEIGHT' ) ).
DATA(lv2) = lo_ship->cost( '10.000' ).                 " now weight-based: 5.00
```

**Detailed Walkthrough:**
- **Factory chooses the strategy** — selection logic lives once (Topic 3.3), not in the context.
- **`set_strategy`** — the algorithm is swappable at runtime, the defining trait of Strategy.
- **Context unchanged for new carriers** — add a class + a factory branch; `cost( )` never changes.

**How This Works in Practice.** Strategy + Factory is a standard pairing: the factory decides *which* algorithm (often from customizing), Strategy makes it pluggable into the context. New rules ship as new classes.

**Why This Implementation.** Composition over inheritance (Clean ABAP) — the context *has-a* algorithm rather than *being* a particular subclass, so behaviour can change at runtime, not just at compile time.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: A "strategy" that still branches on a type code.**
```abap
METHOD lif_shipping~calculate.
  CASE mv_kind.                    " the CASE just moved inside the "strategy"
    WHEN 'A'. rv = ...
    WHEN 'B'. rv = ...
  ENDCASE.
ENDMETHOD.
```
**Why this is wrong:** the branching anti-pattern was relocated, not removed; this is one class pretending to be several.
**Correct approach:** one class per algorithm, each with a single, branch-free `calculate`.

**Mistake #2: Context coupled to concrete strategies.**
```abap
DATA mo_strategy TYPE REF TO zcl_by_weight.   " concrete, not the interface
```
**Why this is wrong:** the context can now hold only one concrete type; you've lost interchangeability.
**Correct approach:** type the reference as the interface `lif_shipping`.

---

## 8. Comparison With Similar Concepts

**Strategy vs Template Method (Topic 2.7):** both vary an algorithm. Template Method uses *inheritance* — subclasses override steps of a fixed flow (compile-time, one variation axis). Strategy uses *composition* — swap the whole algorithm object at runtime. Prefer Strategy when behaviour must change dynamically or be combined freely.

**Strategy vs State pattern:** structurally similar (delegate to a pluggable object); intent differs. Strategy varies *how* one operation is done; State varies behaviour as an object's internal *state* changes, often switching itself.

**Strategy vs `CASE`/`IF`:** the branch ladder is fine for a small, fixed, local choice; Strategy pays off when variants are many, grow over time, must be tested individually, or are config-driven.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Factory (Topic 3.3) & DI (Topic 3.5):** select and inject the strategy; the composition root or customizing decides which.
- **BAdIs:** an enhancement spot is essentially a strategy seam — implementations are interchangeable algorithms.
- **Events (Topic 2.9):** different concerns; events fan out notifications, Strategy swaps one computation.

**SAP-Specific Considerations:** strategies map naturally onto customizing tables (carrier → class name), letting consultants change behaviour by configuration. Keep each strategy stateless where possible so one instance can be reused.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: an interface per trivial, never-varying operation.**
```abap
INTERFACE lif_add. METHODS run IMPORTING a TYPE i b TYPE i RETURNING VALUE(r) TYPE i. ENDINTERFACE.
" ...for a one-line addition that will never have a second variant
```
**Why this fails:** Strategy adds classes and indirection; for a single, stable algorithm that overhead buys nothing.
**Correct approach:** introduce Strategy when there are (or will credibly be) multiple variants; otherwise a plain method is right.

**Common Gotcha:** if strategies need shared context data, pass it as method parameters rather than giving each strategy a back-reference to the context — back-references re-couple the pieces you separated.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you add a new algorithm without editing the context, and unit-test each algorithm in isolation? If adding a variant edits the context, it isn't Strategy.

**Unit Test Example:**
```abap
CLASS ltcl_shipping DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS weight_strategy_alone FOR TESTING.
    METHODS context_delegates    FOR TESTING.
ENDCLASS.
CLASS ltcl_shipping IMPLEMENTATION.
  METHOD weight_strategy_alone.
    " each algorithm is testable on its own:
    DATA lo_s TYPE REF TO lif_shipping.
    lo_s = NEW zcl_by_weight( ).
    cl_abap_unit_assert=>assert_equals( act = lo_s->calculate( '4.000' ) exp = CONV p( '2.00' ) ).
  ENDMETHOD.
  METHOD context_delegates.
    " context can be tested with a stub strategy:
    DATA(lo_ship) = NEW zcl_shipment( io_strategy = NEW zcl_flat_rate( ) ).
    cl_abap_unit_assert=>assert_equals( act = lo_ship->cost( '99.000' ) exp = CONV p( '5.00' ) ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** strategies are tested standalone and the context is tested with a known strategy — the isolation that the `CASE`-ladder version could not offer.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Strategy = a family of interchangeable algorithms behind one interface, selected/swapped via composition.
- The context depends on the interface and delegates; new algorithms are new classes (OCP).
- Pair with a factory to choose the strategy (often from customizing); swap at runtime via a setter.

**When to Apply:** an operation has multiple variants that grow, must be tested individually, or are configuration-driven.

**Red Flags:** a `CASE` hidden inside a strategy; context typed to a concrete strategy; interfaces invented for single, stable operations.

---

## 13. Dependency Map

**Depends On:**
- `01_05_polymorphism.md` — runtime dispatch is the engine of Strategy.
- `02_06_interfaces_in_abap.md` — the shared algorithm contract.
- `03_01_solid_principles.md` — OCP/DIP motivate it.

**Enables:**
- `03_05_dependency_injection.md` — strategies are injected.
- Configuration-driven business-rule designs.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "Interfaces", "Interface Reference Variables", "SWITCH".

**Design Patterns & Best Practices:** Pattern origin: Strategy (Gang of Four). Clean ABAP → prefer composition over inheritance; avoid type checks / `CASE` on type (`github.com/SAP/styleguides`).
