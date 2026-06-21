# 07 — Design Patterns in ABAP

> Proven solutions to recurring design problems, each with a complete, SAP-grounded example.

---

## Singleton

**Exactly one instance per session.** Three pieces are required together: `CREATE PRIVATE` to block external `NEW`, a static accessor, and a static reference that holds the single instance.

```abap
CLASS zcl_logger DEFINITION CREATE PRIVATE.
  PUBLIC SECTION.
    CLASS-METHODS: get_instance RETURNING VALUE(ro_logger) TYPE REF TO zcl_logger.
    METHODS: log IMPORTING iv_msg TYPE string,
             get_messages RETURNING VALUE(rt_msg) TYPE string_table.
  PRIVATE SECTION.
    CLASS-DATA: go_instance TYPE REF TO zcl_logger.
    DATA: mt_messages TYPE string_table.
ENDCLASS.

CLASS zcl_logger IMPLEMENTATION.
  METHOD get_instance.
    IF go_instance IS NOT BOUND.
      go_instance = NEW zcl_logger( ).   " created only on first call
    ENDIF.
    ro_logger = go_instance.
  ENDMETHOD.

  METHOD log.
    APPEND iv_msg TO mt_messages.
  ENDMETHOD.

  METHOD get_messages.
    rt_msg = mt_messages.
  ENDMETHOD.
ENDCLASS.

" Usage - every caller in the session shares the same logger object
zcl_logger=>get_instance( )->log( 'Step 1 done' ).
zcl_logger=>get_instance( )->log( 'Step 2 done' ).
DATA(lt_all) = zcl_logger=>get_instance( )->get_messages( ).   " both messages
```

> `CREATE PRIVATE` alone is not a singleton — it only blocks external `NEW`. All three pieces together enforce a single instance.

**SAP use:** application logger, configuration reader, RFC destination/connection manager.

---

## Factory

**Encapsulate object creation.** The caller states what it needs; the factory returns the right concrete class behind an interface, so the caller never names a concrete class.

```abap
CLASS zcl_channel_factory DEFINITION.
  PUBLIC SECTION.
    CLASS-METHODS: create IMPORTING iv_type TYPE string
                          RETURNING VALUE(ri_sender) TYPE REF TO zif_sender
                          RAISING   zcx_unknown_channel.
ENDCLASS.

CLASS zcl_channel_factory IMPLEMENTATION.
  METHOD create.
    CASE iv_type.
      WHEN 'IDOC'. ri_sender = NEW zcl_channel_idoc( ).
      WHEN 'REST'. ri_sender = NEW zcl_channel_rest( ).
      WHEN 'RFC'.  ri_sender = NEW zcl_channel_rfc( ).
      WHEN OTHERS. RAISE EXCEPTION TYPE zcx_unknown_channel.
    ENDCASE.
  ENDMETHOD.
ENDCLASS.

" Usage - caller depends only on the interface and the factory
TRY.
    DATA(li_sender) = zcl_channel_factory=>create( 'REST' ).
    li_sender->send( ls_payload ).
  CATCH zcx_unknown_channel.
    " unsupported channel type
ENDTRY.
```

**SAP use:** payment processors, output channels, document creators.

---

## Strategy

**Inject the algorithm as an interface so it can be swapped at runtime.**

```abap
INTERFACE zif_discount.
  METHODS: calculate IMPORTING iv_amount TYPE dmbtr
                     RETURNING VALUE(rv_discount) TYPE dmbtr.
ENDINTERFACE.

CLASS zcl_discount_seasonal DEFINITION.
  PUBLIC SECTION.
    INTERFACES zif_discount.
ENDCLASS.
CLASS zcl_discount_seasonal IMPLEMENTATION.
  METHOD zif_discount~calculate.
    rv_discount = iv_amount * '0.10'.   " 10% off
  ENDMETHOD.
ENDCLASS.

CLASS zcl_pricing DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING ii_discount TYPE REF TO zif_discount,
             final_price IMPORTING iv_amount TYPE dmbtr
                         RETURNING VALUE(rv_price) TYPE dmbtr.
  PRIVATE SECTION.
    DATA: mi_discount TYPE REF TO zif_discount.
ENDCLASS.
CLASS zcl_pricing IMPLEMENTATION.
  METHOD constructor.
    mi_discount = ii_discount.
  ENDMETHOD.
  METHOD final_price.
    rv_price = iv_amount - mi_discount->calculate( iv_amount ).
  ENDMETHOD.
ENDCLASS.

" Inject whichever strategy you want - swap without touching zcl_pricing
DATA(lo_pricing) = NEW zcl_pricing( ii_discount = NEW zcl_discount_seasonal( ) ).
DATA(lv_price)   = lo_pricing->final_price( iv_amount = '100.00' ).   " 90.00
```

**SAP use:** different tax, discount, or rounding rules selected by configuration.

