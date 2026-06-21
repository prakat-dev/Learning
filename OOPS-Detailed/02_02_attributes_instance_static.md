# Attributes: Instance vs Static

**Learning Objective:** After this topic you can choose correctly between instance attributes (`DATA`), static attributes (`CLASS-DATA`), and constants, and explain why static state is shared across all objects of a class.

**Difficulty Level:** Intermediate
**Time to Master:** 50–60 minutes
**Prerequisites:** `02_01_abap_class_syntax.md`
**Official Sources:**
- ABAP Keyword Documentation → *DATA*, *CLASS-DATA*, *CONSTANTS*, *READ-ONLY* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Members* / *Constants* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** You want every order object to carry its own total (per-object state) but you also want one shared counter of how many orders were created in this session (one value for the whole class). These are two different *kinds* of state.

**What Happens WITHOUT This Concept.** Putting the shared counter in instance state means every object has its own copy and the count is always 1; putting per-order totals in static state means all orders overwrite one value. Mixing the two up causes subtle, hard-to-find data bugs.

**Why This Matters in SAP.** Static attributes are effectively per-class globals for the running session — powerful (caches, counters, configuration) and dangerous (hidden shared state). Knowing the difference prevents both bugs and accidental coupling.

---

## 3. Core Concept Explanation

**Definition.**
- An **instance attribute** (`DATA`) belongs to *each object*; every instance has its own copy.
- A **static attribute** (`CLASS-DATA`) belongs to the *class*; there is exactly one copy, shared by all instances (and accessible without any instance).
- A **constant** (`CONSTANTS`) is a fixed, named value that never changes.

**Key Principles:**
- Default to instance state; use static state deliberately and sparingly.
- Static attributes live for the session — they are shared mutable state, so guard them.
- Prefer `CONSTANTS` over magic literals; expose read-only state with `READ-ONLY`.

**How It Works.** `DATA mv_x` reserves space in every object. `CLASS-DATA gv_count` reserves one slot at class level, accessed as `class=>gv_count` or, from inside, just `gv_count`. Static attributes are initialized by the **class constructor** (Topic 2.4) the first time the class is used.

**Why It's Designed This Way.** Separating per-object from per-class state lets a class model both "each thing's data" and "facts about the kind of thing," without forcing one to imitate the other.

---

## 4. Visual Representation

```
   CLASS zcl_order
   ┌──────────────────────────── class level (ONE copy) ───────────────┐
   │  CLASS-DATA gv_count   = 3      ← shared by all instances          │
   │  CONSTANTS  c_currency = 'EUR'  ← fixed                            │
   └───────────────────────────────────────────────────────────────────┘
        │ instantiation
        ▼
   ┌───────────┐   ┌───────────┐   ┌───────────┐
   │ object 1  │   │ object 2  │   │ object 3  │   instance level
   │ mv_total  │   │ mv_total  │   │ mv_total  │   (its OWN copy each)
   │   = 100   │   │   = 250   │   │   = 90    │
   └───────────┘   └───────────┘   └───────────┘
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Per-object total plus a shared creation counter.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    DATA mv_count TYPE i.        " wrong: a per-instance "counter" is always local
    METHODS constructor.
ENDCLASS.
CLASS lcl_order IMPLEMENTATION.
  METHOD constructor.
    mv_count = mv_count + 1.     " every object sees its own 0 → ends at 1
  ENDMETHOD.
ENDCLASS.
```
**Problems with this code:**
- `mv_count` is per-object, so it can never count instances across the class.
- The intent (a class-wide count) is mismatched to instance storage.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    CLASS-DATA gv_count TYPE i READ-ONLY.       " ONE shared, externally read-only
    METHODS constructor.
    DATA mv_total TYPE p LENGTH 13 DECIMALS 2.  " per-object
ENDCLASS.
CLASS lcl_order IMPLEMENTATION.
  METHOD constructor.
    gv_count = gv_count + 1.     " increments the single shared slot
  ENDMETHOD.
ENDCLASS.

DATA(lo1) = NEW lcl_order( ).
DATA(lo2) = NEW lcl_order( ).
WRITE lcl_order=>gv_count.        " 2  (accessed via the CLASS, no instance needed)
```
**Why this is better:**
- One shared `gv_count` correctly tallies all instances.
- `mv_total` stays per-object; the two kinds of state are matched to their storage.

**Step-by-Step Explanation:**
- `CLASS-DATA gv_count … READ-ONLY` — one slot for the whole class; outsiders may read but not write it.
- `class=>gv_count` — static members are addressed through the class name with `=>`.
- `DATA mv_total` — ordinary per-object state.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A configuration value (currency) is a constant; a small in-session cache of exchange rates is shared static state with controlled access.

```abap
CLASS zcl_fx DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    CONSTANTS c_base_currency TYPE waers VALUE 'EUR'.   " fixed configuration
    METHODS rate_to IMPORTING iv_curr TYPE waers
                    RETURNING VALUE(rv_rate) TYPE p LENGTH 9 DECIMALS 5.
  PRIVATE SECTION.
    " shared cache: filled once, reused by every instance this session
    CLASS-DATA gt_cache TYPE SORTED TABLE OF zfx_rate WITH UNIQUE KEY curr.
    METHODS load_rate IMPORTING iv_curr TYPE waers
                      RETURNING VALUE(rv_rate) TYPE p LENGTH 9 DECIMALS 5.
ENDCLASS.

