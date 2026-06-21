# Singleton Pattern

**Learning Objective:** After this topic you can implement a correct ABAP singleton (`CREATE PRIVATE` + static `get_instance`), understand its session scope, and — just as importantly — know when *not* to use it and what to prefer instead.

**Difficulty Level:** Advanced
**Time to Master:** 60 minutes
**Prerequisites:** `02_02_attributes_instance_static.md`, `02_04_constructors.md`, `02_11_friends.md`
**Official Sources:**
- ABAP Keyword Documentation → *CREATE PRIVATE*, *CLASS-DATA*, *CLASS-METHODS* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Object Orientation* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** You have an in-memory configuration cache that is expensive to build and should exist exactly once per session. Every part of the program must use the *same* instance, and nobody should be able to create a second one.

**What Happens WITHOUT This Concept.** If any caller can `NEW` the cache, you get several instances, duplicated loads, and inconsistent state across the program. Coordinating "exactly one" by convention always eventually fails.

**Why This Matters in SAP.** Singletons appear throughout SAP for per-session shared services (caches, registries, configuration). Knowing the correct implementation — and its serious downsides — lets you use it where appropriate and avoid the global-state trap elsewhere.

---

## 3. Core Concept Explanation

**Definition.** The **Singleton** pattern guarantees a class has exactly one instance and provides a global access point to it. In ABAP: a `CREATE PRIVATE` class with a private static reference to its single instance and a public static `get_instance( )` that creates it on first call and returns it thereafter.

**Key Principles:**
- `CREATE PRIVATE` blocks external `NEW` — only the class itself can instantiate.
- A private `CLASS-DATA go_instance` holds the one instance.
- `get_instance( )` lazily creates it once, then returns the same reference.
- Scope in ABAP is the **internal session**, not the whole system or all users.

**How It Works.** The first `get_instance( )` call finds `go_instance` unbound, creates the instance via the private constructor, stores it, and returns it. Subsequent calls return the stored reference — the same object for the session.

**Why It's Designed This Way.** Combining `CREATE PRIVATE` (no outside creation) with a controlled static accessor is the only way to *enforce* "exactly one" rather than merely hoping callers cooperate.

---

## 4. Visual Representation

```
   first call                      later calls
   ───────────                     ───────────
   get_instance( )                 get_instance( )
        │ go_instance is initial?       │ go_instance bound?
        ▼ yes → create once             ▼ yes → return same ref
   go_instance = NEW (private ctor)     return go_instance
        │
        ▼
   ┌───────────────────────────┐
   │ THE one instance          │  CREATE PRIVATE: nobody else can NEW it
   │ (private CLASS-DATA)       │  scope = current internal session
   └───────────────────────────┘
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** A configuration cache that must exist once per session.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS zcl_config DEFINITION PUBLIC CREATE PUBLIC.   " anyone can NEW → many instances
  PUBLIC SECTION.
    METHODS get IMPORTING iv_key TYPE string RETURNING VALUE(rv) TYPE string.
ENDCLASS.
" callers each do:
DATA(lo1) = NEW zcl_config( ).    " loads cache
DATA(lo2) = NEW zcl_config( ).    " loads AGAIN, separate state
```
**Problems with this code:**
- Multiple instances, duplicated expensive loads, divergent state.
- "Exactly one" is unenforceable.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS zcl_config DEFINITION PUBLIC CREATE PRIVATE.   " no outside NEW
  PUBLIC SECTION.
    CLASS-METHODS get_instance RETURNING VALUE(ro) TYPE REF TO zcl_config.
    METHODS get IMPORTING iv_key TYPE string RETURNING VALUE(rv) TYPE string.
  PRIVATE SECTION.
    CLASS-DATA go_instance TYPE REF TO zcl_config.   " the single instance
    METHODS constructor.                              " private: only the class creates it
    DATA mt_cache TYPE SORTED TABLE OF zconfig WITH UNIQUE KEY key.
