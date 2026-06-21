# 02 — The Four Pillars of OOP (in ABAP)

> The "why OOP" file. Interviewers almost always open here. Know how each pillar maps to a concrete ABAP feature, with a real example for each.

---

## Overview

| Pillar | One-line meaning | ABAP mechanism |
|--------|------------------|----------------|
| Encapsulation | Hide internal state, expose a controlled interface | Visibility sections, getters/setters |
| Abstraction | Expose essential behavior, hide the implementation | Interfaces, abstract classes |
| Inheritance | Reuse and extend an existing class | `INHERITING FROM`, `REDEFINITION` |
| Polymorphism | One call, many behaviors depending on the object | Interfaces, redefinition, casting |

---

## 1. Encapsulation

**Bundle data and behavior together, and control access so the object can never be put into an invalid state.** Data is kept private; all access goes through public methods that enforce the rules.

```abap
CLASS zcl_account DEFINITION.
  PUBLIC SECTION.
    METHODS: deposit  IMPORTING iv_amount TYPE dmbtr
                      RAISING   zcx_invalid_amount,
             withdraw IMPORTING iv_amount TYPE dmbtr
                      RAISING   zcx_invalid_amount
                                zcx_insufficient_funds,
             get_balance RETURNING VALUE(rv_balance) TYPE dmbtr.
  PRIVATE SECTION.
    DATA: mv_balance TYPE dmbtr.   " cannot be touched directly from outside
ENDCLASS.

CLASS zcl_account IMPLEMENTATION.
  METHOD deposit.
    IF iv_amount <= 0.
      RAISE EXCEPTION TYPE zcx_invalid_amount.
    ENDIF.
    mv_balance = mv_balance + iv_amount.
  ENDMETHOD.

  METHOD withdraw.
    IF iv_amount <= 0.
      RAISE EXCEPTION TYPE zcx_invalid_amount.
    ENDIF.
    IF iv_amount > mv_balance.
      RAISE EXCEPTION TYPE zcx_insufficient_funds.
    ENDIF.
    mv_balance = mv_balance - iv_amount.
  ENDMETHOD.

  METHOD get_balance.
    rv_balance = mv_balance.
  ENDMETHOD.
ENDCLASS.
```

Because `mv_balance` is private, no caller can write `lo_account->mv_balance = -9999`. Every change is forced through the validation in `deposit`/`withdraw`. The object guards its own invariants.

---

## 2. Abstraction

**Expose what an object does, hide how it does it.** The caller programs against a simplified view — an interface or abstract method — and never depends on the concrete implementation.

```abap
INTERFACE zif_sender.
  METHODS: send IMPORTING is_payload TYPE zorder_payload
                RETURNING VALUE(rv_ok) TYPE abap_bool.
ENDINTERFACE.

CLASS zcl_sender_rest DEFINITION.
  PUBLIC SECTION.
    INTERFACES zif_sender.
ENDCLASS.

CLASS zcl_sender_rest IMPLEMENTATION.
  METHOD zif_sender~send.
    " HTTP POST via cl_http_client - details hidden from the caller
    rv_ok = abap_true.
  ENDMETHOD.
ENDCLASS.

" The caller only knows "I can send a payload" - not whether it is REST, IDoc, or RFC
DATA: li_sender TYPE REF TO zif_sender.
li_sender = NEW zcl_sender_rest( ).
DATA(lv_ok) = li_sender->send( ls_payload ).
```

Encapsulation hides *data*; abstraction hides *implementation/complexity*. They are related but distinct.

---

## 3. Inheritance

**A subclass reuses and extends a superclass — an "is-a" relationship.** It inherits the parent's public and protected members and may override methods with `REDEFINITION`.

```abap
CLASS zcl_employee DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor   IMPORTING iv_base TYPE dmbtr,
             get_salary    RETURNING VALUE(rv_salary) TYPE dmbtr.
  PROTECTED SECTION.
    DATA: mv_base TYPE dmbtr.
ENDCLASS.

CLASS zcl_employee IMPLEMENTATION.
  METHOD constructor.
    mv_base = iv_base.
  ENDMETHOD.
  METHOD get_salary.
    rv_salary = mv_base.
  ENDMETHOD.
ENDCLASS.


CLASS zcl_manager DEFINITION INHERITING FROM zcl_employee.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING iv_base TYPE dmbtr iv_bonus TYPE dmbtr,
             get_salary  REDEFINITION.
  PRIVATE SECTION.
    DATA: mv_bonus TYPE dmbtr.
ENDCLASS.

CLASS zcl_manager IMPLEMENTATION.
  METHOD constructor.
    super->constructor( iv_base = iv_base ).   " mandatory - init the parent first
    mv_bonus = iv_bonus.
  ENDMETHOD.
  METHOD get_salary.
    rv_salary = mv_base + mv_bonus.   " mv_base is inherited (protected)
  ENDMETHOD.
ENDCLASS.
```

ABAP supports **single inheritance only** — one parent per class. For multiple contracts, use interfaces (File 04). Favor composition over inheritance unless the "is-a" relationship is genuine and stable.

---

## 4. Polymorphism

**The same method call produces different behavior depending on the actual object behind the reference.** Achieved through redefinition (inheritance) or interface implementation.

```abap
DATA: lt_employees TYPE STANDARD TABLE OF REF TO zcl_employee.

APPEND NEW zcl_employee( iv_base = '3000' )                      TO lt_employees.
APPEND NEW zcl_manager(  iv_base = '5000' iv_bonus = '2000' )    TO lt_employees.

LOOP AT lt_employees INTO DATA(lo_emp).
  " each object runs its OWN get_salary - no CASE on type needed
  WRITE: / lo_emp->get_salary( ).
ENDLOOP.
" Output: 3000 (employee), 7000 (manager)
```

Add a new subclass (e.g. `zcl_director`) and this loop never changes. This is the engine behind BADIs, factories, and plugin frameworks in SAP.

---

## How they work together

A typical custom posting framework uses all four pillars at once:

- **Abstraction** — an abstract base `zcl_posting` declares `validate( )` and `post( )`.
- **Inheritance** — `zcl_posting_fi`, `zcl_posting_mm` extend it and supply specifics.
- **Encapsulation** — BAPI structures stay private; callers see only clean methods.
- **Polymorphism** — calling code holds a `zcl_posting` reference and runs whichever subclass logic is appropriate.

---

## Quick self-test

1. Encapsulation vs abstraction — what is the precise difference?
2. Which keywords implement inheritance and method overriding?
3. How many parents can an ABAP class have? How do you get multiple contracts?
4. Name the two ways to achieve polymorphism in ABAP.
5. In the polymorphism example, why is there no `CASE` on the object type?
6. Give a single SAP scenario that exercises all four pillars.
