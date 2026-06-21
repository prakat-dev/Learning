# Dependency Injection

**Learning Objective:** After this topic you can apply constructor (and setter) injection in ABAP, depend on injected abstractions instead of self-created concretions, organize wiring in a composition root, and thereby make classes testable in isolation.

**Difficulty Level:** Advanced
**Time to Master:** 75 minutes
**Prerequisites:** `02_04_constructors.md`, `02_06_interfaces_in_abap.md`, `03_01_solid_principles.md`
**Official Sources:**
- Clean ABAP → *Object Orientation* / *Constructors* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)
- SAP test isolation examples (`github.com/SAP-samples/abap-test-isolation-examples`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A `zcl_order_service` creates its own database access object and email sender with `NEW` inside its methods. To unit-test the pricing logic you'd have to hit the real database and send real email — so it's never tested.

**What Happens WITHOUT This Concept.** A class that constructs its own collaborators is welded to them. You cannot substitute a fake for tests, cannot swap implementations, and a change in any collaborator ripples into the class.

**Why This Matters in SAP.** DI is the mechanism that makes the Dependency Inversion principle (Topic 3.1) real and that makes ABAP Unit testing (Topic 3.6) possible. It is the difference between code you can test in milliseconds and code you can only test against a live system.

---

## 3. Core Concept Explanation

**Definition.** **Dependency Injection** means a class receives its collaborators from outside (injected) rather than creating them itself. The most common form is **constructor injection**: dependencies are passed as constructor parameters, typed as interfaces.

**Key Principles:**
- The class declares *what it needs* (an interface), not *how to build it*.
- Dependencies are typed as abstractions (Topic 2.6), so any implementation — real or fake — fits.
- Wiring (deciding which concretions to inject) lives in one place: a **composition root** near the program entry point.
- Prefer constructor injection (mandatory, immutable deps); use setter injection for optional ones.

**How It Works.** Instead of `NEW zcl_db( )` inside a method, the constructor takes `io_repo TYPE REF TO lif_repo` and stores it. Production passes the real repo; a test passes a fake. The class is unchanged either way.

**Why It's Designed This Way.** Inverting "who creates the dependency" moves the coupling out of the class and into a single, swappable wiring point — exactly what makes substitution and isolated testing possible.

---

## 4. Visual Representation

```
   WITHOUT DI (welded)                    WITH DI (injected)
   ───────────────────                    ──────────────────
   zcl_service                            composition root (program start)
     METHOD do()                            lo = NEW zcl_service(
       NEW zcl_db( )   ← hard-wired              io_repo = NEW zcl_db( ) )  ← chosen here
       NEW zcl_mail( ) ← hard-wired         zcl_service( io_repo : lif_repo )
                                              uses lif_repo only
   can't test without real DB/mail        test: NEW zcl_service( io_repo = NEW fake_repo( ) )
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** A service that needs a repository.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS zcl_order_service DEFINITION PUBLIC.
  PUBLIC SECTION.
    METHODS total IMPORTING iv_id TYPE vbeln RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
ENDCLASS.
CLASS zcl_order_service IMPLEMENTATION.
  METHOD total.
    DATA(lo_repo) = NEW zcl_order_db( ).      " creates its own concrete dependency
    rv = lo_repo->read( iv_id )-netwr.        " welded to the database → untestable
  ENDMETHOD.
ENDCLASS.
```
**Problems with this code:**
- Hard-wired to `zcl_order_db`; can't substitute a fake.
- Testing `total` requires a real database.

**The RIGHT Way (Following Best Practice):**
```abap
INTERFACE lif_order_repo.
  METHODS read IMPORTING iv_id TYPE vbeln RETURNING VALUE(rs) TYPE vbak.
ENDINTERFACE.

CLASS zcl_order_service DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING io_repo TYPE REF TO lif_order_repo.   " injected
    METHODS total IMPORTING iv_id TYPE vbeln RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
  PRIVATE SECTION.
    DATA mo_repo TYPE REF TO lif_order_repo.
ENDCLASS.
CLASS zcl_order_service IMPLEMENTATION.
  METHOD constructor.
    mo_repo = io_repo.                  " store the abstraction
  ENDMETHOD.
  METHOD total.
    rv = mo_repo->read( iv_id )-netwr.  " uses whatever was injected
  ENDMETHOD.
ENDCLASS.

" Production wiring (composition root):
DATA(lo_service) = NEW zcl_order_service( io_repo = NEW zcl_order_db( ) ).
```
**Why this is better:**
- The service depends on `lif_order_repo`; the real DB repo is injected from outside.
- A test injects a fake repo and runs with no database.

**Step-by-Step Explanation:**
- `constructor IMPORTING io_repo TYPE REF TO lif_order_repo` — the dependency, as an interface.
- The service never `NEW`s a repo; it uses the one given.
- The concrete `zcl_order_db` is chosen at the wiring site, not inside the service.

---

## 6. Code Example 2: Real-World Application (composition root + multiple dependencies)

**Business Scenario:** A checkout service needs a repository, a tax strategy, and a notifier. All are injected; a single composition root wires production, and tests wire fakes.

```abap
CLASS zcl_checkout DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING io_repo     TYPE REF TO lif_order_repo
                                  io_tax      TYPE REF TO lif_tax
                                  io_notifier TYPE REF TO lif_notifier.
    METHODS checkout IMPORTING iv_id TYPE vbeln
                     RETURNING VALUE(rv_gross) TYPE p LENGTH 13 DECIMALS 2.
  PRIVATE SECTION.
    DATA: mo_repo     TYPE REF TO lif_order_repo,
          mo_tax      TYPE REF TO lif_tax,
          mo_notifier TYPE REF TO lif_notifier.
ENDCLASS.
CLASS zcl_checkout IMPLEMENTATION.
  METHOD constructor.
    mo_repo = io_repo. mo_tax = io_tax. mo_notifier = io_notifier.
  ENDMETHOD.
  METHOD checkout.
    DATA(ls) = mo_repo->read( iv_id ).
    rv_gross = ls-netwr + mo_tax->calc( ls-netwr ).
    mo_notifier->notify( |Order { iv_id } total { rv_gross }| ).
  ENDMETHOD.
ENDCLASS.

" ---- COMPOSITION ROOT: the ONE place that knows concrete classes ----
" (e.g. in the report's START-OF-SELECTION or a RAP handler entry point)
DATA(lo_checkout) = NEW zcl_checkout(
    io_repo     = NEW zcl_order_db( )
    io_tax      = zcl_tax_factory=>create( 'DE' )      " factory chooses the strategy
    io_notifier = NEW zcl_email_notifier( ) ).
DATA(lv_gross) = lo_checkout->checkout( '0000004711' ).
```

**Detailed Walkthrough:**
- **All three collaborators injected** as interfaces — the service is fully decoupled from concretions.
- **Composition root** — the only code that names `zcl_order_db`, `zcl_email_notifier`, etc.; everything downstream is abstract.
- **Factory + DI together** — the tax strategy is chosen by a factory (Topic 3.3) and injected.

**How This Works in Practice.** Push object creation to the edges (the entry point); keep the core logic dependency-free. ABAP has no built-in DI *container*, so wiring is explicit and manual — which is fine and clear at typical SAP scale.

**Why This Implementation.** Centralizing wiring makes dependencies visible and swappable; the business class reads as pure logic over abstractions.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: "Injecting" but defaulting to `NEW` inside.**
```abap
METHOD constructor.
  mo_repo = COND #( WHEN io_repo IS BOUND THEN io_repo
                    ELSE NEW zcl_order_db( ) ).   " hidden hard dependency remains
ENDMETHOD.
```
**Why this is wrong:** the fallback `NEW` re-welds the class to the concrete DB; tests that pass nothing silently hit the database.
**Correct approach:** make the dependency mandatory (no default), or default only to a harmless null-object — never to a real infrastructure class.

**Mistake #2: A service-locator masquerading as DI.**
```abap
METHOD total.
  DATA(lo) = zcl_registry=>get( 'REPO' ).   " pulls dependency from a global registry
ENDMETHOD.
```
**Why this is wrong:** the dependency is hidden (not in the signature) and globally mutable — the testability and clarity benefits of DI are lost.
**Correct approach:** declare dependencies explicitly in the constructor so they're visible and injectable.

---

## 8. Comparison With Similar Concepts

**Constructor vs setter injection:** constructor injection makes dependencies mandatory and the object valid-once-built (Topic 2.4); setter injection allows optional/late-bound dependencies but permits half-wired objects. Prefer constructor injection; use setters for genuinely optional collaborators.

**DI vs Service Locator:** both supply dependencies, but a locator *hides* them behind a global lookup (callers ask a registry), while DI *declares* them in the signature. DI is preferred for visibility and testability.

**DI vs `new`-ing internally:** internal `NEW` is fine for value objects and trivial, stable helpers; reserve injection for collaborators you want to substitute, mock, or vary (repositories, services, strategies, gateways).

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Factories (Topic 3.3):** choose concretions; the result is injected.
- **Strategy (Topic 3.4):** the strategy is an injected dependency.
- **ABAP Unit & test doubles (Topic 3.6):** DI is what lets a test pass a `cl_abap_testdouble` or hand-written fake.
- **RAP / handler classes:** keep handlers thin and inject domain services so the logic is testable apart from the framework.

**SAP-Specific Considerations:** there's no standard DI container in ABAP; wire manually at a composition root (report entry, factory, or a small assembler class). For hard-to-reach `NEW`s in legacy code, SAP offers *test seams* / *test injection* (`TEST-SEAM` / `TEST-INJECTION`) and the test-double framework as bridges (Topic 3.6).

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the over-injected "constructor with ten parameters."**
```abap
METHODS constructor IMPORTING io_a io_b io_c io_d io_e io_f io_g ...   " too many deps
```
**Why this fails:** many dependencies usually means the class does too much (SRP, Topic 3.1); the long constructor is a symptom, not the disease.
**Correct approach:** split the class by responsibility; group cohesive dependencies into a higher-level collaborator.

**Common Gotcha:** DI does not require a framework. Over-engineering a "DI container" in ABAP often adds more complexity than the manual wiring it replaces. Explicit construction at a composition root is usually the clearest design.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can each class be constructed in a test with fake collaborators and exercised without touching a database, RFC, or UI? If not, a dependency is still welded in.

**Unit Test Example:**
```abap
CLASS ltd_repo DEFINITION.
  PUBLIC SECTION. INTERFACES lif_order_repo.
ENDCLASS.
CLASS ltd_repo IMPLEMENTATION.
  METHOD lif_order_repo~read.
    rs = VALUE vbak( vbeln = iv_id netwr = '200.00' ).   " canned, no DB
  ENDMETHOD.
ENDCLASS.

CLASS ltcl_service DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION. METHODS total_reads_injected_repo FOR TESTING.
ENDCLASS.
CLASS ltcl_service IMPLEMENTATION.
  METHOD total_reads_injected_repo.
    DATA(lo) = NEW zcl_order_service( io_repo = NEW ltd_repo( ) ).   " inject the fake
    cl_abap_unit_assert=>assert_equals( act = lo->total( '1' ) exp = CONV p( '200.00' ) ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** injecting a fake repository lets the service's logic be verified in isolation, instantly and deterministically — the central payoff of DI.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- DI = collaborators are passed in (as interfaces), not created inside the class.
- Prefer constructor injection; wire concretions once at a composition root near the entry point.
- DI realizes DIP and is the prerequisite for isolated ABAP Unit tests.

**When to Apply:** any collaborator you want to substitute, fake, or vary — repositories, gateways, services, strategies.

**Red Flags:** `NEW` of infrastructure inside business methods; fallback `NEW` defaults; service-locator lookups; ten-parameter constructors; building a heavyweight DI container.

---

## 13. Dependency Map

**Depends On:**
- `02_04_constructors.md` — constructor injection.
- `02_06_interfaces_in_abap.md` — dependencies typed as abstractions.
- `03_01_solid_principles.md` — DI implements Dependency Inversion.

**Enables:**
- `03_06_abap_unit_and_test_doubles.md` — injecting doubles.
- `03_07_clean_abap_in_practice.md` — testable, decoupled everyday code.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "TEST-SEAM", "TEST-INJECTION", "CONSTRUCTOR".

**Design Patterns & Best Practices:** Clean ABAP → inject dependencies; avoid global state; keep constructors simple (`github.com/SAP/styleguides`). Test isolation techniques: `github.com/SAP-samples/abap-test-isolation-examples`.
