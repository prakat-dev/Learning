# FRIENDS (Controlled Access to Private/Protected Members)

**Learning Objective:** After this topic you can use the `FRIENDS` addition to grant one class privileged access to another's private and protected members, apply the `CREATE PRIVATE` + factory-friend pattern, and recognize why friendship should be rare and deliberate.

**Difficulty Level:** Advanced
**Time to Master:** 50–60 minutes
**Prerequisites:** `02_05_visibility_sections.md`, `02_04_constructors.md`
**Official Sources:**
- ABAP Keyword Documentation → *FRIENDS*, *CREATE PRIVATE*, *Friendship* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Object Orientation* / *Scope* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** You want a class that can be created *only* through its factory — no one may call `NEW` on it directly. But the factory itself must be able to instantiate it. The factory therefore needs access the rest of the world is denied: it needs to be a *friend*.

**What Happens WITHOUT This Concept.** To let the factory in, you would have to make the constructor public — which reopens direct instantiation to everybody, defeating the point. Without controlled friendship you cannot grant one trusted collaborator special access while keeping everyone else out.

**Why This Matters in SAP.** Two everyday uses: (1) a factory that must instantiate a `CREATE PRIVATE` class, and (2) a unit-test class that must reach a production class's internals. `FRIENDS` makes both possible *without* widening the public surface for everyone else.

---

## 3. Core Concept Explanation

**Definition.** The **`FRIENDS`** addition names other classes (or interfaces) that are granted full access to *all* components of the befriending class — including its `PRIVATE` and `PROTECTED` members — and the right to instantiate it even when it is `CREATE PRIVATE`/`CREATE PROTECTED`.