ENDCLASS.
CLASS zcl_config IMPLEMENTATION.
  METHOD get_instance.
    IF go_instance IS NOT BOUND.
      go_instance = NEW zcl_config( ).   " created exactly once (lazy)
    ENDIF.
    ro = go_instance.
  ENDMETHOD.
  METHOD constructor.
    SELECT * FROM zconfig INTO TABLE @mt_cache.   " expensive load happens once
  ENDMETHOD.
  METHOD get.
    rv = VALUE #( mt_cache[ key = iv_key ]-value OPTIONAL ).
  ENDMETHOD.
ENDCLASS.

" Every caller shares the one instance:
DATA(lv) = zcl_config=>get_instance( )->get( 'TAX_RATE' ).
```
**Why this is better:**
- `CREATE PRIVATE` makes a second instance impossible.
- The expensive load runs once; all callers see one consistent cache.

**Step-by-Step Explanation:**
- `CREATE PRIVATE` + private `constructor` — only the class can instantiate itself.
- `CLASS-DATA go_instance` — the single shared reference (Topic 2.2).
- `get_instance( )` — lazy create-once, then return the same object.

---

## 6. Code Example 2: Real-World Application (and why DI is often better)

**Business Scenario:** The same cache, but designed so it can still be *tested* and *substituted* — addressing the singleton's biggest weakness by hiding it behind an interface and allowing injection.

```abap
INTERFACE lif_config.
  METHODS get IMPORTING iv_key TYPE string RETURNING VALUE(rv) TYPE string.
ENDINTERFACE.

CLASS zcl_config DEFINITION PUBLIC CREATE PRIVATE.
  PUBLIC SECTION.
    INTERFACES lif_config.
    CLASS-METHODS get_instance RETURNING VALUE(ro) TYPE REF TO lif_config.   " returns the ABSTRACTION
    CLASS-METHODS inject FOR TESTING IMPORTING io TYPE REF TO lif_config.    " test seam
  PRIVATE SECTION.
    CLASS-DATA go_instance TYPE REF TO lif_config.
    METHODS constructor.
    DATA mt_cache TYPE SORTED TABLE OF zconfig WITH UNIQUE KEY key.
ENDCLASS.
CLASS zcl_config IMPLEMENTATION.
  METHOD get_instance.
    IF go_instance IS NOT BOUND.
      go_instance = NEW zcl_config( ).
    ENDIF.
    ro = go_instance.
  ENDMETHOD.
  METHOD inject.
    go_instance = io.            " let tests swap in a fake config
  ENDMETHOD.
  METHOD constructor.
    SELECT * FROM zconfig INTO TABLE @mt_cache.
  ENDMETHOD.
  METHOD lif_config~get.
    rv = VALUE #( mt_cache[ key = iv_key ]-value OPTIONAL ).
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **`get_instance` returns `lif_config`** — callers depend on the abstraction, not the concrete singleton, so a fake can replace it.
- **`inject`** — a deliberate test seam so unit tests aren't stuck with the real (DB-loading) instance.
- **Still `CREATE PRIVATE`** — production code can't make a second one.

**How This Works in Practice.** This tames the singleton's worst trait (untestable global state) while keeping "one per session." But note: a plain singleton accessed as `zcl_config=>get_instance( )` deep inside business code is *hidden* coupling. Prefer **injecting** a single shared instance from a composition root (Topic 3.5) and reserving the singleton accessor for true cross-cutting infrastructure.

**Why This Implementation.** Returning an interface and providing an injection seam converts a rigid global into something substitutable — the pragmatic ABAP compromise.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Calling `get_instance` deep inside business logic.**
```abap
METHOD calculate.
  DATA(lv_rate) = zcl_config=>get_instance( )->get( 'TAX' ).   " hidden global dependency
ENDMETHOD.
```
**Why this is wrong:** the dependency is invisible in the method/constructor signature, can't be substituted, and makes the method untestable without the real singleton.
**Correct approach:** inject `lif_config` via the constructor; the *caller* decides whether to pass the shared instance.

**Mistake #2: Forgetting singletons are session-scoped, not global.**
```abap
" Assuming zcl_config=>get_instance( ) is shared across users / work processes — it is NOT.
```
**Why this is wrong:** an ABAP singleton is one instance per *internal session*; other users, sessions, and work processes each have their own.
**Correct approach:** for cross-session sharing use Shared Objects (shared memory); a static singleton won't do it.

---

## 8. Comparison With Similar Concepts

