# ABAP Class Syntax (Local vs Global)

**Learning Objective:** After this topic you can write both local and global ABAP classes correctly, lay out the `DEFINITION`/`IMPLEMENTATION` parts, and choose which class kind a situation calls for.

**Difficulty Level:** Intermediate
**Time to Master:** 60–75 minutes
**Prerequisites:** `01_02_classes_and_objects.md`
**Official Sources:**
- ABAP Keyword Documentation → *CLASS*, *CLASS … DEFINITION*, *CLASS … IMPLEMENTATION* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Classes* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** You write a helper class inside a report. Three other reports later need the same logic. If it was a *local* class trapped in one program, you copy-paste it — and now four copies drift apart. If it had been a *global* class, all four reuse one definition.

**What Happens WITHOUT This Concept.** Not knowing the local/global distinction leads either to duplicated local classes (maintenance drift) or to bloating one program with classes that belong in the repository as reusable assets.

**Why This Matters in SAP.** SAP development is repository-centric. Reusable behaviour belongs in global classes (transportable, testable, discoverable); one-off helpers can stay local. Picking the right home is a daily decision.

---

## 3. Core Concept Explanation

**Definition.**
- A **local class** is defined inside a single program/include and is visible only there.
- A **global class** is a standalone repository object (built in ADT or SE24), transportable and reusable system-wide.

Both have the same two-part structure: a **`DEFINITION`** part (the declarations — what exists) and an **`IMPLEMENTATION`** part (the method bodies — how it works).

**Key Principles:**
- `DEFINITION` declares; `IMPLEMENTATION` implements. They are separate blocks.
- A global class is named like `ZCL_…` / `YCL_…` (customer namespace) or `CL_…` (SAP).
- `CREATE PUBLIC|PROTECTED|PRIVATE` controls *who may instantiate* the class.

**How It Works.** For a local class you write both blocks in your program. For a global class the tool stores the parts in repository sections, but conceptually it is the same `DEFINITION PUBLIC … ENDCLASS.` + `IMPLEMENTATION … ENDCLASS.`

**Why It's Designed This Way.** Splitting declaration from implementation lets the compiler resolve the full interface of a class before any bodies are compiled, and lets readers see the whole contract in one place.

---

## 4. Visual Representation

```
   LOCAL CLASS (inside one program)        GLOBAL CLASS (repository object)
   ┌─────────────────────────────┐         ┌──────────────────────────────┐
   │ REPORT z_demo.               │         │ CLASS zcl_pricing DEFINITION │
   │ CLASS lcl_x DEFINITION.      │         │   PUBLIC                     │
   │   ...                        │         │   CREATE PUBLIC.             │
   │ ENDCLASS.                    │         │   ... (Public/Protected/Priv)│
   │ CLASS lcl_x IMPLEMENTATION.  │         │ ENDCLASS.                    │
   │   ...                        │         │ CLASS zcl_pricing IMPLEMENT. │
   │ ENDCLASS.                    │         │   ...                        │
   │ START-OF-SELECTION. ...      │         │ ENDCLASS.                    │
   └─────────────────────────────┘         └──────────────────────────────┘
        visible: this program only              visible: whole system, transportable
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** The minimal correct skeleton of a class.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS lcl_x DEFINITION.
  PUBLIC SECTION.
    METHODS greet.
    METHOD greet.                  " wrong: implementation inside DEFINITION
      WRITE 'hi'.
    ENDMETHOD.
ENDCLASS.
```
**Problems with this code:**
- Method *bodies* may not live in the `DEFINITION` block — this does not compile.
- Mixing declaration and implementation defeats the two-part design.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_x DEFINITION.
  PUBLIC SECTION.
    METHODS greet.                 " DECLARE here
ENDCLASS.

CLASS lcl_x IMPLEMENTATION.
  METHOD greet.                    " IMPLEMENT here
    cl_demo_output=>write( 'hi' ).
  ENDMETHOD.
ENDCLASS.
```
**Why this is better:**
- The declaration (contract) and implementation (mechanics) are cleanly separated.
- It compiles, and readers see the full interface at a glance.

**Step-by-Step Explanation:**
- `CLASS lcl_x DEFINITION … ENDCLASS.` — the contract: every member is *declared* here.
- `CLASS lcl_x IMPLEMENTATION … ENDCLASS.` — every non-abstract method *declared* above gets a body here.
- `METHOD greet … ENDMETHOD.` — exactly one implementation per declared method.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A reusable, instantiable pricing class — written as it would appear as a *global* class source.

```abap
CLASS zcl_pricing DEFINITION
    PUBLIC                          " global, system-wide visibility
    FINAL                           " not meant to be subclassed
    CREATE PUBLIC.                  " anyone may instantiate it

  PUBLIC SECTION.
    METHODS constructor IMPORTING it_items   TYPE ztt_item
                                  iv_tax_rate TYPE p LENGTH 5 DECIMALS 4.
    METHODS gross_amount RETURNING VALUE(rv_gross) TYPE p LENGTH 13 DECIMALS 2.

  PRIVATE SECTION.
    DATA mt_items   TYPE ztt_item.
    DATA mv_taxrate TYPE p LENGTH 5 DECIMALS 4.
    METHODS net_amount RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
ENDCLASS.

CLASS zcl_pricing IMPLEMENTATION.
  METHOD constructor.
    mt_items   = it_items.
    mv_taxrate = iv_tax_rate.
  ENDMETHOD.

  METHOD net_amount.
    LOOP AT mt_items INTO DATA(ls_item).
      rv = rv + ls_item-price * ls_item-qty.
    ENDLOOP.
  ENDMETHOD.

  METHOD gross_amount.
    rv_gross = net_amount( ) * ( 1 + mv_taxrate ).
  ENDMETHOD.
