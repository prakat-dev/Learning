# Encapsulation

**Learning Objective:** After this topic you can hide an object's internal state behind a deliberate, stable interface, and explain why that protects an object from ever entering an invalid state.

**Difficulty Level:** Foundational
**Time to Master:** 60–75 minutes
**Prerequisites:** `01_02_classes_and_objects.md`
**Official Sources:**
- ABAP Keyword Documentation → *Visibility Sections*, *READ-ONLY* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Classes* / *Members* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A bank-account object holds a `balance`. If any code anywhere can write `account->balance = -9999`, you can produce an overdraft that violates a business rule — and you will never find *where* it happened, because dozens of places can do it.

**What Happens WITHOUT This Concept.** Public, writable state means the object cannot defend its own rules. Validation logic (`balance must not go below the overdraft limit`) lives — and is forgotten — at every call site instead of in one place.

**Why This Matters in SAP.** Business rules (credit limits, posting periods, status transitions) must hold *invariantly*. Encapsulation is how an object guarantees its own rules regardless of who calls it.

---

## 3. Core Concept Explanation

**Definition.** **Encapsulation** is hiding an object's internal data and exposing only a controlled set of operations. State changes happen *only* through the object's own methods, which can enforce rules.

**Key Principles:**
- Make attributes **private**; expose behaviour, not data.
- Provide change through **intention-revealing methods** (`deposit`, `withdraw`), not raw setters.
- If outside code must *read* a value but never write it, expose a getter or a `READ-ONLY` attribute.

**How It Works.** Visibility sections (`PUBLIC`/`PROTECTED`/`PRIVATE`) control who can see each member. Private data is reachable only from inside the class. Every legal change must pass through a public method that can validate first.

**Why It's Designed This Way.** If the only door into the state is a method, that method becomes the single place to enforce a rule. The invariant ("balance ≥ limit") is guaranteed by construction, not by everyone remembering to check.

---

## 4. Visual Representation

```
            WITHOUT encapsulation            WITH encapsulation
            (state exposed)                  (state guarded)

   caller ─────► [ balance ]  ◄── caller     caller ──► withdraw(amt) ──┐
   caller ─────►              ◄── caller                                │
        anyone writes anything            ┌──────────────────────────┐ │
        → invalid states possible         │  validate, then change   │◄┘
                                          │  private balance          │
                                          └──────────────────────────┘
                                            only methods touch state
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** An account that must never drop below an overdraft limit.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS lcl_account DEFINITION.
  PUBLIC SECTION.
    DATA mv_balance TYPE p_amount.   " PUBLIC + writable
ENDCLASS.
CLASS lcl_account IMPLEMENTATION.
ENDCLASS.

" Any caller can break the rule:
lo_account->mv_balance = lo_account->mv_balance - 100000.   " overdraft, unchecked
```
**Problems with this code:**
- The rule "never below the limit" cannot be enforced — there is no choke point.
- Bugs are untraceable: any of N call sites could be the culprit.
- The object cannot guarantee its own validity.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_account DEFINITION.
  PUBLIC SECTION.
    METHODS constructor IMPORTING iv_limit TYPE p_amount.
    METHODS deposit  IMPORTING iv_amount TYPE p_amount.
    METHODS withdraw IMPORTING iv_amount TYPE p_amount
                     RAISING   cx_sy_arithmetic_error.   " illustrative
    METHODS balance  RETURNING VALUE(rv_balance) TYPE p_amount.
  PRIVATE SECTION.
    DATA mv_balance TYPE p_amount.   " HIDDEN
    DATA mv_limit   TYPE p_amount.
ENDCLASS.

