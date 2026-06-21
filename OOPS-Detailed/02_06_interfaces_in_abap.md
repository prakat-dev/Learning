# Interfaces in ABAP

**Learning Objective:** After this topic you can define and implement ABAP interfaces, access interface components with the `~` operator, use aliases and interface references, and compose multiple interfaces in one class.

**Difficulty Level:** Intermediate
**Time to Master:** 75 minutes
**Prerequisites:** `01_06_abstraction_and_interfaces.md`, `02_01_abap_class_syntax.md`
**Official Sources:**
- ABAP Keyword Documentation → *INTERFACE*, *INTERFACES*, *ALIASES* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Interfaces* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A payment processor must support card, bank transfer, and voucher. They share nothing in common code, only a need to "authorize and capture an amount." Inheritance from a common base would force shared implementation that doesn't exist; an interface gives them a shared *contract* without forced shared code.

**What Happens WITHOUT This Concept.** You either invent an artificial base class (Topic 1.4's "is-a" abuse) or branch on type everywhere (Topic 1.5's `CASE` ladder). Both are brittle.

**Why This Matters in SAP.** ABAP has *single* inheritance but *unlimited* interface implementation. Interfaces are how unrelated classes share a contract — the backbone of BAdIs, RAP, and testable design.

---

## 3. Core Concept Explanation

**Definition.** An **interface** is a named, implementation-free set of components — methods, and optionally constants, types, and events. A class **implements** an interface by adding `INTERFACES if_name.` and providing every method body as `if_name~method`.

**Key Principles:**
- Interfaces declare *what*, never *how* (no method bodies, no instance attributes with state logic).
- A class can implement many interfaces; an interface can compose others.
- Access interface components with `~`: inside the class `if_name~method`, via a reference `ref->method`.
- Use **`ALIASES`** to expose an interface component under a shorter local name.

**How It Works.** An interface reference (`TYPE REF TO if_name`) can point to any object whose class implements that interface, enabling polymorphism (Topic 1.5). The `~` operator disambiguates which interface a component belongs to.

**Why It's Designed This Way.** Separating contract from implementation lets callers depend on the contract and lets any number of unrelated classes satisfy it — the most flexible decoupling ABAP offers.

---

## 4. Visual Representation

```
   INTERFACE if_payment            (contract: authorize / capture)
   ────────────────────────────────────────────────
        ▲              ▲              ▲
   implements      implements     implements
   ┌──────────┐  ┌────────────┐  ┌───────────┐
   │ card     │  │ transfer   │  │ voucher   │   (unrelated classes,
   └──────────┘  └────────────┘  └───────────┘    no shared base)

   DATA lo_pay TYPE REF TO if_payment.   ← holds ANY implementer
   lo_pay->authorize( amount ).          ← interface reference call (no ~)
   inside class: METHOD if_payment~authorize.  ← implementation uses ~
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Define a payment contract and one implementation.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS lcl_card DEFINITION.
  PUBLIC SECTION.
    METHODS authorize.    " plain method, no shared contract
ENDCLASS.
" Callers must know it's specifically a card → no polymorphism, no substitution.
```
**Problems with this code:**
- No common type; callers bind to `lcl_card` specifically.
- A second payment kind cannot be used interchangeably.

**The RIGHT Way (Following Best Practice):**
```abap
INTERFACE lif_payment.
  METHODS authorize IMPORTING iv_amount TYPE p LENGTH 13 DECIMALS 2
                    RETURNING VALUE(rv_ok) TYPE abap_bool.
ENDINTERFACE.

CLASS lcl_card DEFINITION.
  PUBLIC SECTION.
    INTERFACES lif_payment.                 " promise to fulfil the contract
ENDCLASS.
CLASS lcl_card IMPLEMENTATION.
  METHOD lif_payment~authorize.             " note the ~ qualification
    rv_ok = xsdbool( iv_amount <= 5000 ).   " card-specific rule
  ENDMETHOD.
ENDCLASS.

" Used through the contract:
DATA lo_pay TYPE REF TO lif_payment.
lo_pay = NEW lcl_card( ).
DATA(lv_ok) = lo_pay->authorize( '199.00' ).   " no ~ when calling via a reference
```
**Why this is better:**
- Callers depend on `lif_payment`; any implementer substitutes freely (polymorphism).
- A second payment kind needs no caller changes.

