# Visibility Sections (Public / Protected / Private)

**Learning Objective:** After this topic you can deliberately place each class member in the right visibility section, defaulting to the most restrictive that works, to enforce encapsulation and a small, stable public surface.

**Difficulty Level:** Intermediate
**Time to Master:** 45–60 minutes
**Prerequisites:** `01_03_encapsulation.md`, `02_01_abap_class_syntax.md`
**Official Sources:**
- ABAP Keyword Documentation → *Visibility Sections*, *PUBLIC/PROTECTED/PRIVATE SECTION* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Classes* / *Scope* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A class exposes everything as public "to be flexible." Six months later, half its methods are called from across the codebase — including helpers that were never meant to be public. Now you cannot refactor any of them without breaking callers.

**What Happens WITHOUT This Concept.** An over-large public surface freezes implementation details into a contract. Internal helpers become de-facto APIs; encapsulation (Topic 1.3) erodes; every internal change risks external breakage.

**Why This Matters in SAP.** Global classes are reused widely and transported. A small, intentional public surface is what lets you change internals safely across the landscape.

---

## 3. Core Concept Explanation

**Definition.** A class has three visibility sections controlling who can access each member:
- **`PUBLIC SECTION`** — accessible by everyone (the contract).
- **`PROTECTED SECTION`** — accessible by the class and its subclasses only.
- **`PRIVATE SECTION`** — accessible only inside the class itself.

**Key Principles:**
- Default to **private**; promote to protected/public only when a real need exists.
- Public = the stable contract; keep it minimal.
- Protected = the extension surface for subclasses (Topic 1.4).
- Section order is fixed: `PUBLIC` → `PROTECTED` → `PRIVATE` (each at most once).

**How It Works.** The compiler enforces visibility at every access site. A private member referenced from outside is a compile error; a protected member is reachable from a subclass but not from unrelated callers.

**Why It's Designed This Way.** Visibility is the language-level lever for information hiding: it lets you publish *only* what callers should depend on, keeping the rest free to change.

---

## 4. Visual Representation

```
   CLASS zcl_x
   ┌──────────────────────────────────────────────┐
   │ PUBLIC SECTION     ◄── everyone (the contract) │
   │ PROTECTED SECTION  ◄── this class + subclasses │
   │ PRIVATE SECTION    ◄── this class only         │
   └──────────────────────────────────────────────┘

   Access reach:
     outside caller   →  PUBLIC only
     subclass         →  PUBLIC + PROTECTED
     the class itself →  PUBLIC + PROTECTED + PRIVATE
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** A small public API over private internals.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS lcl_calc DEFINITION.
  PUBLIC SECTION.
    DATA mt_buffer TYPE string_table.    " internal buffer exposed
    METHODS round_internal.              " helper exposed
    METHODS total RETURNING VALUE(r) TYPE i.
ENDCLASS.
```
**Problems with this code:**
- The buffer and helper become part of the contract; callers may depend on them.
- You cannot change rounding or buffering without risking external breakage.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_calc DEFINITION.
  PUBLIC SECTION.
    METHODS total RETURNING VALUE(r) TYPE i.    " minimal contract
  PRIVATE SECTION.
    DATA mt_buffer TYPE string_table.            " hidden
    METHODS round_internal.                      " hidden helper
ENDCLASS.
```
**Why this is better:**
- The public surface is one method; everything else is free to change.
- Encapsulation is intact; refactoring internals cannot break callers.

**Step-by-Step Explanation:**
- `PUBLIC SECTION … total` — the only thing callers may rely on.
- `PRIVATE SECTION … mt_buffer / round_internal` — implementation details, invisible outside.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A base pricing class designed for extension exposes a *protected* hook for subclasses while keeping the calculation engine private and the entry point public.

```abap
CLASS zcl_pricing DEFINITION PUBLIC CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING iv_base TYPE p LENGTH 13 DECIMALS 2.
    METHODS price RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.   " contract
  PROTECTED SECTION.
    " extension point: subclasses may adjust, but outside callers cannot
    METHODS surcharge RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
    DATA mv_base TYPE p LENGTH 13 DECIMALS 2.
  PRIVATE SECTION.
    METHODS apply_tax IMPORTING iv_amount TYPE p LENGTH 13 DECIMALS 2
                      RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.  " hidden engine
ENDCLASS.

CLASS zcl_pricing IMPLEMENTATION.
  METHOD constructor.
    mv_base = iv_base.
  ENDMETHOD.
  METHOD price.
    rv = apply_tax( mv_base + surcharge( ) ).
  ENDMETHOD.
  METHOD surcharge.
    rv = 0.                       " default: no surcharge
  ENDMETHOD.
  METHOD apply_tax.
    rv = iv_amount * '1.19'.
  ENDMETHOD.
ENDCLASS.

CLASS zcl_service_pricing DEFINITION INHERITING FROM zcl_pricing.
  PROTECTED SECTION.
    METHODS surcharge REDEFINITION.    " allowed: protected hook
