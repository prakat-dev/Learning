# Why Object-Oriented Programming?

**Learning Objective:** After this topic you can explain, in terms of concrete maintenance and change problems, *why* OOP exists and when reaching for it pays off in ABAP — independently of any syntax.

**Difficulty Level:** Foundational
**Time to Master:** 60–90 minutes
**Prerequisites:** Procedural ABAP (reports, `FORM` routines, function modules, internal tables)
**Official Sources:**
- ABAP Keyword Documentation → *ABAP Objects* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Classes* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A sales report starts as 200 lines. Over three years it grows to 2,500 lines across 40 `FORM` routines that all read and write the same 30 global `DATA` fields. A new requirement — "apply a customer-group discount" — touches pricing logic that is smeared across `calculate_net`, `apply_tax`, and `build_output`. Nobody can change one without breaking the others, because everything shares global state.

**What Happens WITHOUT OOP (in this style):**

```abap
" Anti-pattern: global state shared by every FORM
DATA: gv_net      TYPE p DECIMALS 2,
      gv_tax      TYPE p DECIMALS 2,
      gv_customer TYPE kunnr,
      gt_items    TYPE STANDARD TABLE OF ty_item.

FORM calculate_net.       " mutates gv_net using gt_items + gv_customer
FORM apply_tax.           " mutates gv_tax, also peeks at gv_net
FORM build_output.        " reads gv_net, gv_tax, gv_customer ... and more
```

The pain points: any `FORM` can change any global field; you cannot reuse `calculate_net` elsewhere without dragging the globals along; you cannot test `apply_tax` in isolation because it depends on hidden state set by another `FORM`; and a small change ripples unpredictably.

**Why This Matters in SAP.** ABAP systems live for *decades*. The cost of software is dominated by change, not initial writing. Procedural global-state code optimizes for writing it once and punishes every future change — exactly the wrong trade-off for long-lived ERP code.

---

## 3. Core Concept Explanation

**Definition.** **Object-oriented programming** organizes a program around **objects** — self-contained units that bundle *data* (state) together with the *behaviour* (operations) that acts on that data, and that expose only a deliberate, stable interface to the outside.

**Key Principles** (each is a full topic later):
- **Encapsulation** — hide internal state; expose behaviour. State changes only through the object's own methods.
- **Abstraction** — callers depend on *what* an object does, not *how*.
- **Inheritance** — specialize an existing type without rewriting it.
- **Polymorphism** — interchangeable types behind one interface.

**How It Works.** Instead of global data + free functions, you create objects that *own* their data. A `pricing` object holds the items and customer internally; you ask it `net_amount( )` and it answers. The data it uses is *its* data — no other code can reach in and corrupt it.

**Why It's Designed This Way.** Bundling state with behaviour and hiding the state means a change to *how* pricing works stays inside the pricing object. The blast radius of a change shrinks from "the whole program" to "one class." That containment is the entire economic argument for OOP.

---

## 4. Visual Representation

```
   PROCEDURAL                          OBJECT-ORIENTED
   (data and behaviour separate)       (data and behaviour bundled, state hidden)

   ┌───────── GLOBAL DATA ─────────┐   ┌──────── object: pricing ────────┐
   │ gv_net  gv_tax  gt_items ...  │   │  - items     (private)          │
   └───────────────────────────────┘   │  - customer  (private)          │
        ▲     ▲     ▲     ▲             │  ────────────────────────────   │
        │     │     │     │             │  + net_amount( )  : value       │
   ┌────┴─┐ ┌─┴───┐ ┌┴────┐            │  + tax_amount( )  : value       │
   │FORM a│ │FORM b│ │FORM c│  any FORM  └──────────────────────────────────┘
   └──────┘ └──────┘ └──────┘  can touch       callers see only the methods;
            any global                          state is unreachable
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Compute a net amount for a set of order items.

**The WRONG Way (Anti-Pattern):**
```abap
DATA: gt_items TYPE STANDARD TABLE OF ty_item,
      gv_net   TYPE p DECIMALS 2.

FORM calculate_net.
  CLEAR gv_net.
  LOOP AT gt_items INTO DATA(ls_item).
    gv_net = gv_net + ls_item-price * ls_item-qty.  " writes a global
  ENDLOOP.