**Step-by-Step Explanation:**
- `INTERFACE … METHODS authorize` — the contract, no body.
- `INTERFACES lif_payment` + `METHOD lif_payment~authorize` — the class commits and implements, using `~`.
- `lo_pay->authorize( )` — through an interface reference, you call without `~`.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A class implements *two* interfaces (payment + auditable), uses an alias for readability, and a service depends only on the contracts.

```abap
INTERFACE lif_auditable.
  METHODS audit_text RETURNING VALUE(rv) TYPE string.
ENDINTERFACE.

CLASS lcl_transfer DEFINITION.
  PUBLIC SECTION.
    INTERFACES lif_payment.
    INTERFACES lif_auditable.
    ALIASES authorize FOR lif_payment~authorize.   " shorter local name
  PRIVATE SECTION.
    DATA mv_iban TYPE string.
ENDCLASS.

CLASS lcl_transfer IMPLEMENTATION.
  METHOD lif_payment~authorize.
    rv_ok = abap_true.                  " transfer-specific rule (abbreviated)
  ENDMETHOD.
  METHOD lif_auditable~audit_text.
    rv = |Bank transfer { mv_iban }|.
  ENDMETHOD.
ENDCLASS.

" A service depends on BOTH contracts, not the class:
CLASS lcl_checkout DEFINITION.
  PUBLIC SECTION.
    METHODS pay IMPORTING io_payment TYPE REF TO lif_payment
                          io_audit   TYPE REF TO lif_auditable
                          iv_amount  TYPE p LENGTH 13 DECIMALS 2.
ENDCLASS.
CLASS lcl_checkout IMPLEMENTATION.
  METHOD pay.
    IF io_payment->authorize( iv_amount ) = abap_true.
      cl_demo_output=>write( io_audit->audit_text( ) ).   " uses the audit contract
    ENDIF.
  ENDMETHOD.
ENDCLASS.

" One object satisfies both interfaces:
DATA(lo_tx) = NEW lcl_transfer( ).
NEW lcl_checkout( )->pay( io_payment = lo_tx io_audit = lo_tx iv_amount = '100.00' ).
```

**Detailed Walkthrough:**
- **Two `INTERFACES`** — `lcl_transfer` is both a payment and auditable, impossible with single inheritance.
- **`ALIASES authorize FOR lif_payment~authorize`** — lets the class (and aware callers) use `authorize` directly.
- **`lcl_checkout` depends on interface references** — it works with any payment+auditable implementer.

**How This Works in Practice.** Mixing capabilities via multiple interfaces (a payment that is also auditable, also serializable, …) is the idiomatic ABAP way to compose behaviour without inheritance.

**Why This Implementation.** Each interface stays small and focused (Interface Segregation, Topic 3.1); the class opts into exactly the contracts it fulfils.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Omitting the `~` when implementing.**
```abap
CLASS lcl_card IMPLEMENTATION.
  METHOD authorize.            " wrong: must be lif_payment~authorize
  ENDMETHOD.
ENDCLASS.
```
**Why this is wrong:** the method belongs to the interface; ABAP requires `interface~method` in the implementation (unless an alias is defined).
**Correct approach:** `METHOD lif_payment~authorize.` (or define an `ALIASES`).

**Mistake #2: Not implementing every interface method.**
```abap
INTERFACES lif_payment.        " declares authorize AND capture...
" ...but only authorize is implemented → class is missing a method body
```
**Why this is wrong:** a class must implement *all* methods of every interface it declares (unless they are abstract, for an abstract class).
**Correct approach:** implement every method, or make the class abstract.

---

## 8. Comparison With Similar Concepts

**Interface vs Abstract class (Topic 2.7):** an interface is pure contract and a class may implement many; an abstract class can hold shared implementation and state but you inherit only one. Use interfaces for "can-do" capabilities across unrelated classes; abstract classes for a true "is-a" family with shared code.