---

## Template Method

**Fix the sequence of steps in an abstract base class; let subclasses supply the varying steps.** The orchestrating method is `FINAL` so subclasses cannot change the sequence — only the individual steps.

```abap
CLASS zcl_posting DEFINITION ABSTRACT.
  PUBLIC SECTION.
    METHODS: post FINAL RETURNING VALUE(rv_success) TYPE abap_bool.
  PROTECTED SECTION.
    METHODS: validate   ABSTRACT RETURNING VALUE(rv_valid)   TYPE abap_bool,
             build_bapi ABSTRACT,
             call_bapi  ABSTRACT RETURNING VALUE(rv_success) TYPE abap_bool,
             commit_work,
             write_log.
ENDCLASS.

CLASS zcl_posting IMPLEMENTATION.
  METHOD post.                                  " the template - fixed sequence
    rv_success = abap_false.

    IF validate( ) = abap_false.                " step 1 (subclass-specific)
      write_log( ).
      RETURN.
    ENDIF.

    build_bapi( ).                              " step 2 (subclass-specific)

    IF call_bapi( ) = abap_true.                " step 3 (subclass-specific)
      commit_work( ).                           " step 4 (shared)
      rv_success = abap_true.
    ENDIF.

    write_log( ).                               " step 5 (shared)
  ENDMETHOD.

  METHOD commit_work.
    CALL FUNCTION 'BAPI_TRANSACTION_COMMIT' EXPORTING wait = abap_true.
  ENDMETHOD.

  METHOD write_log.
    " write collected BAPIRET2 messages to the application log
  ENDMETHOD.
ENDCLASS.
```

A concrete subclass fills in only the abstract steps:

```abap
CLASS zcl_posting_fi DEFINITION INHERITING FROM zcl_posting.
  PROTECTED SECTION.
    METHODS: validate   REDEFINITION,
             build_bapi REDEFINITION,
             call_bapi  REDEFINITION.
ENDCLASS.

CLASS zcl_posting_fi IMPLEMENTATION.
  METHOD validate.   rv_valid = abap_true.   ENDMETHOD.
  METHOD build_bapi. " fill BAPI_ACC_DOCUMENT_POST structures   ENDMETHOD.
  METHOD call_bapi.  rv_success = abap_true. ENDMETHOD.
ENDCLASS.

" Caller runs the whole fixed workflow with one call
DATA(lo_posting) = CAST zcl_posting( NEW zcl_posting_fi( ) ).
DATA(lv_ok) = lo_posting->post( ).
```

**SAP use:** posting frameworks, IDoc processors, output generators.

---

## Observer

**One subject, many observers reacting to its changes.** ABAP implements this natively with events (see File 05) — no manual subscriber list needed.

**SAP example:** goods receipt posting triggering accounting, stock, quality, and warehouse reactions, each a separate event handler.

---

## Model-View-Controller (MVC)

Separation of concerns into three roles: the **Model** holds business data and logic, the **View** renders output (ALV grid, Web Dynpro view, Fiori UI), and the **Controller** handles user input and coordinates the two. Web Dynpro ABAP is built on MVC; classic dynpro programs often emulate it with a controller class plus model classes.

---

## Dependency Injection (DI)

**Pass dependencies in rather than creating them internally.** This is what makes a class unit-testable — you can inject a mock in place of the real database or RFC dependency.

```abap
" Hard dependency - difficult to test, bound to the real repository
CLASS zcl_proc_bad DEFINITION.
  PUBLIC SECTION.
    METHODS run.
ENDCLASS.
CLASS zcl_proc_bad IMPLEMENTATION.
  METHOD run.
    DATA(lo_repo) = NEW zcl_order_repository( ).   " tightly coupled to DB
  ENDMETHOD.
ENDCLASS.

" Injected dependency - flexible and testable
CLASS zcl_proc_good DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING ii_repo TYPE REF TO zif_order_repository,
             run.
  PRIVATE SECTION.
    DATA: mi_repo TYPE REF TO zif_order_repository.
ENDCLASS.
CLASS zcl_proc_good IMPLEMENTATION.
  METHOD constructor.
    mi_repo = ii_repo.   " accepts any implementation, real or mock
  ENDMETHOD.
  METHOD run.
    DATA(lt_orders) = mi_repo->get_open_orders( ).
  ENDMETHOD.
ENDCLASS.
```

**SAP use:** any class you want to unit test — inject the DB/RFC access as an interface and mock it in tests (see File 09).

---

## Quick self-test

1. What three pieces make a singleton, and why is `CREATE PRIVATE` not enough alone?
2. What does a factory hide from the caller?
3. Strategy vs Template Method — what is the key difference?
4. Why is the `post( )` template method marked `FINAL`?
5. Which ABAP language feature implements Observer natively?
6. What does Dependency Injection enable for testing?