**Key Principles:**
- Friendship is **granted by** the class that exposes itself (`… DEFINITION … FRIENDS other_class`).
- Friendship is **one-directional**: if A befriends B, B can access A — not automatically the reverse.
- Friendship is **not inherited** in the way normal members are; subclasses of a friend are friends only if friendship is granted to that hierarchy (`… FRIENDS` of a class extends to its subclasses, but befriending does not flow to the friend's *unrelated* classes).
- Use it sparingly — it deliberately breaks encapsulation for a named, trusted party.

**How It Works.** When class `A` is declared `… FRIENDS B`, the compiler permits code in `B` to read/write `A`'s private and protected attributes, call its private methods, and create instances of `A` even under `CREATE PRIVATE`. Everyone else still sees only `A`'s public surface.

**Why It's Designed This Way.** Sometimes one specific collaborator legitimately needs inside access (a factory, a test). Rather than forcing you to make those members public to all, `FRIENDS` opens a narrow, explicit, named door.

---

## 4. Visual Representation

```
   CLASS zcl_account DEFINITION CREATE PRIVATE      (nobody may NEW it...)
     FRIENDS zcl_account_factory.                   (...except this named friend)
   ┌─────────────────────────────────────────────┐
   │ PRIVATE: constructor, mv_balance             │◄── factory may reach these
   └─────────────────────────────────────────────┘
            ▲ friendship is ONE-WAY
            │  (factory → account;  account does NOT get into factory)
   ┌─────────────────────────────┐
   │ zcl_account_factory         │  the only code that can create accounts
   │  create( ) → NEW zcl_account│
   └─────────────────────────────┘

   Everyone else:  sees only zcl_account's PUBLIC members, cannot NEW it.
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** A class whose constructor only its factory may call.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS zcl_account DEFINITION PUBLIC CREATE PUBLIC.   " anyone can NEW it
  PUBLIC SECTION.
    METHODS constructor IMPORTING iv_balance TYPE p.  " public constructor
ENDCLASS.
" The "factory" is pointless — callers just bypass it:
DATA(lo) = NEW zcl_account( iv_balance = -999 ).      " creates an invalid account directly
```
**Problems with this code:**
- `CREATE PUBLIC` + public constructor means the factory guarantees nothing; anyone bypasses it.
- Invariants the factory would enforce can be skipped.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS zcl_account DEFINITION PUBLIC CREATE PRIVATE   " no outside NEW
    FRIENDS zcl_account_factory.                     " ...except the factory
  PUBLIC SECTION.
    METHODS balance RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
  PRIVATE SECTION.
    METHODS constructor IMPORTING iv_balance TYPE p LENGTH 13 DECIMALS 2.
    DATA mv_balance TYPE p LENGTH 13 DECIMALS 2.
ENDCLASS.
CLASS zcl_account IMPLEMENTATION.
  METHOD constructor.
    mv_balance = iv_balance.
  ENDMETHOD.
  METHOD balance.
    rv = mv_balance.
  ENDMETHOD.
ENDCLASS.

CLASS zcl_account_factory DEFINITION PUBLIC CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS create IMPORTING iv_balance TYPE p LENGTH 13 DECIMALS 2
                   RETURNING VALUE(ro)  TYPE REF TO zcl_account
                   RAISING   cx_sy_create_object_error.
ENDCLASS.
CLASS zcl_account_factory IMPLEMENTATION.
  METHOD create.
    IF iv_balance < 0.
      RAISE EXCEPTION TYPE cx_sy_create_object_error.   " enforce the rule
    ENDIF.
    ro = NEW zcl_account( iv_balance ).   " allowed ONLY because factory is a friend
  ENDMETHOD.
ENDCLASS.
```
**Why this is better:**
- `CREATE PRIVATE` blocks direct `NEW`; only the friend factory can instantiate.
- The factory's validation cannot be bypassed — every account is created through it.

**Step-by-Step Explanation:**
- `CREATE PRIVATE FRIENDS zcl_account_factory` — closes instantiation to all except the named friend.
- The constructor is `PRIVATE` — unreachable from outside, reachable by the friend.
- `NEW zcl_account( … )` inside the factory compiles *only* because of the friendship grant.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A production class needs no public test hooks, yet its unit tests must inspect private state. The class befriends its local test class.

```abap
" Production global class — note it befriends its own (local) test class:
CLASS zcl_ledger DEFINITION PUBLIC CREATE PUBLIC
    FRIENDS ltcl_ledger.                 " grant the test class inside access
  PUBLIC SECTION.
    METHODS post IMPORTING iv_amount TYPE p LENGTH 13 DECIMALS 2.
  PRIVATE SECTION.
    DATA mt_entries TYPE STANDARD TABLE OF p.   " internal, not exposed publicly
ENDCLASS.
CLASS zcl_ledger IMPLEMENTATION.
  METHOD post.
    APPEND iv_amount TO mt_entries.
  ENDMETHOD.
ENDCLASS.

" Local test class (in the test include) — declared as the friend:
CLASS ltcl_ledger DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS post_appends_entry FOR TESTING.
ENDCLASS.
CLASS ltcl_ledger IMPLEMENTATION.
  METHOD post_appends_entry.
    DATA(lo) = NEW zcl_ledger( ).
    lo->post( '100.00' ).

    " Friendship lets the test read the PRIVATE table directly:
    cl_abap_unit_assert=>assert_equals( act = lines( lo->mt_entries ) exp = 1 ).
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **`FRIENDS ltcl_ledger`** — the production class names its test class as a friend.
- **`lo->mt_entries`** in the test — direct access to a `PRIVATE` attribute, permitted only by the friendship.
- **No public test hooks** — the production contract stays clean; the test reaches in without widening visibility for everyone (contrast Topic 2.5's anti-pattern of making methods public just to test them).

**How This Works in Practice.** Local-friend test classes are a common ABAP technique for white-box testing legacy or stateful classes — but prefer testing through the public API when the design allows it.

**Why This Implementation.** Friendship confines the encapsulation break to exactly one trusted collaborator (here, the tests) rather than exposing internals to the whole system.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Expecting friendship to be mutual.**
```abap
CLASS a DEFINITION FRIENDS b.   " A lets B in
" ...then code in A tries to touch B's private members → NOT allowed
```
**Why this is wrong:** friendship is one-directional. `A FRIENDS B` lets **B** access **A**, not the reverse.
**Correct approach:** if both directions are truly needed, each class must grant friendship explicitly — but reconsider the design first.

**Mistake #2: Using `FRIENDS` to avoid designing a proper interface.**
```abap
CLASS zcl_service DEFINITION FRIENDS zcl_consumer_a zcl_consumer_b zcl_consumer_c.
" several "friends" reaching into internals instead of calling a clean API
```
**Why this is wrong:** many friends reaching into private state is global coupling in disguise; it defeats encapsulation broadly.
**Correct approach:** expose the needed behaviour through public methods or an interface; reserve `FRIENDS` for the rare factory/test case.

---

## 8. Comparison With Similar Concepts

**`FRIENDS` vs visibility sections (Topic 2.5):** visibility is the broad, default rule (public/protected/private for *everyone* in that category); `FRIENDS` is a narrow, named exception that grants one specific class full inside access. Use visibility first; `FRIENDS` only when a named collaborator genuinely needs more.

**`FRIENDS` vs `PROTECTED`:** protected opens members to *all* subclasses (an open-ended family); `FRIENDS` opens *everything* to a *named* class that need not be related by inheritance. Different axis (named party vs subclass family) and different breadth (all members vs protected only).

**`CREATE PRIVATE` + `FRIENDS` vs public constructor:** the former guarantees creation goes through the factory (enforced invariants); the latter lets anyone bypass it. Use `CREATE PRIVATE` + friend factory when controlled construction matters (Topics 3.2, 3.3).

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Factory pattern (Topic 3.3) & Singleton (Topic 3.2):** `CREATE PRIVATE` + a friend factory/`class_constructor` is the canonical controlled-instantiation idiom.
- **ABAP Unit (Topic 3.6):** befriending the local test class enables white-box tests without public hooks; the ADT "local test classes" test include is the usual home.
- **SAP standard classes:** some SAP classes use friendship internally between tightly-coupled cooperating classes (e.g. a manager and its handle).

**SAP-Specific Considerations:** local friends (a class and its local test/helper classes) are common and low-risk; global-to-global friendship couples two transportable objects tightly, so use it only for genuinely cooperating pairs. Friendship granted to a class also applies to that class's subclasses.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: friendship as a convenience shortcut.**
```abap
" "It was easier to just befriend it than add a getter."
CLASS zcl_order DEFINITION FRIENDS zcl_report_helper.
```
**Why this fails:** it normalizes reaching into internals; over time many classes poke at each other's private state and encapsulation collapses.
**Correct approach:** add the needed public method/interface; friendship is for the few cases where a clean API genuinely cannot serve (factory, test).

**Common Gotcha:** friendship grants access to **all** components at once — there is no "friend for this one attribute." It is an all-or-nothing inside pass, which is exactly why it must be reserved for trusted, tightly-coupled collaborators.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you justify each `FRIENDS` grant in one sentence ("the factory must instantiate a `CREATE PRIVATE` class"; "the local test class inspects private state")? If a grant has no such justification, remove it and use the public API.

**Unit Test Example:**
```abap
CLASS ltcl_factory DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS factory_is_only_creator FOR TESTING.
ENDCLASS.

CLASS ltcl_factory IMPLEMENTATION.
  METHOD factory_is_only_creator.
    " Creation via the friend factory succeeds and enforces the rule:
    TRY.
        DATA(lo_ok) = NEW zcl_account_factory( )->create( iv_balance = '100.00' ).
        cl_abap_unit_assert=>assert_equals( act = lo_ok->balance( ) exp = CONV p( '100.00' ) ).
      CATCH cx_sy_create_object_error.
        cl_abap_unit_assert=>fail( 'valid balance should succeed' ).
    ENDTRY.

    " Negative balance is rejected by the factory's enforced invariant:
    TRY.
        NEW zcl_account_factory( )->create( iv_balance = '-1.00' ).
        cl_abap_unit_assert=>fail( 'negative balance should be rejected' ).
      CATCH cx_sy_create_object_error.
        " expected
    ENDTRY.
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** because `zcl_account` is `CREATE PRIVATE` and only the factory is its friend, every account must pass through the factory's validation — the test confirms both the success and the rejection paths.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- `FRIENDS` grants one named class full access to another's private/protected members and the right to instantiate it under `CREATE PRIVATE`/`PROTECTED`.
- Friendship is one-directional and all-or-nothing; it deliberately breaks encapsulation for a trusted party.
- Canonical uses: factory creating a `CREATE PRIVATE` class; local test class inspecting internals. Use sparingly.

**When to Apply:** controlled instantiation (factory/singleton) and white-box unit testing — almost nowhere else.

**Red Flags:** expecting mutual friendship; many friends on one class; friendship used to skip designing a clean API.

---

## 13. Dependency Map

**Depends On:**
- `02_05_visibility_sections.md` — `FRIENDS` is the deliberate exception to visibility.
- `02_04_constructors.md` — friendship enables calling a private constructor.

**Enables:**
- `03_02_singleton_pattern.md` — `CREATE PRIVATE` + controlled instantiation.
- `03_03_factory_pattern.md` — friend factory creating an otherwise-uninstantiable class.
- `03_06_abap_unit_and_test_doubles.md` — local-friend test classes.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "FRIENDS", "CREATE PRIVATE", "Friendship", "CLASS - DEFINITION (friends)".

**Design Patterns & Best Practices:** Clean ABAP → keep scope tight; prefer composition and clean APIs over reaching into internals (`github.com/SAP/styleguides`).
