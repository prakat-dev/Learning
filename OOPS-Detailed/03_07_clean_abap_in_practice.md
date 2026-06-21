# Clean ABAP in Practice

**Learning Objective:** After this topic you can apply the Clean ABAP style guide as a daily practice — naming, small methods and classes, modern syntax, composition over inheritance, sound error handling, and tests — and use tooling (abaplint, Code Pal) to enforce it, tying together everything in this curriculum.

**Difficulty Level:** Advanced
**Time to Master:** 90 minutes
**Prerequisites:** All of Module 02; `03_01_solid_principles.md` through `03_06_abap_unit_and_test_doubles.md`
**Official Sources:**
- Clean ABAP style guide (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)
- SAP Code Pal for ABAP (`github.com/SAP/code-pal-for-abap`)
- ABAP Keyword Documentation (`help.sap.com/doc/abapdocu_latest_index_htm`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** Two teams write OO ABAP. One follows a shared style — small classes, clear names, modern syntax, tests; the other improvises. Six months on, the first codebase is changed confidently in minutes; the second is a thicket nobody dares touch. The language was identical; the *discipline* was not.

**What Happens WITHOUT This Concept.** Knowing OOP mechanics (Modules 01–02) and patterns (3.1–3.6) isn't enough; without consistent everyday practice, code still drifts into long methods, cryptic names, mixed old/new syntax, and silent error handling.

**Why This Matters in SAP.** Clean ABAP is SAP's community-and-company-endorsed style guide. It converts the principles in this curriculum into concrete, reviewable habits, and tooling can enforce them — so quality survives team turnover and time.

---

## 3. Core Concept Explanation

**Definition.** **Clean ABAP** is a style guide of concrete recommendations for readable, maintainable ABAP, oriented toward modern object-oriented code. It is opinionated, example-driven, and enforceable by static analysis.

**Key Themes (selected):**
- **Names:** reveal intent; avoid encodings/Hungarian noise; method names are verbs, classes are nouns.
- **Methods:** small, do one thing, few parameters, prefer `RETURNING` (functional); avoid boolean/flag parameters.
- **Classes:** small and cohesive; `FINAL` by default; prefer composition over inheritance; keep the public surface minimal.
- **Modern syntax:** `NEW` over `CREATE OBJECT`; inline `DATA(...)`; `VALUE`/`COND`/`SWITCH`/`REDUCE`; string templates.
- **Error handling:** class-based exceptions; don't catch everything; fail fast.
- **Tests:** unit-test behaviour; isolate with doubles.

**How It Works.** You apply the guidelines while writing and in code review; tools (abaplint, ABAP Code Pal, ATC) flag deviations automatically. The guide is a reference you consult and cite, not a one-time read.

**Why It's Designed This Way.** Consistency is what makes a large codebase legible to many people over years. A shared, tool-enforced style removes bikeshedding and keeps the OO designs in this curriculum from eroding.

---

## 4. Visual Representation

```
   THE CURRICULUM, ROLLED UP                ENFORCEMENT LOOP
   ─────────────────────────                ────────────────
   Tier 1  why + pillars      ┐             write  ─► abaplint / Code Pal / ATC
   Tier 2  ABAP mechanics     ├─► Clean        ▲                     │ flags issues
   Tier 3  SOLID + patterns   ┘   ABAP         └──── fix / review ◄──┘
                                  (daily habits)        + unit tests prove behaviour
```

---

## 5. Code Example 1: Basic Concept (names, modern syntax, functional method)

**Scenario:** The same logic, improvised vs. clean.

**The WRONG Way (Anti-Pattern):**
```abap
DATA lt_t TYPE STANDARD TABLE OF vbak.
DATA wa LIKE LINE OF lt_t.
DATA v_x TYPE p DECIMALS 2.
CREATE OBJECT go_obj.                         " legacy instantiation
SELECT * FROM vbak INTO TABLE lt_t WHERE kunnr = p_k.
LOOP AT lt_t INTO wa.
  v_x = v_x + wa-netwr.                        " cryptic names, manual loop
ENDLOOP.
go_obj->do( v_x ).                             " "do" tells you nothing
```
**Problems with this code:**
- Cryptic names (`v_x`, `do`), legacy `CREATE OBJECT`, verbose manual aggregation.
- Intent is hidden; hard to read or change.

**The RIGHT Way (Following Best Practice):**
```abap
DATA(lt_orders) = NEW zcl_order_repo( )->read_for_customer( p_customer ).

DATA(lv_total) = REDUCE netwr( INIT sum = 0
                               FOR ls IN lt_orders
                               NEXT sum = sum + ls-netwr ).   " expressive aggregation

NEW zcl_dunning( )->raise_reminder( lv_total ).               " name states intent
```
**Why this is better:**
- Names reveal intent (`lt_orders`, `lv_total`, `raise_reminder`); `NEW` and `REDUCE` are concise and modern.
- Fewer moving parts; the *what* is obvious.

**Step-by-Step Explanation:**
- `NEW …->read_for_customer( … )` — `NEW` over `CREATE OBJECT`; a clearly named method.
- `REDUCE` — declarative aggregation instead of a manual `LOOP`/accumulator.
- `raise_reminder( )` — a verb naming the action precisely.

---

## 6. Code Example 2: Real-World Application (refactor toward Clean ABAP)

**Business Scenario:** Refactor a flag-driven, do-everything method into small, intention-revealing methods with composition and exceptions — the cumulative lesson of the curriculum.

```abap
" BEFORE: one method, a flag, mixed concerns, return code
METHODS process IMPORTING iv_mode TYPE c RETURNING VALUE(rv_subrc) TYPE sysubrc.

" AFTER: small cohesive methods; behaviour split; exceptions; composition
CLASS zcl_invoice DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING io_tax TYPE REF TO lif_tax.   " inject (Topic 3.5)
    METHODS gross_amount RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
    METHODS post RAISING zcx_posting_failed.                    " exception, not subrc
  PRIVATE SECTION.
    DATA mo_tax TYPE REF TO lif_tax.
    METHODS net_amount RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
ENDCLASS.
CLASS zcl_invoice IMPLEMENTATION.
  METHOD constructor.
    mo_tax = io_tax.
  ENDMETHOD.
  METHOD net_amount.
    rv = '100.00'.                       " (abbreviated)
  ENDMETHOD.
  METHOD gross_amount.                    " functional, one job
    rv = net_amount( ) + mo_tax->calc( net_amount( ) ).
  ENDMETHOD.
  METHOD post.                            " one job; fails loudly
    IF gross_amount( ) <= 0.
      RAISE EXCEPTION NEW zcx_posting_failed( ).
    ENDIF.
    " ...post...
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **Flag parameter removed** — `process( mode )` became distinct, named methods (`gross_amount`, `post`).
- **Injected tax strategy** — composition + DIP instead of an internal `CASE`/`NEW`.
- **Exception over `sysubrc`** — failure is explicit and unignorable (Topic 2.10).

**How This Works in Practice.** Clean ABAP refactoring is incremental: rename for intent, split methods that do more than one thing, replace flags with separate methods or strategies, swap return codes for exceptions, add tests as you go. Each step is small and test-protected (Topic 3.6).

**Why This Implementation.** The result is the curriculum in miniature — encapsulation, interfaces, injection, polymorphism, exceptions, and tests — expressed as ordinary, readable code.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Boolean/flag parameters that hide two methods.**
```abap
METHODS save IMPORTING iv_commit TYPE abap_bool.   " save( abap_true ) — what does true mean at the call site?
```
**Why this is wrong:** the call site is unreadable (`save( abap_true )`), and the method likely does two things.
**Correct approach:** split into intention-revealing methods (`save_and_commit`, `save_to_buffer`), per Clean ABAP.

**Mistake #2: Mixing legacy and modern style arbitrarily.**
```abap
CREATE OBJECT lo_x.                  " legacy
DATA(lv) = lo_x->calc( ).            " modern
DATA lt TYPE TABLE OF ty.            " classic declaration next to inline DATA(...)
```
**Why this is wrong:** inconsistency raises cognitive load and signals no shared standard.
**Correct approach:** adopt modern syntax consistently (`NEW`, inline declarations, constructor expressions); reserve legacy forms for genuine compatibility needs.

---

## 8. Comparison With Similar Concepts

**Clean ABAP vs ABAP Programming Guidelines:** Clean ABAP is the modern, OO-oriented community/SAP style guide focused on readability and maintainability; the older official guidelines cover broader/lower-level concerns. Clean ABAP is the day-to-day reference for OO code.

**Style guide vs static analysis (ATC/Code Pal/abaplint):** the guide states the *rules*; the tools *enforce* a subset automatically. The guide is broader (includes judgment-based advice); tools catch mechanical violations in CI and review.

**Clean code vs clever code:** Clean ABAP favours clear over clever — a readable `LOOP` can beat an unreadable nested expression. Modern syntax is a means to clarity, not terseness for its own sake.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **ABAP Test Cockpit (ATC):** the platform's check framework; runs on transport and in ADT, can include Code Pal checks.
- **SAP Code Pal for ABAP:** open-source ATC checks implementing many Clean ABAP rules (`github.com/SAP/code-pal-for-abap`).
- **abaplint:** linter for abapGit projects, runnable in CI pipelines for cloud/on-prem.
- **ADT:** Quick Fixes help apply modern syntax; ABAP Unit + coverage run alongside.

**SAP-Specific Considerations:** Clean ABAP explicitly notes context — legacy code, performance-critical sections, and generated code may justify exceptions; apply judgment, document deviations. Adopt the guide incrementally (boy-scout rule: leave touched code cleaner) rather than mass-rewriting.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: dogmatic application without judgment.**
```abap
" forcing a deep chain of tiny one-line methods or an interface per class
" everywhere, "because Clean ABAP", until the code is harder to follow
```
**Why this fails:** Clean ABAP is guidance, not law; over-applying any rule (tiny methods, ubiquitous interfaces) can reduce clarity — the opposite of the goal.
**Correct approach:** apply rules where they improve readability/maintainability; the guide itself advises pragmatism and documenting exceptions.

**Common Gotcha:** modern syntax requires a recent enough release/version. On older systems some constructs (e.g. certain inline or constructor-expression forms) may be unavailable — verify against your system's ABAP release before relying on them.

---

## 11. Testing & Validation

**How to Verify Understanding:** Pick a class you wrote. Are names intention-revealing, methods single-purpose with few parameters, classes `FINAL` and cohesive, errors exception-based, and is there a unit test? If several are "no," you have a concrete Clean ABAP backlog.

**Unit Test Example:**
```abap
" Clean code is testable code — the refactored invoice proves it:
CLASS ltcl_invoice DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION. METHODS gross_includes_tax FOR TESTING.
ENDCLASS.
CLASS ltcl_invoice IMPLEMENTATION.
  METHOD gross_includes_tax.
    " inject a predictable tax strategy (no DB, no CASE):
    DATA(lo) = NEW zcl_invoice( io_tax = NEW ltd_tax_ten_percent( ) ).
    cl_abap_unit_assert=>assert_equals( act = lo->gross_amount( ) exp = CONV p( '110.00' ) ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** because the refactor used injection and small methods, the behaviour is verifiable in isolation — clean design and testability are the same property viewed from two sides.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Clean ABAP turns this curriculum's principles into daily habits: clear names, small single-purpose methods/classes, modern syntax, composition, exception-based errors, tests.
- Enforce it with tooling (ATC, Code Pal, abaplint) and code review; adopt incrementally.
- Apply with judgment — the goal is readability and maintainability, not rule-counting.

**When to Apply:** every change. Leave each touched unit a little cleaner.

**Red Flags:** flag parameters; cryptic names; mixed legacy/modern syntax; god classes; silent error handling; no tests; or dogmatic over-application.

---

## 13. Dependency Map

**Depends On:**
- All of Module 02 — the mechanics Clean ABAP styles.
- `03_01`–`03_06` — SOLID, patterns, DI, and testing that Clean ABAP operationalizes.

**Enables:**
- Sustained, maintainable OO ABAP across teams and time — the destination of this curriculum.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — modern syntax: search "NEW", "Inline Declarations", "Constructor Expressions", "REDUCE", "Table Expressions".

**Design Patterns & Best Practices:** Clean ABAP style guide (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`); SAP Code Pal for ABAP (`github.com/SAP/code-pal-for-abap`); abaplint (`abaplint.org`). Conceptual roots: *Clean Code* (Robert C. Martin), adapted to ABAP by the community/SAP.
