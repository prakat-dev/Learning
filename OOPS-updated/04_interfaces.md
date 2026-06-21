# 04 — Interfaces

> Pure contracts. The foundation of decoupling, ABAP's form of "multiple inheritance", and the BADI framework.

---

## What an interface is

An interface declares method signatures, constants, types, events, and data — but contains **no implementation**. Any class that implements it must provide code for every method. All interface components are **PUBLIC** by definition.

```abap
INTERFACE zif_printable.
  METHODS: print RETURNING VALUE(rv_output) TYPE string.
ENDINTERFACE.
```

---

## Implementing an interface

```abap
CLASS zcl_report DEFINITION.
  PUBLIC SECTION.
    INTERFACES zif_printable.
ENDCLASS.

CLASS zcl_report IMPLEMENTATION.
  METHOD zif_printable~print.        " interface method uses the ~ notation
    rv_output = 'Report content'.
  ENDMETHOD.
ENDCLASS.
```

You can call the method through either an object reference or an interface reference:

```abap
DATA(lo_report) = NEW zcl_report( ).
DATA(lv1) = lo_report->zif_printable~print( ).   " via object reference

DATA: li_print TYPE REF TO zif_printable.
li_print = lo_report.                            " upcast to interface
DATA(lv2) = li_print->print( ).                  " via interface reference
```

---

## Multiple interfaces — ABAP's "multiple inheritance"

A class may inherit from only one parent, but it can implement **many interfaces**. This is how you combine several capabilities in one class.

```abap
CLASS zcl_channel_idoc DEFINITION.
  PUBLIC SECTION.
    INTERFACES: zif_sender,
                zif_loggable,
                zif_transportable.
ENDCLASS.

CLASS zcl_channel_idoc IMPLEMENTATION.
  METHOD zif_sender~send.        " ... ENDMETHOD.
  METHOD zif_loggable~write_log. " ... ENDMETHOD.
  METHOD zif_transportable~add_to_transport. " ... ENDMETHOD.
ENDCLASS.

" One object usable wherever any of those interfaces is expected
DATA(lo_idoc) = NEW zcl_channel_idoc( ).
DATA: li_sender TYPE REF TO zif_sender.
li_sender = lo_idoc.   " works - the class implements zif_sender
```

---

## ALIASES

Aliases give a short name to an interface method and can also set its external visibility.

```abap
CLASS zcl_channel_idoc DEFINITION.
  PUBLIC SECTION.
    INTERFACES zif_sender.
    ALIASES: send FOR zif_sender~send.
ENDCLASS.

" Now you can call the short form:
DATA(lv_ok) = lo_idoc->send( ls_payload ).   " instead of lo_idoc->zif_sender~send( )
```

Declaring an alias in `PROTECTED SECTION` makes that interface method protected to external callers, even though interface methods are public by default.

---

## Interface constants and types

Interfaces can hold constants and type definitions usable **without implementing** the interface — a very common SAP pattern for shared enumerations.

```abap
INTERFACE zif_doc_status.
  CONSTANTS: c_draft     TYPE char2 VALUE '01',
             c_submitted TYPE char2 VALUE '02',
             c_approved  TYPE char2 VALUE '03',
             c_rejected  TYPE char2 VALUE '04'.
  TYPES: ty_doc_id TYPE c LENGTH 10.
ENDINTERFACE.

" Access the constant directly via the interface name:
IF ls_doc-status = zif_doc_status=>c_approved.
  " proceed
ENDIF.
```

---

## Interface composition (nesting)

An interface can include other interfaces. A class implementing the outer interface must implement all nested methods too.

```abap
INTERFACE zif_full_channel.
  INTERFACES: zif_sender,
              zif_loggable.
  METHODS: get_channel_name RETURNING VALUE(rv_name) TYPE string.
ENDINTERFACE.

" A class implementing zif_full_channel must implement
" zif_sender~send, zif_loggable~write_log, and zif_full_channel~get_channel_name.
```

---

## A note on "default methods"

ABAP interfaces have no default-method mechanism (unlike modern Java). If you need a shared default implementation alongside a contract, use an **abstract class** instead (File 03), or implement the method in each class.

---

## Link to BADIs

BADIs are built on interfaces. Defining a BADI generates an interface (for example `IF_EX_ME_PROCESS_PO_CUST`); your implementation class implements that interface.

```abap
CLASS zcl_badi_po_check DEFINITION.
  PUBLIC SECTION.
    INTERFACES if_ex_me_process_po_cust.
ENDCLASS.
```

The enhancement framework / `CL_EXITHANDLER` finds your implementation **through the interface**, never by class name. Understanding interfaces is understanding BADIs.

---

## Interface vs Abstract class — when to use

| Use an Interface when | Use an Abstract class when |
|-----------------------|----------------------------|
| You need only a contract, no shared code | You need a contract **plus** shared implementation |
| A class must fulfill several contracts | You have a single "is-a" hierarchy with common logic |
| Building a plugin / BADI architecture | Implementing a template-method workflow |
| Decoupling otherwise unrelated classes | Reusing base behavior across subclasses |

---

## Quick self-test

1. What is the default visibility of interface methods?
2. How does a class implement multiple interfaces, and why can it not inherit from multiple classes?
3. What two things do `ALIASES` give you?
4. How do you read an interface constant without implementing the interface?
5. How are BADIs related to interfaces?
6. ABAP interfaces have no default methods — what do you use instead when you need shared default logic?