ENDFORM.
```
**Problems with this code:**
- **Hidden coupling:** the result lives in `gv_net`, which *anything* can overwrite before you read it.
- **Not reusable:** you cannot compute two independent totals at once — there is one global.
- **Untestable:** to test it you must set up globals and call a `FORM` for its side effect.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    METHODS constructor IMPORTING it_items TYPE tt_item.
    METHODS net_amount RETURNING VALUE(rv_net) TYPE p_amount. " ask, get answer
  PRIVATE SECTION.
    DATA mt_items TYPE tt_item.                                " state is OWNED + HIDDEN
ENDCLASS.

CLASS lcl_order IMPLEMENTATION.
  METHOD constructor.
    mt_items = it_items.                 " object remembers its own items
  ENDMETHOD.
  METHOD net_amount.
    LOOP AT mt_items INTO DATA(ls_item).
      rv_net = rv_net + ls_item-price * ls_item-qty.  " writes a local, returns it
    ENDLOOP.
  ENDMETHOD.
ENDCLASS.
```
**Why this is better:**
- **No shared state:** each `lcl_order` instance owns its items; two orders never interfere.
- **Reusable:** create as many orders as you like.
- **Testable:** construct, call `net_amount( )`, assert the return value — no globals.

**Step-by-Step Explanation:**
- `PRIVATE SECTION ... mt_items` — the data is locked inside the object; no outsider can read or write it.
- `constructor ... mt_items = it_items` — the object is handed its data once, at birth.
- `net_amount RETURNING VALUE(rv_net)` — a *functional* method: input in, value out, no side effects on shared state.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A pricing service must total an order, then add tax, and be reusable by both an online report and a background job — without those two callers sharing any state.

```abap
CLASS lcl_pricing DEFINITION.
  PUBLIC SECTION.
    METHODS constructor IMPORTING it_items    TYPE tt_item
                                  iv_tax_rate  TYPE p_rate.
    METHODS net_amount   RETURNING VALUE(rv_net)   TYPE p_amount.
    METHODS tax_amount   RETURNING VALUE(rv_tax)   TYPE p_amount.
    METHODS gross_amount RETURNING VALUE(rv_gross) TYPE p_amount.
  PRIVATE SECTION.
    DATA mt_items   TYPE tt_item.
    DATA mv_taxrate TYPE p_rate.
ENDCLASS.

CLASS lcl_pricing IMPLEMENTATION.
  METHOD constructor.
    mt_items   = it_items.
    mv_taxrate = iv_tax_rate.
  ENDMETHOD.

  METHOD net_amount.
    LOOP AT mt_items INTO DATA(ls_item).
      rv_net = rv_net + ls_item-price * ls_item-qty.
    ENDLOOP.
  ENDMETHOD.

  METHOD tax_amount.
    rv_tax = net_amount( ) * mv_taxrate.    " reuses own behaviour
  ENDMETHOD.

  METHOD gross_amount.
    rv_gross = net_amount( ) + tax_amount( ).
  ENDMETHOD.
ENDCLASS.

" Two independent callers — no shared globals between them:
DATA(lo_cart)  = NEW lcl_pricing( it_items = lt_cart_items  iv_tax_rate = '0.19' ).
DATA(lo_quote) = NEW lcl_pricing( it_items = lt_quote_items iv_tax_rate = '0.07' ).
WRITE: / lo_cart->gross_amount( ), / lo_quote->gross_amount( ).
```

**Detailed Walkthrough:**
- **Construction:** each caller builds its *own* pricing object with its own items and rate.
- **Composition of behaviour:** `gross_amount` calls `net_amount` and `tax_amount`; the logic is defined once and reused internally.
- **Isolation:** `lo_cart` and `lo_quote` cannot affect each other — the old global-state bug is structurally impossible.

**How This Works in Practice.** The report and the background job each instantiate `lcl_pricing` independently. They never collide, even if they run in the same session, because the state is per-object.

**Why This Implementation.** Functional methods (input → return value) make each calculation independently verifiable, which is what makes the unit test in §11 trivial.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: "OO" code that is really procedural (public mutable state).**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    DATA mt_items TYPE tt_item.   " PUBLIC + writable = a global with extra steps
    METHODS net_amount RETURNING VALUE(rv_net) TYPE p_amount.
ENDCLASS.
" elsewhere:
lo_order->mt_items = lt_something.   " any caller mutates internals → no encapsulation
```
**Why this is wrong:** exposing writable state recreates the global-variable problem; the object can be put into an invalid state from outside.
**Correct approach:** keep state `PRIVATE`; set it via the constructor or intention-revealing methods.

**Mistake #2: One giant method (a `FORM` in a class costume).**
```abap
METHOD do_everything.
  " 300 lines: select, calculate, format, output, email...