ENDCLASS.
CLASS zcl_service_pricing IMPLEMENTATION.
  METHOD surcharge.
    rv = 50.                          " specialize via the protected hook
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **Public `price`** — the only contract callers see.
- **Protected `surcharge`** — a deliberate extension point; subclasses override it, outsiders cannot call it.
- **Private `apply_tax`** — the tax engine stays sealed; not even subclasses depend on it.

**How This Works in Practice.** Subclasses customize behaviour through the *intended* protected hook, not by reaching into private internals — extension without erosion of encapsulation.

**Why This Implementation.** Visibility encodes design intent: "this is the API," "this is for subclasses," "this is mine." The compiler then enforces it.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Making everything public "for flexibility."**
```abap
PUBLIC SECTION.
  DATA mv_state TYPE i.    " public mutable internal
  METHODS helper.          " public helper nobody outside should call
```
**Why this is wrong:** every public member is a promise; over-exposure freezes internals and breaks encapsulation.
**Correct approach:** start private; expose only what callers must use.

**Mistake #2: Wrong section order.**
```abap
CLASS lcl_x DEFINITION.
  PRIVATE SECTION.
    DATA mv_a TYPE i.
  PUBLIC SECTION.          " wrong order → syntax error
    METHODS m.
ENDCLASS.
```
**Why this is wrong:** ABAP mandates `PUBLIC` → `PROTECTED` → `PRIVATE`.
**Correct approach:** declare sections in that fixed order.

---

## 8. Comparison With Similar Concepts

**Public vs Protected vs Private:** public is for everyone (the contract), protected is for the inheritance family (extension hooks), private is for the class alone (implementation). Choose the most restrictive that still works.

**Visibility vs `FRIENDS` (Topic 2.11):** visibility is the normal, broad rule; `FRIENDS` is a narrow, explicit exception that grants one named class access to private/protected members. Prefer visibility; reach for `FRIENDS` rarely.

**Visibility vs `READ-ONLY`:** `READ-ONLY` keeps a *public* attribute readable everywhere but writable only inside the class — a middle ground for exposing values without a getter.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Global class APIs:** the public section is the documented, reused contract; SAP and customers depend on it.
- **Inheritance / BAdI extension:** protected members are the supported customization points.
- **Unit tests:** private members are not directly testable from outside — test through the public surface (a sign of good design); local test classes can be granted access via `FRIENDS` when necessary.

**SAP-Specific Considerations:** changing a public signature in a transported global class can break consumers landscape-wide — treat the public section as a stable API. Protected members form the *extension* contract for subclasses and BAdI implementations.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: testing private methods by making them public.**
```abap
PUBLIC SECTION.
  METHODS round_internal.   " made public only so a test can call it
```
**Why this fails:** it pollutes the contract to suit tests; the need usually signals the method belongs in its own (testable) class.
**Correct approach:** test behaviour through the public API, or extract the logic into a collaborator you can test directly.

**Common Gotcha:** protected members are visible to *all* subclasses, now and future — they are part of your extension contract, so design them as deliberately as public ones.

---

## 11. Testing & Validation

**How to Verify Understanding:** List your class's public members. Could you delete or rename a private one without touching any other program? If not, something private leaked into the contract.

**Unit Test Example:**
```abap
CLASS ltcl_pricing DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS public_contract_is_enough FOR TESTING.
ENDCLASS.

CLASS ltcl_pricing IMPLEMENTATION.
  METHOD public_contract_is_enough.
    " The test exercises ONLY the public surface — private engine stays hidden:
    DATA(lo) = NEW zcl_service_pricing( iv_base = '100.00' ).
    cl_abap_unit_assert=>assert_equals(
      act = lo->price( ) exp = CONV p( '178.50' ) ).   " (100 + 50) * 1.19
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the behaviour is fully verifiable through the public `price( )`; the private tax engine never needs to be exposed.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Public = contract (minimal), protected = subclass extension surface, private = implementation.
- Default to the most restrictive section that works; promote only on real need.
- Fixed order: `PUBLIC` → `PROTECTED` → `PRIVATE`.

**When to Apply:** every member — decide its visibility deliberately, smallest scope first.

**Red Flags:** large public surfaces; public mutable attributes; widening visibility just to test; private leaking into the contract.

---

## 13. Dependency Map

**Depends On:**
- `01_03_encapsulation.md` — visibility is the mechanism that realizes encapsulation.
- `02_01_abap_class_syntax.md` — sections live in the class definition.

**Enables:**
- `02_07_inheritance_in_abap.md` — protected members as the extension contract.
- `02_11_friends.md` — the controlled exception to visibility.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "Visibility Sections", "PUBLIC SECTION", "PROTECTED SECTION", "PRIVATE SECTION", "READ-ONLY".

**Design Patterns & Best Practices:** Clean ABAP → keep the public surface small; prefer the narrowest scope (`github.com/SAP/styleguides`). Conceptual basis: information hiding (Parnas).