CLASS zcl_fx IMPLEMENTATION.
  METHOD rate_to.
    READ TABLE gt_cache INTO DATA(ls) WITH KEY curr = iv_curr.
    IF sy-subrc = 0.
      rv_rate = ls-rate.                      " cache hit (shared across instances)
    ELSE.
      rv_rate = load_rate( iv_curr ).         " miss → load, then cache
      INSERT VALUE #( curr = iv_curr rate = rv_rate ) INTO TABLE gt_cache.
    ENDIF.
  ENDMETHOD.

  METHOD load_rate.
    " ... read from DB / service; abbreviated ...
    rv_rate = '1.00000'.
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **`CONSTANTS c_base_currency`** — a fixed value, no magic literal scattered around.
- **`CLASS-DATA gt_cache`** — a single cache shared by all `zcl_fx` objects this session; the second instance benefits from the first instance's loads.
- **Private static + private helper** — the cache is hidden; access is funnelled through `rate_to`.

**How This Works in Practice.** Two callers each create a `zcl_fx`; the first lookup of `'USD'` loads and caches it, the second reuses it — because the cache is class-level, not per-object.

**Why This Implementation.** Shared caches are a legitimate use of static state, *provided* access is encapsulated. Public mutable static state would be the anti-pattern.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Public, writable static state.**
```abap
CLASS-DATA gv_config TYPE string.   " public + writable
" anywhere: zcl_x=>gv_config = '...'   ← a global variable with a class prefix
```
**Why this is wrong:** it is a global mutable variable; any code can change it, reintroducing the shared-state problems from Topic 1.1, system-wide.
**Correct approach:** keep static state `PRIVATE` (or `READ-ONLY`); change it only through methods.

**Mistake #2: Expecting static state to reset per object.**
```abap
" Assuming a new instance gives a fresh gv_count — it does NOT; static lives for the session.
```
**Why this is wrong:** static attributes persist for the program's session, not per instance; they are initialized once by the class constructor.
**Correct approach:** if you need per-object data, use `DATA`, not `CLASS-DATA`.

---

## 8. Comparison With Similar Concepts

**Instance (`DATA`) vs Static (`CLASS-DATA`):** instance = one copy per object; static = one copy per class for the whole session. Use static only for genuinely class-wide facts (counters, caches, registries).

**Static attribute vs Constant:** a constant never changes and is set at declaration; a static attribute is mutable shared state. Prefer constants for fixed values.

**`READ-ONLY` vs getter method:** `READ-ONLY` gives cheap external read access to a public attribute; a getter method is better when reading should compute or validate (Topic 1.3).

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Singletons (Topic 3.2):** rely on a private static reference holding the single instance.
- **Caches:** static tables cache read-mostly data within a session/work process.
- **Class constructor (Topic 2.4):** initializes static attributes exactly once.

**SAP-Specific Considerations:** static state lives per *internal session* (roughly, per running program context), **not** across users or work processes — for cross-session sharing you need Shared Objects (shared memory). Large static caches consume work-process memory; size them deliberately. Static state can leak between test methods, so reset it in test `setup`.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: hidden global configuration via public static.**
```abap
CLASS zcl_settings DEFINITION PUBLIC.
  PUBLIC SECTION.
    CLASS-DATA gv_debug_mode TYPE abap_bool.   " toggled from anywhere
ENDCLASS.
```
**Why this fails:** invisible coupling; tests interfere with each other; behaviour depends on who last wrote the flag.
**Correct approach:** pass configuration in (constructor/parameters) or encapsulate behind methods.

**Common Gotcha:** static attributes are **not** cleared when you create a new object or even re-run within the same session — they persist until the session ends. Reset them explicitly in unit test setup to avoid test bleed-through.

---

## 11. Testing & Validation

**How to Verify Understanding:** For each attribute, ask "one per object, or one for the class?" If you cannot answer, the design intent is unclear.

**Unit Test Example:**
```abap
CLASS ltcl_order DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS setup.
    METHODS counter_is_shared FOR TESTING.
ENDCLASS.

CLASS ltcl_order IMPLEMENTATION.
  METHOD setup.
    lcl_order=>gv_count = 0.          " reset shared static to avoid test bleed-through
  ENDMETHOD.
  METHOD counter_is_shared.
    DATA(lo1) = NEW lcl_order( ).
    DATA(lo2) = NEW lcl_order( ).
    cl_abap_unit_assert=>assert_equals( act = lcl_order=>gv_count exp = 2 ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** `setup` resets the shared counter (because static state survives between tests), then the test proves two instances share one counter.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- `DATA` = per-object; `CLASS-DATA` = one shared copy per class; `CONSTANTS` = fixed values.
- Static state is session-lived shared state — keep it private/read-only and guard it.
- Reset static state in test setup; it does not auto-reset per object.

**When to Apply:** instance state by default; static only for true class-wide facts (counters, caches, the singleton reference).

**Red Flags:** public writable `CLASS-DATA`; expecting static to reset per instance; magic literals that should be `CONSTANTS`.

---

## 13. Dependency Map

**Depends On:** `02_01_abap_class_syntax.md` — attributes live in the class sections.

**Enables:**
- `02_04_constructors.md` — the class constructor initializes static attributes.
- `03_02_singleton_pattern.md` — uses a private static instance reference.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "DATA" (instance), "CLASS-DATA" (static), "CONSTANTS", "READ-ONLY".

**Design Patterns & Best Practices:** Clean ABAP → prefer constants to literals; avoid public mutable members (`github.com/SAP/styleguides`).
