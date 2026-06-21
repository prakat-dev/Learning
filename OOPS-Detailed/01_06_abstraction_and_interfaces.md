# Abstraction and Interfaces

**Learning Objective:** After this topic you can separate *what* an operation does from *how* it does it, expressing the "what" as an interface so callers depend on a contract instead of a concrete class.

**Difficulty Level:** Foundational
**Time to Master:** 75 minutes
**Prerequisites:** `01_02_classes_and_objects.md`
**Official Sources:**
- ABAP Keyword Documentation → *INTERFACE*, *INTERFACES* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Interfaces* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** An order needs to send a notification. Today it sends email; next quarter it must also support SMS and a Fiori in-app message. If the order class calls `lcl_email_sender` directly, adding SMS means editing the order class — and it now knows about three senders it shouldn't care about.

**What Happens WITHOUT This Concept.** Callers bind to concrete classes, so every new variant forces edits to the caller. Code becomes a web of `IF channel = 'EMAIL' … ELSEIF channel = 'SMS' …` that grows with every option and cannot be unit-tested without the real email server.

**Why This Matters in SAP.** SAP code outlives its integrations. Channels, providers, and back-ends change. Depending on a *contract* rather than a concrete implementation is what lets the stable part stay stable while the volatile part is swapped.

---

## 3. Core Concept Explanation

**Definition.**
- **Abstraction** is exposing only the essential operations of something while hiding the details — describing *what* it does, not *how*.
- An **interface** is the concrete ABAP construct for abstraction: a named set of method signatures (a **contract**) with no implementation. Any class can **implement** it by providing the bodies.

**Key Principles:**
- Callers should depend on the interface, not on a concrete class.
- A class may implement *many* interfaces (unlike single inheritance).
- An interface has no state and no method bodies — pure contract.

**How It Works.** `INTERFACE lif_sender. METHODS send … ENDINTERFACE.` declares the contract. A class adds `INTERFACES lif_sender.` and implements `lif_sender~send`. Callers hold a `REF TO lif_sender` and never know which class is behind it.

**Why It's Designed This Way.** Programming to an interface decouples the *user* of a capability from its *provider*. You can add providers, or substitute a fake one in tests, without touching the user — the foundation of testability and extensibility.

---

## 4. Visual Representation

```
                    lif_sender   (the CONTRACT: send(msg))
                    ───────────────────────────────────
                          ▲           ▲           ▲
            implements    │           │           │  implements
       ┌──────────────────┘   ┌───────┘   ┌───────┘
   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
   │ email_sender  │   │ sms_sender    │   │ fiori_sender  │
   └───────────────┘   └───────────────┘   └───────────────┘

   caller ──► REF TO lif_sender ──► send( msg )
              (knows the contract, NOT the concrete class)
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Define a notification contract and one implementation.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    METHODS notify IMPORTING iv_msg TYPE string.
  PRIVATE SECTION.
    DATA mo_email TYPE REF TO lcl_email_sender.   " bound to a CONCRETE class
ENDCLASS.
CLASS lcl_order IMPLEMENTATION.
  METHOD notify.
    mo_email->send_email( iv_msg ).               " can ONLY ever email
  ENDMETHOD.
ENDCLASS.
```
**Problems with this code:**
- Adding SMS requires editing `lcl_order`.
- You cannot test `notify` without a real email sender.
- `lcl_order` knows details it shouldn't.

**The RIGHT Way (Following Best Practice):**
```abap
INTERFACE lif_sender.
  METHODS send IMPORTING iv_msg TYPE string.      " the CONTRACT: just 'what'
ENDINTERFACE.

CLASS lcl_email_sender DEFINITION.
  PUBLIC SECTION.
    INTERFACES lif_sender.                          " promises to fulfil the contract
ENDCLASS.
CLASS lcl_email_sender IMPLEMENTATION.
  METHOD lif_sender~send.
    " ... real email-sending detail lives HERE, hidden from callers ...
    cl_demo_output=>write( |EMAIL: { iv_msg }| ).
  ENDMETHOD.
ENDCLASS.
```
**Why this is better:**
- `lif_sender` names the capability; the email detail is hidden behind it.
- A second implementation (SMS) requires *no* change to anything that depends on `lif_sender`.