CLASS lcl_account IMPLEMENTATION.
  METHOD constructor.
    mv_limit = iv_limit.
  ENDMETHOD.
  METHOD deposit.
    mv_balance = mv_balance + iv_amount.
  ENDMETHOD.
  METHOD withdraw.
    IF mv_balance - iv_amount < mv_limit.        " single choke point for the rule
      RAISE EXCEPTION TYPE cx_sy_arithmetic_error.
    ENDIF.
    mv_balance = mv_balance - iv_amount.
  ENDMETHOD.
  METHOD balance.
    rv_balance = mv_balance.                      " read-only access via getter
  ENDMETHOD.
ENDCLASS.
```
**Why this is better:**
- The overdraft rule is enforced in exactly one place; it cannot be bypassed.
- Outsiders can read the balance (via `balance( )`) but never write it directly.
- The object is always in a valid state.

**Step-by-Step Explanation:**
- `PRIVATE SECTION … mv_balance` — the state is sealed inside.
- `withdraw` checks the rule *before* mutating — invariant protection.
- `balance( )` exposes a read path without a write path.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A sales-document object must only allow status transitions that follow a business workflow (`open → released → closed`, never `closed → open`).

```abap
CLASS lcl_sales_doc DEFINITION.
  PUBLIC SECTION.
    TYPES ty_status TYPE c LENGTH 1.    " 'O' open, 'R' released, 'C' closed
    METHODS constructor.
    METHODS release.
    METHODS close.
    METHODS status RETURNING VALUE(rv_status) TYPE ty_status.
  PRIVATE SECTION.
    DATA mv_status TYPE ty_status.
    METHODS is_transition_allowed IMPORTING iv_to TYPE ty_status
                                  RETURNING VALUE(rv_ok) TYPE abap_bool.
ENDCLASS.

CLASS lcl_sales_doc IMPLEMENTATION.
  METHOD constructor.
    mv_status = 'O'.                                   " new docs start open
  ENDMETHOD.

  METHOD release.
    IF is_transition_allowed( 'R' ) = abap_true.
      mv_status = 'R'.
    ENDIF.
  ENDMETHOD.

  METHOD close.
    IF is_transition_allowed( 'C' ) = abap_true.
      mv_status = 'C'.
    ENDIF.
  ENDMETHOD.

  METHOD status.
    rv_status = mv_status.
  ENDMETHOD.

  METHOD is_transition_allowed.
    " encapsulated rule table: from current status, which targets are legal
    rv_ok = xsdbool(
      ( mv_status = 'O' AND iv_to = 'R' ) OR
      ( mv_status = 'R' AND iv_to = 'C' ) ).
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **Private `mv_status`** — no caller can jump straight to `'C'`.
- **`is_transition_allowed`** — a *private helper* that centralizes the legal-transitions rule; it is implementation detail, hidden from callers.
- **Public `release`/`close`** — the only doors, and each consults the rule first.

**How This Works in Practice.** A caller does `lo_doc->release( ).` then `lo_doc->close( ).`; an attempt to `close` an `open` document simply does nothing illegal because the transition is not allowed.

**Why This Implementation.** The workflow rule lives once, inside the object. Changing the allowed transitions means editing one private method — no call site changes.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: "Encapsulation" that is just public getters and setters for every field.**
```abap
METHODS set_status IMPORTING iv_status TYPE ty_status.   " plain setter
METHOD set_status. mv_status = iv_status. ENDMETHOD.       " no rule enforced
```
**Why this is wrong:** a setter that blindly assigns is a public field with extra typing — it enforces nothing.
**Correct approach:** expose *intent* (`release`, `close`) that validates, not a raw `set_status`.

**Mistake #2: Leaking a mutable internal reference.**
```abap
METHODS get_items RETURNING VALUE(rt_items) TYPE REF TO tt_items.
METHOD get_items. rt_items = REF #( mt_items ). ENDMETHOD.   " caller can now mutate internals
```
**Why this is wrong:** handing out a reference to internal data lets outsiders modify it, breaking encapsulation.
**Correct approach:** return a *copy* (by value), or expose specific operations instead of the raw table.

