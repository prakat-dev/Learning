# Events and Event Handlers

**Learning Objective:** After this topic you can declare events, raise them, write handler methods, and register handlers with `SET HANDLER` — decoupling a publisher from its subscribers so new reactions can be added without changing the publisher.

**Difficulty Level:** Advanced
**Time to Master:** 75 minutes
**Prerequisites:** `02_03_methods.md`, `02_06_interfaces_in_abap.md`
**Official Sources:**
- ABAP Keyword Documentation → *EVENTS*, *CLASS-EVENTS*, *RAISE EVENT*, *SET HANDLER*, *FOR EVENT* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Object Orientation* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** When an order is confirmed, several things must happen: send a confirmation, write an audit entry, update a dashboard. Tomorrow marketing wants a fourth reaction. If the order class calls each reaction directly, every new reaction edits the order — and the order accumulates knowledge of systems it shouldn't care about.

**What Happens WITHOUT This Concept.** The publisher (order) hard-codes calls to every subscriber. Adding/removing reactions means editing the publisher; the publisher depends on all of them; testing the order pulls in email, audit, and dashboard code.

**Why This Matters in SAP.** Events implement the **Observer** pattern: the publisher announces "something happened" and any number of handlers react, registered from outside. This is how ABAP decouples "what occurred" from "who cares" — central to ALV, GUI controls, and clean domain events.

---

## 3. Core Concept Explanation

**Definition.** An **event** is a declared signal a class can **raise**; **handler methods** in other (or the same) classes react to it. Handlers are connected at runtime with **`SET HANDLER`**.

**Key Principles:**
- The publisher declares `EVENTS evt` (instance) or `CLASS-EVENTS evt` (static) and raises it with `RAISE EVENT`.
- A handler method is declared `FOR EVENT evt OF publisher_type` and may import the event's `EXPORTING` parameters (plus the implicit `sender`).
- `SET HANDLER handler->meth FOR publisher.` registers; `SET HANDLER … ACTIVATION abap_false.` deregisters.
- The publisher does **not** know its handlers — it only raises.

**How It Works.** `RAISE EVENT evt EXPORTING …` synchronously calls every registered handler, in registration order, in the same work process. Event parameters are `EXPORTING` on the event side and `IMPORTING` on the handler side; `sender` (the raising instance) is available to instance-event handlers.

**Why It's Designed This Way.** Inverting the dependency (handlers know the publisher, not vice-versa) lets you add reactions without touching the source of the event — the Open–Closed principle (Topic 3.1) for "things that happen."

---

## 4. Visual Representation

```
        PUBLISHER (zcl_order)                 SUBSCRIBERS (handlers)
        ┌───────────────────────┐
        │ EVENTS confirmed       │             zcl_mailer   ──┐
        │ ...                    │             zcl_auditor  ──┤ SET HANDLER ...
        │ RAISE EVENT confirmed  │──────────►  zcl_dashboard──┘ FOR lo_order
        └───────────────────────┘
              "announces"                       each handler's method runs
        publisher knows NOTHING                 (registration order, synchronous)
        about who is listening
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** An order raises a `confirmed` event; a mailer reacts.

**The WRONG Way (Anti-Pattern):**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    METHODS confirm.
  PRIVATE SECTION.
    DATA mo_mailer TYPE REF TO lcl_mailer.   " publisher KNOWS the subscriber
ENDCLASS.
CLASS lcl_order IMPLEMENTATION.
  METHOD confirm.
    mo_mailer->send( 'confirmed' ).          " hard-coded reaction
    " ...add audit? edit this method again...
  ENDMETHOD.
ENDCLASS.
```
**Problems with this code:**
- Every new reaction edits `confirm`.
- The order depends on the mailer (and soon auditor, dashboard…).