**Step-by-Step Explanation:**
- `INTERFACE lif_sender … METHODS send` — declares *what* a sender does, with no body.
- `INTERFACES lif_sender` — the class commits to the contract.
- `METHOD lif_sender~send` — the class supplies the *how*; note the `interface~method` naming.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** An order notifies via whatever sender it is *given*. New channels plug in without editing the order.

```abap
CLASS lcl_sms_sender DEFINITION.
  PUBLIC SECTION.
    INTERFACES lif_sender.
ENDCLASS.
CLASS lcl_sms_sender IMPLEMENTATION.
  METHOD lif_sender~send.
    cl_demo_output=>write( |SMS: { iv_msg }| ).
  ENDMETHOD.
ENDCLASS.

CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    " The order is GIVEN a sender; it depends only on the contract:
    METHODS constructor IMPORTING io_sender TYPE REF TO lif_sender.
    METHODS confirm.
  PRIVATE SECTION.
    DATA mo_sender TYPE REF TO lif_sender.    " interface reference, not a class
ENDCLASS.
CLASS lcl_order IMPLEMENTATION.
  METHOD constructor.
    mo_sender = io_sender.
  ENDMETHOD.
  METHOD confirm.
    mo_sender->send( 'Your order is confirmed' ).   " 'what', not 'how'
  ENDMETHOD.
ENDCLASS.

" Choose the channel at the edge; the order never changes:
DATA(lo_order_email) = NEW lcl_order( NEW lcl_email_sender( ) ).
DATA(lo_order_sms)   = NEW lcl_order( NEW lcl_sms_sender( ) ).
lo_order_email->confirm( ).   " EMAIL: ...
lo_order_sms->confirm( ).     " SMS: ...
```

**Detailed Walkthrough:**
- **`io_sender TYPE REF TO lif_sender`** — the order accepts *any* sender; this is dependency injection (Topic 3.5) in embryo.
- **`mo_sender->send( … )`** — the order calls the contract; it has no idea whether it's email or SMS.
- **The edge code** decides the concrete channel; the order is closed to that change.

**How This Works in Practice.** Marketing adds a push-notification channel: write `lcl_push_sender` implementing `lif_sender`, pass it in. `lcl_order` is not recompiled in spirit — it never knew the channels.

**Why This Implementation.** This is the practical payoff of abstraction: the volatile thing (channel) varies freely behind a fixed contract while the stable thing (order) stays put.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: A "fat" interface forcing implementers to stub methods they don't need.**
```abap
INTERFACE lif_sender.
  METHODS send.
  METHODS connect.       " not every sender connects
  METHODS authenticate.  " not every sender authenticates
  METHODS retry.         " ...
ENDINTERFACE.
" lcl_fiori_sender must now implement empty connect/authenticate/retry → smell
```
**Why this is wrong:** it violates the Interface Segregation Principle (Topic 3.1); implementers carry irrelevant methods.
**Correct approach:** keep interfaces small and focused; split into `lif_sender` and `lif_connectable` if needed.

**Mistake #2: Forgetting the `interface~` prefix when implementing.**
```abap
CLASS lcl_email_sender IMPLEMENTATION.
  METHOD send.            " wrong: must be lif_sender~send
  ENDMETHOD.
ENDCLASS.
```
**Why this is wrong:** the method belongs to the interface; ABAP needs the `lif_sender~send` qualification.
**Correct approach:** `METHOD lif_sender~send.`

---

## 8. Comparison With Similar Concepts

**Interface vs Abstract class (Topic 2.7):** an interface is pure contract (no state, no bodies) and a class can implement many; an abstract class can provide shared implementation and state but you can inherit only one. Use an interface for "can-do" capabilities across unrelated classes; use an abstract base for a true "is-a" family that shares code.