**Interface method vs class method:** identical parameter rules (Topic 2.3); the only difference is the `~` qualification inside the implementing class and that the contract has no body.

**`ALIASES` vs full `~` name:** aliases are convenience/readability; the underlying component is the same. Use aliases to tidy frequently used interface components.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **BAdIs / enhancement spots:** defined as interfaces; implementations are classes.
- **Factories & DI (Topics 3.3, 3.5):** return/inject interface references so callers stay decoupled.
- **Test doubles (Topic 3.6):** a fake class implements the same interface; the ABAP test-double framework (`cl_abap_testdouble`) generates one for an interface.
- **Standard SAP interfaces (`IF_*`):** implementing them lets your class plug into SAP frameworks.

**SAP-Specific Considerations:** interface references can be cast to/from class references (Topic 2.8); frequent casting hints at a leaky abstraction. Interfaces may declare constants/types/events too — useful for shared enumerations and contracts.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the "header interface" mirroring one class's every public method.**
```abap
INTERFACE lif_order.   " has exactly the public methods of lcl_order, 1:1
ENDINTERFACE.
```
**Why this fails:** an interface with a single implementer and no second consumer/test need adds indirection for nothing (Clean ABAP cautions against this).
**Correct approach:** introduce an interface when a second implementation or a test double actually requires it.

**Common Gotcha:** when two implemented interfaces declare a method of the same name, you must qualify with `interface~method` (or aliases) to disambiguate — a plain unqualified call is ambiguous.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you write a second class implementing the same interface and use both through one `REF TO if_…` variable? If not, revisit the contract.

**Unit Test Example:**
```abap
" A test double implementing the contract — substitutes the real payment:
CLASS ltd_payment DEFINITION.
  PUBLIC SECTION.
    INTERFACES lif_payment.
    DATA mv_called_with TYPE p LENGTH 13 DECIMALS 2.
ENDCLASS.
CLASS ltd_payment IMPLEMENTATION.
  METHOD lif_payment~authorize.
    mv_called_with = iv_amount.
    rv_ok = abap_true.
  ENDMETHOD.
ENDCLASS.

CLASS ltcl_checkout DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS pay_authorizes FOR TESTING.
ENDCLASS.
CLASS ltcl_checkout IMPLEMENTATION.
  METHOD pay_authorizes.
    DATA(lo_double) = NEW ltd_payment( ).
    NEW lcl_checkout( )->pay( io_payment = lo_double
                              io_audit   = NEW lcl_transfer( )
                              iv_amount  = '42.00' ).
    cl_abap_unit_assert=>assert_equals( act = lo_double->mv_called_with exp = CONV p( '42.00' ) ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** because `lcl_checkout` depends on `lif_payment`, the test injects a recording double — verifying interaction without any real payment system.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- An interface is a pure contract; a class implements it with `INTERFACES` + `interface~method` bodies.
- One class can implement many interfaces; an interface reference enables polymorphism.
- Use `ALIASES` for readability; implement *every* interface method.

**When to Apply:** when a capability has (or will have) multiple implementers, must be mixed across unrelated classes, or must be faked in tests.

**Red Flags:** missing `~`; unimplemented interface methods; single-implementer "header" interfaces; ambiguous same-named methods left unqualified.

---

## 13. Dependency Map

**Depends On:**
- `01_06_abstraction_and_interfaces.md` — the *why* of interfaces.
- `02_01_abap_class_syntax.md` — classes implement interfaces.

**Enables:**
- `02_08_casting_and_rtti.md` — casting between class and interface references.
- `03_03_factory_pattern.md`, `03_05_dependency_injection.md`, `03_06_abap_unit_and_test_doubles.md`.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "INTERFACE", "INTERFACES" (in classes), "ALIASES", "Interface Reference Variables", "Component Selector" (the `~`).

**Design Patterns & Best Practices:** Clean ABAP → *Interfaces* (keep them small; don't create interfaces for single implementations) (`github.com/SAP/styleguides`).