---

## 8. Comparison With Similar Concepts

**Encapsulation vs Abstraction:** abstraction is *what* you expose (the simplified interface concept — Topic 1.6); encapsulation is *hiding the how* and guarding the data behind it. They are complementary: abstraction decides the surface, encapsulation protects what is below it.

**Encapsulation vs `PRIVATE` keyword:** `PRIVATE` is the *mechanism*; encapsulation is the *design goal*. You can use `PRIVATE` and still fail at encapsulation (e.g. by leaking references, as above).

**Getter vs intention-revealing method:** `balance( )` (read) is a legitimate getter; `set_balance( )` (blind write) usually is not — prefer `deposit`/`withdraw` that carry meaning and enforce rules.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Business rules / Customizing:** rule checks (limits, periods, statuses) belong inside the owning object's methods.
- **Authorization:** method boundaries are natural places for `AUTHORITY-CHECK`, since all access funnels through them.
- **RAP:** behaviour definitions enforce validations centrally — the same encapsulation idea at framework scale.

**SAP-Specific Considerations:** returning large internal tables by value costs a copy; for big data, expose targeted query methods rather than the whole table. `READ-ONLY` public attributes (ABAP) give cheap read access without a getter method when no logic is needed on read.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the "anemic" object** — all public data, no behaviour, with logic living in separate "manager" classes.
```abap
CLASS lcl_account DEFINITION.
  PUBLIC SECTION.
    DATA balance TYPE p_amount.   " data only; rules live elsewhere
ENDCLASS.
```
**Why this fails:** the object cannot protect its invariants; you are back to procedural code with scattered rules.
**Correct approach:** move the behaviour that uses the data *into* the object.

**Common Gotcha:** ABAP `READ-ONLY` makes a public attribute readable everywhere but writable only inside the class — handy, but it still exposes the data shape. Prefer a method when the read should compute or validate.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can an outside caller put your object into an invalid state? If yes, encapsulation is incomplete.

**Unit Test Example:**
```abap
CLASS ltcl_account DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS overdraft_is_blocked FOR TESTING.
ENDCLASS.

CLASS ltcl_account IMPLEMENTATION.
  METHOD overdraft_is_blocked.
    DATA(lo) = NEW lcl_account( iv_limit = 0 ).
    lo->deposit( 100 ).

    TRY.
        lo->withdraw( 500 ).                 " violates limit → should raise
        cl_abap_unit_assert=>fail( 'overdraft should have been blocked' ).
      CATCH cx_sy_arithmetic_error.
        cl_abap_unit_assert=>assert_equals( act = lo->balance( ) exp = CONV p_amount( 100 ) ).
    ENDTRY.
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the test proves the invariant holds — the balance is unchanged after an illegal withdrawal, because the rule is enforced inside the object.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Hide state (`PRIVATE`); change it only through methods that can enforce rules.
- Expose *intent* and *reads*, not raw writable fields.
- Encapsulation guarantees invariants by construction — the rule has exactly one home.

**When to Apply:** whenever an object has rules about valid state (almost always).

**Red Flags:** public writable attributes; blind setters; returning references to internal data; rules duplicated at call sites.

---

## 13. Dependency Map

**Depends On:** `01_02_classes_and_objects.md` — you must have instance state before you can hide it.

**Enables:**
- `01_04_inheritance.md` — `PROTECTED` visibility is meaningful only once encapsulation is understood.
- `02_05_visibility_sections.md` — the full visibility mechanics.
- `03_01_solid_principles.md` — encapsulation underpins the Single-Responsibility and Open–Closed principles.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "Visibility Sections", "READ-ONLY", and "Functional Methods".

**Design Patterns & Best Practices:** Clean ABAP → prefer immutability / minimal public surface; avoid exposing internals (`github.com/SAP/styleguides`). Conceptual basis: information hiding (Parnas) as adapted in *Clean Code*.
