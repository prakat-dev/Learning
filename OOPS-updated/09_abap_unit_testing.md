# 09 — ABAP Unit Testing & Test Doubles

> Automated developer tests, the Test Double Framework, and how to design OO code that is testable.

---

## ABAP Unit basics

Test code lives in a **local** test class marked `FOR TESTING`, usually in the test include of a global class. Tests run with Ctrl+Shift+F10 in ADT (or from SE80) and are never executed in production.

```abap
CLASS ltc_calc DEFINITION FINAL FOR TESTING
  DURATION SHORT
  RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    DATA: mo_cut TYPE REF TO zcl_calc.    " CUT = Class Under Test
    METHODS: setup,                        " runs before EACH test method
             test_add            FOR TESTING,
             test_divide_by_zero FOR TESTING.
ENDCLASS.

CLASS ltc_calc IMPLEMENTATION.
  METHOD setup.
    mo_cut = NEW zcl_calc( ).
  ENDMETHOD.

  METHOD test_add.
    cl_abap_unit_assert=>assert_equals(
      exp = 5
      act = mo_cut->add( iv_a = 2 iv_b = 3 ) ).
  ENDMETHOD.

  METHOD test_divide_by_zero.
    TRY.
        mo_cut->divide( iv_num = 1 iv_den = 0 ).
        cl_abap_unit_assert=>fail( 'Expected exception was not raised' ).
      CATCH zcx_division_by_zero.
        " expected path - test passes
    ENDTRY.
  ENDMETHOD.
ENDCLASS.
```

---

## Test class attributes

| Setting | Values | Meaning |
|---------|--------|---------|
| `DURATION` | SHORT / MEDIUM / LONG | Expected runtime budget |
| `RISK LEVEL` | HARMLESS / DANGEROUS / CRITICAL | DANGEROUS/CRITICAL may change data; the system must be configured to allow them |

`FINAL` prevents the test class being inherited. Fixture methods, by scope: `class_setup` (once before all tests), `setup` (before each test), `teardown` (after each test), `class_teardown` (once after all tests).

---

## Assertions — CL_ABAP_UNIT_ASSERT

```abap
cl_abap_unit_assert=>assert_equals(     exp = 5 act = lv_result ).
cl_abap_unit_assert=>assert_initial(    act = lv_value ).
cl_abap_unit_assert=>assert_not_initial( act = lo_obj ).
cl_abap_unit_assert=>assert_true(       act = lv_flag ).
cl_abap_unit_assert=>assert_bound(      act = lo_obj ).
cl_abap_unit_assert=>assert_char_cp(    act = lx->get_text( ) exp = '*not found*' ).
cl_abap_unit_assert=>fail(              msg = 'Should not reach here' ).
```

---

## The dependency problem

A class that creates its own database or RFC dependency cannot be tested in isolation — the test would hit the real system. The solution is to depend on an **interface** and inject the implementation, so a **test double** can be injected in tests.

```abap
CLASS zcl_order_processor DEFINITION.
  PUBLIC SECTION.
    METHODS: constructor IMPORTING ii_repo TYPE REF TO zif_order_repository,
             count_open_orders RETURNING VALUE(rv_count) TYPE i.
  PRIVATE SECTION.
    DATA: mi_repo TYPE REF TO zif_order_repository.   " interface, not concrete
ENDCLASS.

CLASS zcl_order_processor IMPLEMENTATION.
  METHOD constructor.
    mi_repo = ii_repo.
  ENDMETHOD.
  METHOD count_open_orders.
    rv_count = lines( mi_repo->get_open_orders( ) ).
  ENDMETHOD.
ENDCLASS.
```

---

## Test Double Framework (CL_ABAP_TESTDOUBLE)

Available since Basis 7.40 SP8. It generates a double for any **global interface** automatically — no hand-written stub class needed.

```abap
CLASS ltc_processor DEFINITION FINAL FOR TESTING
  DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    DATA: mo_cut         TYPE REF TO zcl_order_processor,
          mi_repo_double TYPE REF TO zif_order_repository.
    METHODS: setup,
             test_count FOR TESTING.
ENDCLASS.

CLASS ltc_processor IMPLEMENTATION.
  METHOD setup.
    " create a double for the interface
    mi_repo_double ?= cl_abap_testdouble=>create( 'ZIF_ORDER_REPOSITORY' ).
    " inject it into the class under test
    mo_cut = NEW zcl_order_processor( ii_repo = mi_repo_double ).
  ENDMETHOD.

  METHOD test_count.
    DATA(lt_fake) = VALUE ztt_order( ( vbeln = '1' ) ( vbeln = '2' ) ).

    " configure: the NEXT call to get_open_orders returns lt_fake
    cl_abap_testdouble=>configure_call( mi_repo_double )->returning( lt_fake ).
    mi_repo_double->get_open_orders( ).   " this call arms the stub

    " act + assert
    cl_abap_unit_assert=>assert_equals(
      exp = 2
      act = mo_cut->count_open_orders( ) ).
  ENDMETHOD.
ENDCLASS.
```

Configuration options on `configure_call`: `returning( value )` for a canned result, `set_parameter( ... )` to fill exporting/changing parameters, `raise_exception( lx )` to make the double throw. Use `cl_abap_testdouble=>verify_expectations( )` together with `times( n )` to assert how often a method was called (mock-style verification).

---

## Types of test doubles (Meszaros terminology)

| Type | Behavior |
|------|----------|
| **Dummy** | Passed to fill a parameter slot, never actually used |
| **Stub** | Returns canned answers for queries (read dependencies) |
| **Mock** | Pre-programmed with expectations; the test fails if they are not met |
| **Fake** | A working but shortcut implementation (e.g. an in-memory table instead of the DB) |
| **Spy** | Records how it was called so the test can inspect it afterwards |

---

## Injection mechanisms

The cleanest is **constructor injection** (pass the dependency to `constructor`). Alternatives are **setter injection** (a `set_repo( )` method), **parameter injection** (pass it into the specific method), and **back-door injection** via a friended test class for legacy code that you cannot easily refactor.

---

## Designing testable OO code

Program to interfaces rather than concrete classes, and inject dependencies instead of creating them with `NEW` inside the class. Keep methods small and single-purpose, and separate business logic from database, RFC, and UI access so the logic can be tested without those systems. Write one test method per scenario (happy path, each error, edge cases), and run the suite in your CI / transport pipeline.

---

## Quick self-test

1. What do `DURATION` and `RISK LEVEL` control?
2. Which fixture method runs before each test — `setup` or `class_setup`?
3. Why depend on an interface instead of a concrete class?
4. What does `cl_abap_testdouble=>create( )` need in order to build a double?
5. Stub vs Mock — what is the key difference?
6. Name three dependency-injection mechanisms.
7. How do you assert that an expected exception was raised?
