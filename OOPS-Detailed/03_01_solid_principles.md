# SOLID Principles in ABAP

**Learning Objective:** After this topic you can recognize and apply the five SOLID principles in ABAP — using them as concrete design checks rather than slogans — to produce classes that are easy to change, extend, and test.

**Difficulty Level:** Advanced
**Time to Master:** 120 minutes
**Prerequisites:** `02_06_interfaces_in_abap.md`, `02_07_inheritance_in_abap.md`, `02_09_events_and_handlers.md`
**Official Sources:**
- Clean ABAP → *Classes*, *Methods*, *Interfaces* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)
- ABAP Keyword Documentation → *Classes*, *Interfaces* (`help.sap.com/doc/abapdocu_latest_index_htm`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A single `zcl_order` class reads the database, calculates pricing, formats a PDF, and sends email. A change to the email server forces you to retest pricing. Adding a new output format means editing the class everyone depends on. Every change risks everything.

**What Happens WITHOUT This Concept.** Classes accrete responsibilities, depend on concretions, and resist extension. Small requirements cause wide, risky edits; unit testing is impossible because everything is wired together.

**Why This Matters in SAP.** Long-lived SAP systems are changed constantly. SOLID gives five concrete levers to keep classes small, substitutable, and decoupled — exactly what Clean ABAP codifies for maintainable enterprise code.

---

## 3. Core Concept Explanation

**Definition.** SOLID is five design principles:
- **S — Single Responsibility:** a class has one reason to change.
- **O — Open/Closed:** open for extension, closed for modification (add behaviour without editing existing code).
- **L — Liskov Substitution:** a subtype must be usable wherever its base type is expected, without surprises.
- **I — Interface Segregation:** prefer several small, focused interfaces over one fat one.
- **D — Dependency Inversion:** depend on abstractions (interfaces), not concretions; inject dependencies.

**Key Principles:**
- They are *checks*, not dogma — apply them where change is likely.
- They reinforce each other: DIP + ISP enable OCP; SRP makes LSP easier.
- In ABAP they map to interfaces (Topic 2.6), polymorphism (Topic 1.5), and injection (Topic 3.5).

**How It Works.** Each principle removes a specific source of fragility: SRP limits blast radius, OCP avoids editing tested code, LSP keeps polymorphism safe, ISP avoids forcing clients to depend on what they don't use, DIP breaks compile-time coupling to implementations.

**Why It's Designed This Way.** Together they target one goal: localize change. Code that obeys SOLID can be extended and tested by adding small pieces rather than editing large ones.

---

## 4. Visual Representation

```
   S  one class, one job            zcl_pricing | zcl_pdf | zcl_mailer  (not one mega-class)
   O  extend, don't modify          new behaviour = new class implementing an interface
   L  subtypes substitute safely    REF TO base ← any subclass, no special-casing
   I  small interfaces              lif_readable + lif_writable  (not lif_everything)
   D  depend on abstractions        zcl_service( io_repo TYPE REF TO lif_repo )  ← injected
                                                                         ▲ interface, not a table/DB class
```

---

## 5. Code Example 1: Basic Concept (Single Responsibility + Dependency Inversion)

**Scenario:** Separate responsibilities and depend on an abstraction.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS zcl_order DEFINITION PUBLIC.
  PUBLIC SECTION.
    METHODS process.
ENDCLASS.
CLASS zcl_order IMPLEMENTATION.
  METHOD process.
    SELECT * FROM vbak INTO TABLE @DATA(lt).   " data access
    " ...pricing math...                       " business logic
    " ...build PDF...                           " presentation
    cl_bcs=>... " send email                    " infrastructure
  ENDMETHOD.
ENDCLASS.
```
**Problems with this code:**
- One class, four reasons to change (SRP violated).
- Hard-wired to DB, PDF, and email classes (DIP violated) — untestable.

**The RIGHT Way (Following Best Practice):**
```abap
INTERFACE lif_order_repo.
  METHODS read IMPORTING iv_id TYPE vbeln RETURNING VALUE(rs) TYPE vbak.
ENDINTERFACE.

CLASS zcl_order_service DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING io_repo TYPE REF TO lif_order_repo.  " inject abstraction
    METHODS total IMPORTING iv_id TYPE vbeln RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
  PRIVATE SECTION.
    DATA mo_repo TYPE REF TO lif_order_repo.
ENDCLASS.
CLASS zcl_order_service IMPLEMENTATION.
  METHOD constructor.
    mo_repo = io_repo.                 " depends on lif_order_repo, not on a DB class
  ENDMETHOD.
  METHOD total.
    DATA(ls) = mo_repo->read( iv_id ).
    rv = ls-netwr.                     " one job: compute the total
  ENDMETHOD.
ENDCLASS.
```
**Why this is better:**
- The service has one responsibility (compute total) and depends on `lif_order_repo`, not the database.
- A test can inject a fake repo; PDF/email live in their own classes.

**Step-by-Step Explanation:**
- `INTERFACE lif_order_repo` — the abstraction the service depends on (DIP).
- `constructor IMPORTING io_repo` — the dependency is injected (Topic 3.5).
- `total` does only pricing — one reason to change (SRP).

---

## 6. Code Example 2: Real-World Application (Open/Closed + Interface Segregation)

**Business Scenario:** Output an order in several formats. New formats must not edit existing code; clients depend only on the small interface they use.

```abap
" ISP: a small, focused interface — not a fat "do everything" one
INTERFACE lif_order_output.
  METHODS render IMPORTING is_order TYPE vbak RETURNING VALUE(rv) TYPE string.
ENDINTERFACE.

CLASS zcl_output_csv DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION. INTERFACES lif_order_output.
ENDCLASS.
CLASS zcl_output_csv IMPLEMENTATION.
  METHOD lif_order_output~render.
    rv = |{ is_order-vbeln };{ is_order-netwr }|.
  ENDMETHOD.
ENDCLASS.

CLASS zcl_output_json DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION. INTERFACES lif_order_output.
ENDCLASS.
CLASS zcl_output_json IMPLEMENTATION.
  METHOD lif_order_output~render.
    rv = |{ "id": "{ is_order-vbeln }", "total": { is_order-netwr } }|.
  ENDMETHOD.
ENDCLASS.

" The consumer is CLOSED to modification but OPEN to new formats:
CLASS zcl_report DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS emit IMPORTING io_output TYPE REF TO lif_order_output
                           is_order  TYPE vbak
                 RETURNING VALUE(rv) TYPE string.
ENDCLASS.
CLASS zcl_report IMPLEMENTATION.
  METHOD emit.
    rv = io_output->render( is_order ).   " works with ANY current/future format
  ENDMETHOD.
ENDCLASS.

" Adding XML output later = a NEW class implementing lif_order_output.
" zcl_report is NEVER edited. (Open/Closed)
DATA(lv) = NEW zcl_report( )->emit( io_output = NEW zcl_output_json( ) is_order = ls_order ).
```

**Detailed Walkthrough:**
- **`lif_order_output` is tiny (ISP)** — one method; clients aren't forced to depend on unrelated capabilities.
- **`zcl_report` is closed (OCP)** — it never changes when formats are added; it depends on the interface.
- **New format = new class** — extension by addition, not modification.

**How This Works in Practice.** This is OCP via polymorphism (and the Strategy pattern, Topic 3.4): the variable part lives behind an interface; the stable part calls it.

**Why This Implementation.** Small interfaces + dependency on abstractions are what *make* a class open/closed — the principles compound.

---

## 7. Code Example 3: Common Mistakes (Liskov Substitution)

**Mistake #1: A subtype that breaks the base contract.**
```abap
CLASS zcl_read_only_list DEFINITION INHERITING FROM zcl_list.
  PUBLIC SECTION. METHODS add REDEFINITION.
ENDCLASS.
CLASS zcl_read_only_list IMPLEMENTATION.
  METHOD add.
    RAISE EXCEPTION TYPE cx_sy_illegal_handler.   " base promised add() works; subtype refuses
  ENDMETHOD.
ENDCLASS.
```
**Why this is wrong:** code using a `zcl_list` expects `add( )` to work; substituting the read-only subtype breaks that expectation (LSP violated).
**Correct approach:** don't model "read-only list" as a subtype of a mutable list; separate the interfaces (`lif_readable` vs `lif_writable`, ISP again).

**Mistake #2: Fat interface forcing empty implementations.**
```abap
INTERFACE lif_device.    " print + scan + fax all in one
  METHODS print. METHODS scan. METHODS fax.
ENDINTERFACE.
" a print-only device must implement scan/fax as no-ops or raise → ISP violated
```
**Why this is wrong:** clients and implementers depend on methods they don't use.
**Correct approach:** split into `lif_printer`, `lif_scanner`, `lif_fax`; classes implement only what they support.

---

## 8. Comparison With Similar Concepts

**SRP vs "small classes":** SRP is about *reasons to change*, not line count. A long class with one cohesive job can be fine; a short class touching DB + UI is not.

**OCP via inheritance vs via composition:** you can extend by subclassing (Template Method, Topic 2.7) or by injecting a strategy (composition, Topic 3.4). Clean ABAP prefers composition; both achieve "closed to modification."

**DIP vs DI:** Dependency *Inversion* is the principle (depend on abstractions); Dependency *Injection* (Topic 3.5) is a technique that implements it. You can invert dependencies via interfaces and inject them via the constructor.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **Interfaces & BAdIs (Topic 2.6):** the abstraction seams that enable DIP/OCP.
- **DI & factories (Topics 3.3, 3.5):** wire concrete implementations to abstractions.
- **ABAP Unit (Topic 3.6):** SOLID code is testable code — injected abstractions become test doubles.
- **Code Pal / abaplint:** static checks flag some SOLID smells (long methods, many parameters).

**SAP-Specific Considerations:** RAP separates data model, behaviour, and projection — itself an SRP/ISP-style separation. Over-applying SOLID (an interface per class "just in case") is itself a smell; apply it where change actually concentrates.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the "manager"/"util" god class.**
```abap
CLASS zcl_order_manager DEFINITION PUBLIC.
  PUBLIC SECTION.
    METHODS read. METHODS price. METHODS print. METHODS mail. METHODS archive.
ENDCLASS.
```
**Why this fails:** many responsibilities, many reasons to change, untestable, every consumer coupled to all of it.
**Correct approach:** decompose by responsibility; coordinate small classes via a thin service.

**Common Gotcha:** SOLID principles trade simplicity for flexibility. For genuinely stable, trivial code, introducing interfaces and injection adds indirection for no benefit. Apply the principles where requirements move, not everywhere.

---

## 11. Testing & Validation

**How to Verify Understanding:** For each class ask: one reason to change? Can I add behaviour without editing it? Are its dependencies injected abstractions? If yes, it is testable in isolation.

**Unit Test Example:**
```abap
" DIP pays off: inject a fake repo, no database needed
CLASS ltd_repo DEFINITION.
  PUBLIC SECTION. INTERFACES lif_order_repo.
ENDCLASS.
CLASS ltd_repo IMPLEMENTATION.
  METHOD lif_order_repo~read.
    rs = VALUE vbak( vbeln = iv_id netwr = '500.00' ).   " canned data
  ENDMETHOD.
ENDCLASS.

CLASS ltcl_service DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION. METHODS total_uses_repo FOR TESTING.
ENDCLASS.
CLASS ltcl_service IMPLEMENTATION.
  METHOD total_uses_repo.
    DATA(lo) = NEW zcl_order_service( io_repo = NEW ltd_repo( ) ).
    cl_abap_unit_assert=>assert_equals( act = lo->total( '1' ) exp = CONV p( '500.00' ) ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** because the service depends on the `lif_order_repo` abstraction, the test injects a fake and runs with no database — the direct dividend of DIP.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- SRP (one reason to change), OCP (extend without editing), LSP (safe substitution), ISP (small interfaces), DIP (depend on abstractions).
- In ABAP: interfaces + polymorphism + constructor injection realize SOLID.
- Apply where change concentrates; over-application is its own anti-pattern.

**When to Apply:** classes likely to grow, be extended, or need isolated testing.

**Red Flags:** god/manager classes; subtypes that refuse base behaviour; fat interfaces; classes that `NEW` their own dependencies.

---

## 13. Dependency Map

**Depends On:**
- `02_06_interfaces_in_abap.md` — the abstraction mechanism behind DIP/ISP/OCP.
- `02_07_inheritance_in_abap.md` — LSP and inheritance-based OCP.

**Enables:**
- `03_03_factory_pattern.md`, `03_04_strategy_pattern.md`, `03_05_dependency_injection.md` — concrete techniques that embody these principles.
- `03_07_clean_abap_in_practice.md` — SOLID as part of everyday clean code.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "Classes", "Interfaces" for the language mechanisms.

**Design Patterns & Best Practices:** Clean ABAP → *Classes*/*Methods*/*Interfaces* (small classes, prefer composition, depend on interfaces) (`github.com/SAP/styleguides`). Origin of SOLID: Robert C. Martin's design-principle papers.