ENDCLASS.

" Usage from any program once the global class is activated:
DATA(lo) = NEW zcl_pricing( it_items = lt_items iv_tax_rate = '0.1900' ).
WRITE lo->gross_amount( ).
```

**Detailed Walkthrough:**
- **`DEFINITION PUBLIC FINAL CREATE PUBLIC`** — the three common class options: repository visibility, no-subclassing, open instantiation.
- **`PUBLIC` vs `PRIVATE SECTION`** — the contract is small (constructor + `gross_amount`); helpers stay private.
- **`CREATE PUBLIC`** — controls *who may call `NEW`*; switching to `CREATE PRIVATE` forces a factory (Topic 3.3).

**How This Works in Practice.** As a global class, `zcl_pricing` is reusable by reports, RAP handlers, RFC wrappers, and unit tests across the system — written once.

**Why This Implementation.** `FINAL` + small public surface is the Clean ABAP default: classes are closed to subclassing unless extension is genuinely intended.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: An implementation for a method that was never declared.**
```abap
CLASS lcl_x IMPLEMENTATION.
  METHOD undeclared_method.    " not in DEFINITION → "method is unknown"
  ENDMETHOD.
ENDCLASS.
```
**Why this is wrong:** every implemented method must first be declared in `DEFINITION`.
**Correct approach:** declare it in the right visibility section first.

**Mistake #2: Wrong section order.**
```abap
CLASS lcl_x DEFINITION.
  PRIVATE SECTION.
    DATA mv_x TYPE i.
  PUBLIC SECTION.            " wrong: PUBLIC must precede PROTECTED/PRIVATE
    METHODS m.
ENDCLASS.
```
**Why this is wrong:** ABAP requires the order `PUBLIC SECTION` → `PROTECTED SECTION` → `PRIVATE SECTION`.
**Correct approach:** declare sections in that fixed order (each at most once).

---

## 8. Comparison With Similar Concepts

**Local vs Global class:** local lives in one program and is invisible elsewhere; global is a repository object, transportable and reusable. Start local for program-specific helpers; promote to global when a second consumer appears.

**Class vs Function group:** a function group is one shared state container with function modules; a class can be instantiated many times with independent state (see Topic 1.2). New reusable logic should generally be a class, not a new function group.

**`DEFINITION` vs `IMPLEMENTATION`:** the first is the *what* (the compiler reads it to know the type); the second is the *how* (the bodies). Abstract methods appear only in `DEFINITION`.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **ADT / SE24:** global classes are created and edited here; ADT (Eclipse) is the modern tool.
- **Transports:** global classes move through the landscape as repository objects.
- **RAP / OData:** behaviour implementations are global classes.
- **Local test classes:** unit tests are local classes attached to the global class's test include.

**SAP-Specific Considerations:** respect the customer namespace (`Z`/`Y` or a reserved namespace). Global classes support fixed-point arithmetic and Unicode settings via class attributes. Local classes cannot be reused or transported independently.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: a global class that is really a dumping ground of unrelated static methods.**
```abap
CLASS zcl_misc_utils DEFINITION PUBLIC.
  PUBLIC SECTION.
    CLASS-METHODS format_date.
    CLASS-METHODS read_config.
    CLASS-METHODS send_mail.      " unrelated responsibilities
ENDCLASS.
```
**Why this fails:** no cohesion; the class has many reasons to change (violates SRP, Topic 3.1).
**Correct approach:** split into cohesive classes by responsibility.

**Common Gotcha:** forgetting that `DEFINITION` and `IMPLEMENTATION` are two separate `CLASS … ENDCLASS.` blocks. Each method body lives only in the implementation block.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you write a class that compiles with one declared method and one implemented body, in the correct section order? Can you state why your class is local vs global?

**Unit Test Example:**
```abap
CLASS ltcl_pricing DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS gross_uses_tax FOR TESTING.
ENDCLASS.

CLASS ltcl_pricing IMPLEMENTATION.
  METHOD gross_uses_tax.
    DATA(lt) = VALUE ztt_item( ( price = 100 qty = 1 ) ).
    DATA(lo) = NEW zcl_pricing( it_items = lt iv_tax_rate = '0.1000' ).

    cl_abap_unit_assert=>assert_equals(
      act = lo->gross_amount( ) exp = CONV p( '110.00' ) ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the local test class (in the global class's test include) instantiates and exercises the public contract — the standard ABAP Unit setup.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Every class = a `DEFINITION` block (declares) + an `IMPLEMENTATION` block (bodies).
- Local class = one program only; global class = reusable, transportable repository object.
- Section order is fixed: `PUBLIC` → `PROTECTED` → `PRIVATE`; `CREATE` controls who may instantiate.

**When to Apply:** local for program-private helpers; global the moment a second consumer or a transportable asset is needed.

**Red Flags:** method bodies in `DEFINITION`; implementing undeclared methods; wrong section order; a global "misc utils" grab-bag.

---

## 13. Dependency Map

**Depends On:** `01_02_classes_and_objects.md` — the class/object model.

**Enables:**
- `02_02_attributes_instance_static.md`, `02_03_methods.md`, `02_05_visibility_sections.md` — fill in the sections introduced here.
- `02_04_constructors.md` — the special methods of the class.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "CLASS", "CLASS - DEFINITION", "CLASS - IMPLEMENTATION", "CREATE PUBLIC".

**Design Patterns & Best Practices:** Clean ABAP → *Classes* (prefer `FINAL`; keep classes small and cohesive) (`github.com/SAP/styleguides`).