**The RIGHT Way (Following Best Practice):**
```abap
CLASS lcl_order DEFINITION.
  PUBLIC SECTION.
    EVENTS confirmed EXPORTING VALUE(ev_order_id) TYPE string.   " just announces
    METHODS confirm.
  PRIVATE SECTION.
    DATA mv_id TYPE string VALUE 'ORD-1'.
ENDCLASS.
CLASS lcl_order IMPLEMENTATION.
  METHOD confirm.
    RAISE EVENT confirmed EXPORTING ev_order_id = mv_id.   " no knowledge of handlers
  ENDMETHOD.
ENDCLASS.

CLASS lcl_mailer DEFINITION.
  PUBLIC SECTION.
    METHODS on_confirmed FOR EVENT confirmed OF lcl_order
                         IMPORTING ev_order_id sender.    " reacts
ENDCLASS.
CLASS lcl_mailer IMPLEMENTATION.
  METHOD on_confirmed.
    cl_demo_output=>write( |Mail: order { ev_order_id } confirmed| ).
  ENDMETHOD.
ENDCLASS.

" Wire them together from OUTSIDE the order:
DATA(lo_order)  = NEW lcl_order( ).
DATA(lo_mailer) = NEW lcl_mailer( ).
SET HANDLER lo_mailer->on_confirmed FOR lo_order.    " register
lo_order->confirm( ).                                 " raises → mailer reacts
```
**Why this is better:**
- The order announces; it has no reference to the mailer.
- Adding a reaction = a new handler + one `SET HANDLER`, no order change.

**Step-by-Step Explanation:**
- `EVENTS confirmed EXPORTING ev_order_id` — declares the signal and its payload.
- `RAISE EVENT confirmed EXPORTING …` — fires it; registered handlers run.
- `METHODS on_confirmed FOR EVENT confirmed OF lcl_order IMPORTING …` — the reaction; `sender` is the raising order.
- `SET HANDLER lo_mailer->on_confirmed FOR lo_order.` — connects subscriber to publisher.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** Multiple independent subscribers react to one confirmation; one is later deactivated — all without editing the order.

```abap
CLASS lcl_auditor DEFINITION.
  PUBLIC SECTION.
    METHODS on_confirmed FOR EVENT confirmed OF lcl_order IMPORTING ev_order_id.
ENDCLASS.
CLASS lcl_auditor IMPLEMENTATION.
  METHOD on_confirmed.
    cl_demo_output=>write( |Audit: { ev_order_id } at { sy-uzeit }| ).
  ENDMETHOD.
ENDCLASS.

DATA(lo_order)   = NEW lcl_order( ).
DATA(lo_mailer)  = NEW lcl_mailer( ).
DATA(lo_auditor) = NEW lcl_auditor( ).

" Two subscribers, registered independently:
SET HANDLER lo_mailer->on_confirmed  FOR lo_order.
SET HANDLER lo_auditor->on_confirmed FOR lo_order.

lo_order->confirm( ).        " BOTH react, in registration order

" Later: stop auditing without touching the order or the mailer:
SET HANDLER lo_auditor->on_confirmed FOR lo_order ACTIVATION abap_false.
lo_order->confirm( ).        " now only the mailer reacts
```

**Detailed Walkthrough:**
- **Independent registration** — each subscriber opts in separately; the order is unaware of the count.
- **Registration order** — handlers run in the order registered, synchronously, in the same process.
- **`ACTIVATION abap_false`** — deregisters a single handler at runtime.

**How This Works in Practice.** This is Observer at work: the domain object emits a meaningful event; cross-cutting concerns (mail, audit, dashboards) subscribe at the edges. New concerns plug in without disturbing the core.

**Why This Implementation.** Keeping reactions out of the publisher keeps the domain class focused and testable; the wiring lives in composition root / application code.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Expecting events to be asynchronous or transactional.**
```abap
RAISE EVENT confirmed.    " assuming this returns immediately and handlers run "later"
```
**Why this is wrong:** `RAISE EVENT` is **synchronous** — it calls all handlers in sequence before returning, in the same LUW/process. A slow or failing handler blocks/affects the publisher.
**Correct approach:** keep handlers fast and side-effect-light; for true async/decoupled processing use background tasks, bgPF/RAP events, or message queues — not ABAP class events alone.

**Mistake #2: Handler signature mismatch.**
```abap
METHODS on_confirmed FOR EVENT confirmed OF lcl_order
                     IMPORTING ev_wrong_name.    " name not declared on the event
```
**Why this is wrong:** a handler may import only the event's declared `EXPORTING` parameters (and `sender`); unknown names fail.
**Correct approach:** import exactly the event's parameters (here `ev_order_id`) and optionally `sender`.

---

## 8. Comparison With Similar Concepts

**Event vs direct method call:** a direct call couples caller to callee and runs exactly one reaction; an event decouples them and fans out to any number of registered handlers. Use events when the set of reactions varies or shouldn't be known by the source.