ENDMETHOD.
```
**Why this is wrong:** no separation of concerns; impossible to reuse or test a part; the class isn't really object-oriented, just a procedure relocated.
**Correct approach:** one method, one responsibility — `net_amount( )`, `tax_amount( )`, etc.

---

## 8. Comparison With Similar Concepts

**OOP vs Procedural (`FORM`/function modules):** procedural separates data from the routines that use it, so data is typically global and shared; OOP bundles them and hides the data. Procedural is fine for short, linear scripts; OOP earns its keep when logic is reused, varied, or expected to change.

**OOP vs "modular" function groups:** a function group with its global `DATA` is a *single* shared instance of state. A class can be instantiated many times, each with independent state — the difference that fixes the two-callers-collide bug.

**When to Use Each:** reach for OOP when you have behaviour that varies, must be reused, must be tested in isolation, or will change. A one-off five-line conversion does not need a class.

---

## 9. Integration With the SAP Ecosystem

**How OOP works with:**
- **Reports & transactions:** the report becomes thin — it gathers input and delegates to classes that hold the logic.
- **BAPI / RFC:** classes commonly *wrap* BAPIs, converting BAPI return tables into clean methods and class-based exceptions (see Topic 2.10).
- **SAP Fiori / OData / RAP:** modern SAP application programming (RAP) is class- and method-based throughout; OO is not optional there.
- **Workflows:** business object behaviour is increasingly modelled with classes.

**SAP-Specific Considerations:** object creation has a small cost — do not instantiate inside tight loops needlessly; objects live in the session's memory, so release references you no longer need; encapsulation also supports cleaner authorization checks at method boundaries.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the "God class"** — one class that does selection, calculation, formatting, and persistence.
```abap
CLASS lcl_everything DEFINITION.
  PUBLIC SECTION.
    METHODS run.   " 1,000 lines hiding behind one method
ENDCLASS.
```
**Why this fails:** it has every problem the 2,500-line report had, now harder to spot because it *looks* OO.
**Correct approach:** split by responsibility; let classes collaborate.

**Common Gotcha:** writing a class but exposing all state as `PUBLIC DATA`. Syntax alone does not make code object-oriented — *design* does. If outsiders can freely mutate your fields, you have a global in disguise.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you state, for a piece of code, the *change* that OOP would make cheaper? If not, OOP may be premature.

**Unit Test Example:**
```abap
CLASS ltcl_pricing DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS gross_is_net_plus_tax FOR TESTING.
ENDCLASS.

CLASS ltcl_pricing IMPLEMENTATION.
  METHOD gross_is_net_plus_tax.
    DATA(lt_items) = VALUE tt_item( ( price = 100 qty = 2 ) ).   " net = 200
    DATA(lo) = NEW lcl_pricing( it_items = lt_items iv_tax_rate = '0.10' ).

    cl_abap_unit_assert=>assert_equals(
      act = lo->gross_amount( )      " 200 + 20
      exp = CONV p_amount( 220 )
      msg = 'gross should equal net plus tax' ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** it builds a pricing object, calls a functional method, and asserts the return value. No globals, no database, no setup of hidden state — which is exactly the testability OOP buys.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- OOP bundles data with behaviour and hides the data; this *contains* the cost of change.
- The win is economic: smaller blast radius, reuse, and testability — not syntax.
- A class with public mutable state or one giant method is procedural code wearing OO syntax.

**When to Apply:** behaviour that varies, repeats, must be tested in isolation, or will change over a long life.

**Red Flags:** public writable attributes; one method doing everything; a class you could not unit-test without a database.

---

## 13. Dependency Map

**Depends On:** — (entry point of the curriculum).

**Enables:**
- `01_02_classes_and_objects.md` — the concrete vocabulary (class vs object) for everything above.
- `01_03_encapsulation.md` — the principle that makes "hide the state" precise.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation → *ABAP Objects* overview (search "ABAP Objects"). For functional methods, search "Functional Methods".

**Design Patterns & Best Practices:** Clean ABAP → *Classes* and *Methods* sections (`github.com/SAP/styleguides`). Conceptual grounding: Robert C. Martin, *Clean Code* (the book Clean ABAP adapts for ABAP).