**Abstraction vs Encapsulation (Topic 1.3):** encapsulation hides *data* inside an object; abstraction hides *implementation* behind a simplified contract. Encapsulation protects state; abstraction decouples callers from providers.

**Interface vs concrete dependency:** depending on `lif_sender` rather than `lcl_email_sender` is the difference between code that bends and code that breaks when requirements change.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **BAdIs / enhancement framework:** classic BAdIs are interface-based — SAP defines the contract, customers provide implementations.
- **Factory / DI:** factories (Topic 3.3) typically return interface references so callers stay decoupled.
- **RAP & services:** service consumers depend on published contracts, not internal classes.

**SAP-Specific Considerations:** SAP ships many standard interfaces (e.g. `IF_*`); implementing them lets your class participate in SAP frameworks. Interface references can be cast to/from concrete references (Topic 2.8) — but needing to cast often signals a leaky abstraction.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: an interface with a single implementation created "just in case."**
```abap
INTERFACE lif_thing. METHODS do. ENDINTERFACE.   " only ever lcl_thing implements it
```
**Why this fails:** speculative abstraction adds indirection with no payoff (YAGNI). Clean ABAP cautions against interfaces that exist only for one implementer.
**Correct approach:** introduce the interface when a *second* implementation or a *test double* actually needs it.

**Common Gotcha:** interface components are accessed with `~` (`lif_sender~send`) inside the implementing class, but callers using an interface reference simply write `ref->send( )`.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you add a new implementation without touching the caller? If the caller must change, it is bound to a concretion, not the abstraction.

**Unit Test Example:**
```abap
" A TEST DOUBLE implementing the same contract — no real email server needed:
CLASS ltd_sender DEFINITION.
  PUBLIC SECTION.
    INTERFACES lif_sender.
    DATA mv_last_msg TYPE string.
ENDCLASS.
CLASS ltd_sender IMPLEMENTATION.
  METHOD lif_sender~send.
    mv_last_msg = iv_msg.        " record instead of sending
  ENDMETHOD.
ENDCLASS.

CLASS ltcl_order DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS confirm_sends_message FOR TESTING.
ENDCLASS.
CLASS ltcl_order IMPLEMENTATION.
  METHOD confirm_sends_message.
    DATA(lo_double) = NEW ltd_sender( ).
    DATA(lo_order)  = NEW lcl_order( lo_double ).   " inject the double via the contract

    lo_order->confirm( ).

    cl_abap_unit_assert=>assert_equals(
      act = lo_double->mv_last_msg exp = 'Your order is confirmed' ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** because `lcl_order` depends on `lif_sender`, the test swaps in a recording double — proving the order sends the right message without any real channel. This is the direct testability dividend of abstraction.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Abstraction exposes *what*, hides *how*; an interface is its ABAP form — a contract with no bodies.
- Depend on interfaces, not concrete classes, so providers can change/multiply freely.
- One class can implement many interfaces; keep each interface small and focused.

**When to Apply:** when a capability has (or will have) more than one implementation, or must be faked in tests.

**Red Flags:** caller holding a concrete `REF TO lcl_x` for a varying capability; fat interfaces; speculative single-implementer interfaces.

---

## 13. Dependency Map

**Depends On:** `01_02_classes_and_objects.md` — interfaces are implemented by classes and used via references.

**Enables:**
- `01_05_polymorphism.md` — interfaces are the cleanest route to polymorphism.
- `02_06_interfaces_in_abap.md` — full ABAP interface mechanics.
- `03_03_factory_pattern.md`, `03_05_dependency_injection.md` — both return/inject interface references.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "INTERFACE" (declaration), "INTERFACES" (implementation in a class), and "Interface Reference Variables".

**Design Patterns & Best Practices:** Clean ABAP → *Interfaces* (don't add interfaces for a single implementation; keep them focused) (`github.com/SAP/styleguides`). Conceptual basis: the Dependency Inversion and Interface Segregation principles (Topic 3.1).