**Instance event vs static event (`CLASS-EVENTS`):** an instance event is tied to a specific object (handlers see `sender`); a static event is raised at class level (no instance `sender`). Use static events for class-wide signals.

**Events vs Observer-by-interface:** you can also implement Observer manually with a list of `if_observer` references the publisher notifies. ABAP events are the language-native form; the manual form gives more control (e.g. ordering, error isolation) at the cost of boilerplate.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **ALV / GUI controls (`CL_GUI_*`):** classic event usage — `SET HANDLER` on toolbar/double-click events.
- **RAP:** has its own (decoupled, transactional) business-events concept distinct from class events; don't conflate them.
- **Workflow / event linkage:** SAP also has system-level events (SWE) — a different mechanism from ABAP OO class events.

**SAP-Specific Considerations:** class events are synchronous and in-process — they do **not** cross sessions, work processes, or the database commit boundary. Handler exceptions propagate to the raiser unless caught. For cross-system or guaranteed delivery, use the appropriate async framework, not class events.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: heavy or failure-prone work inside a handler.**
```abap
METHOD on_confirmed.
  COMMIT WORK.                 " a handler doing transactional/long work
  cl_http_client=>...send().   " remote call inside a synchronous event
ENDMETHOD.
```
**Why this fails:** synchronous handlers block the publisher and can break its flow with exceptions or commits.
**Correct approach:** keep handlers small; enqueue or defer heavy work.

**Common Gotcha:** handlers are de-registered automatically only when the *handler* object is garbage-collected; an unbound publisher with no remaining references also stops firing. Forgotten `SET HANDLER` registrations can keep objects alive (memory) and cause surprising double reactions if registered twice.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you add a new reaction to "order confirmed" without editing `lcl_order`? If you must edit the publisher, you're not using events for decoupling.

**Unit Test Example:**
```abap
" A recording handler used as a test spy:
CLASS ltd_spy DEFINITION.
  PUBLIC SECTION.
    DATA mv_seen TYPE string.
    METHODS on_confirmed FOR EVENT confirmed OF lcl_order IMPORTING ev_order_id.
ENDCLASS.
CLASS ltd_spy IMPLEMENTATION.
  METHOD on_confirmed.
    mv_seen = ev_order_id.
  ENDMETHOD.
ENDCLASS.

CLASS ltcl_order DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS confirm_raises_event FOR TESTING.
ENDCLASS.
CLASS ltcl_order IMPLEMENTATION.
  METHOD confirm_raises_event.
    DATA(lo_order) = NEW lcl_order( ).
    DATA(lo_spy)   = NEW ltd_spy( ).
    SET HANDLER lo_spy->on_confirmed FOR lo_order.

    lo_order->confirm( ).

    cl_abap_unit_assert=>assert_equals( act = lo_spy->mv_seen exp = 'ORD-1' ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** a spy handler records the event payload, proving `confirm( )` raised `confirmed` with the right data — testing the publisher without any real subscriber.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Events decouple publisher from subscribers: declare `EVENTS`, `RAISE EVENT`, handle `FOR EVENT … OF`, wire with `SET HANDLER`.
- Class events are **synchronous and in-process**; keep handlers fast and exception-safe.
- The publisher never references its handlers — new reactions need no publisher change (Observer / Open–Closed).

**When to Apply:** "something happened, and an open-ended set of parties may care" — domain notifications, UI control callbacks.

**Red Flags:** assuming async/transactional behaviour; heavy work in handlers; publisher holding subscriber references; handler signature mismatches; duplicate `SET HANDLER` registrations.

---

## 13. Dependency Map

**Depends On:**
- `02_03_methods.md` — handlers and event parameters follow method-parameter rules.
- `02_06_interfaces_in_abap.md` — the alternative interface-based Observer.

**Enables:**
- `03_01_solid_principles.md` — events are a route to Open–Closed for reactions.
- Decoupled UI and domain-notification designs.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "EVENTS", "CLASS-EVENTS", "RAISE EVENT", "SET HANDLER", "Event Handler Methods", "FOR EVENT".

**Design Patterns & Best Practices:** Pattern reference: Observer (Gang of Four). Clean ABAP → keep methods/handlers focused (`github.com/SAP/styleguides`).