**Singleton vs static-only class:** a singleton is a real object (can implement interfaces, be substituted, hold instance state); an all-static "utility class" cannot be mocked or polymorphic. Prefer a singleton (or injected instance) when substitution matters.

**Singleton vs Dependency Injection (Topic 3.5):** the singleton makes the *class* responsible for its uniqueness and is accessed globally; DI keeps the class normal and lets the composition root create one shared instance and inject it. DI is usually the cleaner way to get "one shared instance" without global access.

**Singleton vs Shared Objects (shared memory):** a singleton shares within one session; Shared Objects share across sessions/work processes on an application server. Different scope entirely.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Factories (Topic 3.3):** a factory may itself be a singleton, or return singletons.
- **`FRIENDS` (Topic 2.11):** `CREATE PRIVATE` is the basis of both singleton and friend-factory patterns.
- **ABAP Unit (Topic 3.6):** singletons are notoriously test-hostile; an `inject`/reset seam is the standard mitigation.

**SAP-Specific Considerations:** static state (and therefore the singleton) is reset when the internal session ends; it does not persist to the next dialog step in stateless scenarios the way you might expect, and is **not** shared across users. Reset/inject in test `setup` to prevent state leaking between tests (Topic 2.2).

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the singleton as a dumping ground of global state.**
```abap
zcl_globals=>get_instance( )->mv_user = sy-uname.   " mutable global state via a singleton
zcl_globals=>get_instance( )->mv_mode = 'X'.
```
**Why this fails:** it is global mutable state with extra steps — the very thing OOP set out to avoid (Topic 1.1), now hidden behind a pattern name.
**Correct approach:** pass needed values explicitly; keep singletons for genuine single-resource services, not a grab-bag.

**Common Gotcha:** because a singleton survives for the whole session, stale cached data can persist longer than intended. Provide a `refresh( )`/reset method when the underlying data can change within a session.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you prove a second instance cannot be created, and can a test still run without the real (DB-loading) instance? If the singleton is untestable, add an injection seam or switch to DI.

**Unit Test Example:**
```abap
CLASS ltd_config DEFINITION.
  PUBLIC SECTION. INTERFACES lif_config.
ENDCLASS.
CLASS ltd_config IMPLEMENTATION.
  METHOD lif_config~get.
    rv = '0.19'.                 " canned config, no database
  ENDMETHOD.
ENDCLASS.

CLASS ltcl_config DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS setup.
    METHODS injected_config_is_used FOR TESTING.
ENDCLASS.
CLASS ltcl_config IMPLEMENTATION.
  METHOD setup.
    zcl_config=>inject( NEW ltd_config( ) ).      " swap real singleton for a fake
  ENDMETHOD.
  METHOD injected_config_is_used.
    cl_abap_unit_assert=>assert_equals(
      act = zcl_config=>get_instance( )->get( 'TAX' ) exp = '0.19' ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the injection seam lets the test replace the DB-loading singleton with a fake, demonstrating the standard cure for the singleton's untestability.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- ABAP singleton = `CREATE PRIVATE` + private static `go_instance` + public static `get_instance( )` (lazy, once).
- Scope is the internal session — not global, not cross-user, not cross-work-process.
- Singletons are global state in disguise; prefer injecting one shared instance (DI), and always provide a test seam.

**When to Apply:** a genuine single per-session resource (cache/registry) — sparingly, behind an interface.

**Red Flags:** `get_instance( )` calls buried in business logic; mutable global state via a singleton; assuming cross-session scope; no way to reset/inject for tests.

---

## 13. Dependency Map

**Depends On:**
- `02_02_attributes_instance_static.md` — the private static instance reference.
- `02_04_constructors.md`, `02_11_friends.md` — `CREATE PRIVATE` controlled construction.

**Enables:**
- `03_03_factory_pattern.md` — factories are often singletons.
- `03_05_dependency_injection.md` — the preferred alternative for sharing one instance.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "CREATE PRIVATE", "CLASS-DATA", "CLASS-METHODS", "Static Constructor".

**Design Patterns & Best Practices:** Pattern origin: Singleton (Gang of Four). Clean ABAP → prefer dependency injection and avoid global state; use singletons judiciously (`github.com/SAP/styleguides`).
