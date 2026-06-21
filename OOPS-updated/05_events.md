# 05 — Events

> The observer pattern, built into the language. Events decouple the object that raises them from whatever reacts.

---

## Core idea

A class **raises** an event; one or more **handler** methods react. The raiser does not know who is listening — handlers register themselves at runtime with `SET HANDLER`. This gives loose coupling and easy extensibility.

---

## Declaring and raising an event

Event parameters are always `EXPORTING` from the raiser's perspective and use `VALUE( )`.

```abap
CLASS zcl_order DEFINITION.
  PUBLIC SECTION.
    EVENTS: order_created EXPORTING VALUE(ev_order_id) TYPE vbeln.
    METHODS: create IMPORTING iv_customer TYPE kunnr.
ENDCLASS.

CLASS zcl_order IMPLEMENTATION.
  METHOD create.
    DATA(lv_new_id) = CONV vbeln( '0000005678' ).
    " ... create the order in the database ...
    RAISE EVENT order_created EXPORTING ev_order_id = lv_new_id.
  ENDMETHOD.
ENDCLASS.
```

---

## Declaring a handler

A handler method maps the event's `EXPORTING` parameters to its own `IMPORTING` parameters. The implicit `sender` parameter (instance events only) tells the handler which object raised it.

```abap
CLASS zcl_email_notifier DEFINITION.
  PUBLIC SECTION.
    METHODS: on_order_created
               FOR EVENT order_created OF zcl_order
               IMPORTING ev_order_id sender.
ENDCLASS.

CLASS zcl_email_notifier IMPLEMENTATION.
  METHOD on_order_created.
    WRITE: / 'Email sent for order', ev_order_id.
    " 'sender' is a reference to the zcl_order that raised the event
  ENDMETHOD.
ENDCLASS.
```

---

## Registering with SET HANDLER

```abap
DATA(lo_order)    = NEW zcl_order( ).
DATA(lo_notifier) = NEW zcl_email_notifier( ).

SET HANDLER lo_notifier->on_order_created FOR lo_order.

lo_order->create( iv_customer = '0001000001' ).   " fires event -> handler runs
```

- `FOR lo_order` — handle events from this specific object.
- `FOR ALL INSTANCES` — handle events from every object of that class.
- `ACTIVATION abap_false` — deregister a previously set handler.

```abap
SET HANDLER lo_notifier->on_order_created FOR ALL INSTANCES.
SET HANDLER lo_notifier->on_order_created FOR lo_order ACTIVATION abap_false.
```

---

## Multiple handlers

Several handlers can listen to the same event; all registered handlers run when it fires (execution order is not guaranteed).

```abap
DATA(lo_email)     = NEW zcl_email_notifier( ).
DATA(lo_audit)     = NEW zcl_audit_logger( ).
DATA(lo_inventory) = NEW zcl_inventory_updater( ).

SET HANDLER lo_email->on_order_created     FOR lo_order.
SET HANDLER lo_audit->on_order_created     FOR lo_order.
SET HANDLER lo_inventory->on_order_created FOR lo_order.

lo_order->create( iv_customer = '0001000001' ).   " triggers all three reactions
```

The key win: to add a new reaction you add a handler — you never modify `zcl_order`.

---

## Static vs instance events

| | Instance event (`EVENTS`) | Static event (`CLASS-EVENTS`) |
|--|---------------------------|-------------------------------|
| Raised in | instance method | static or instance method |
| `sender` parameter | available | not available |
| Registered with | `FOR <object>` or `FOR ALL INSTANCES` | `FOR <class>` |

---

## Events vs direct method calls

| Direct call | Event |
|-------------|-------|
| Raiser holds references to every receiver | Raiser knows nothing about receivers |
| Add a reaction → modify the raiser | Add a reaction → add a handler |
| Tight coupling | Loose coupling |
| Fine for simple 1:1 calls | Ideal for 1:many, extensible reactions |

**SAP example:** posting a goods receipt triggers accounting document creation, stock update, quality inspection, warehouse notification, and serial-number processing. Each is an independent handler; the goods-receipt logic does not call any of them directly.

---

## Quick self-test

1. From the raiser's side, are event parameters EXPORTING or IMPORTING? From the handler's side?
2. What does `FOR ALL INSTANCES` do?
3. How do you deregister a handler?
4. What is the `sender` parameter, and for which event type is it available?
5. Why are events preferable to direct calls for one-to-many reactions?
6. Name two differences between a static event and an instance event.
