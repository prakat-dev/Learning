# Class-Based Exceptions

**Learning Objective:** After this topic you can design and use class-based exceptions — choosing the right base (`CX_STATIC_CHECK` / `CX_DYNAMIC_CHECK` / `CX_NO_CHECK`), raising and catching with `TRY`/`CATCH`/`CLEANUP`, attaching meaningful texts, and chaining causes — instead of relying on return codes.

**Difficulty Level:** Intermediate
**Time to Master:** 90 minutes
**Prerequisites:** `02_01_abap_class_syntax.md`, `02_07_inheritance_in_abap.md`
**Official Sources:**
- ABAP Keyword Documentation → *RAISE EXCEPTION*, *TRY*, *CATCH*, *CLEANUP*, *CX_ROOT*, *CX_STATIC_CHECK* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Error Handling* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A posting routine returns `sy-subrc`. Every caller is *supposed* to check it; many don't. A failed posting silently continues, and the error surfaces three steps later as corrupt data with no trace of the original cause.

**What Happens WITHOUT This Concept.** Return-code error handling is opt-in — callers can ignore it, and the *why* (the original message) is lost. Errors propagate as bad data rather than as a clear, catchable signal carrying context.

**Why This Matters in SAP.** Class-based exceptions make failure explicit and hard to ignore: a checked exception *must* be handled or declared, carries a rich message, and can chain to its root cause. This is the modern ABAP standard, replacing classic `RAISE`/`MESSAGE … RAISING`.

---

## 3. Core Concept Explanation

**Definition.** A **class-based exception** is an object (a subclass of `CX_ROOT`) that represents an error. You **raise** it to signal failure and **catch** it to handle failure. Three base categories control how strictly the compiler enforces handling:
- **`CX_STATIC_CHECK`** — must be caught or declared with `RAISING` (compile-time enforced). Use for foreseeable, recoverable errors.
- **`CX_DYNAMIC_CHECK`** — handling not enforced at compile time; unhandled → runtime check. Use where declaration would be noise but failure is possible.
- **`CX_NO_CHECK`** — never needs declaring/catching; propagates freely. Use for pervasive, usually-fatal conditions.

**Key Principles:**
- Choose the category by how the caller should treat the error.
- Raise with context; catch as specifically as possible; clean up resources with `CLEANUP`.
- Chain the original cause with `EXPORTING previous = …` so the root is never lost.

**How It Works.** `RAISE EXCEPTION TYPE zcx_… EXPORTING …` (or `RAISE EXCEPTION NEW zcx_…( … )`, 7.5+) creates and throws the object. `TRY. … CATCH zcx_… INTO DATA(lx). … ENDTRY.` handles it; `CLEANUP` runs if an exception leaves the `TRY` block. `lx->get_text( )` yields the message.

**Why It's Designed This Way.** Making errors first-class objects lets them carry data and cause chains; making the *static* category mandatory to handle prevents the silent-ignore failure mode of return codes.

---

## 4. Visual Representation

```
   CX_ROOT
     ├── CX_STATIC_CHECK     (must catch or declare RAISING)  ← recoverable, foreseeable
     │      └── ZCX_POSTING_FAILED (your custom exception)
     ├── CX_DYNAMIC_CHECK    (compiler lenient; runtime checked)
     └── CX_NO_CHECK         (propagates freely)               ← pervasive/fatal

   FLOW:
     TRY.
         risky( )  ──raise──►  ZCX_POSTING_FAILED( previous = root_cause )
       CATCH zcx_posting_failed INTO DATA(lx).   ← handle, lx->get_text( )
       CLEANUP.                                   ← runs if TRY exited via exception
     ENDTRY.
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Replace a return code with a checked exception.

**The WRONG Way (Anti-Pattern):**
```abap
METHODS post RETURNING VALUE(rv_subrc) TYPE sysubrc.   " caller MUST remember to check
" caller:
lo->post( ).                  " return value ignored → failure silently swallowed
" ... continues as if it worked ...
```
**Problems with this code:**
- The failure is ignorable; nothing forces the caller to react.
- No message, no cause — only a number.

**The RIGHT Way (Following Best Practice):**
```abap
CLASS zcx_posting_failed DEFINITION PUBLIC INHERITING FROM cx_static_check.
  PUBLIC SECTION.
    METHODS constructor IMPORTING textid   LIKE if_t100_message=>t100key OPTIONAL
                                  previous TYPE REF TO cx_root OPTIONAL
                                  iv_doc   TYPE string OPTIONAL.
    DATA mv_doc TYPE string.
ENDCLASS.
CLASS zcx_posting_failed IMPLEMENTATION.
  METHOD constructor.
    super->constructor( textid = textid previous = previous ).
    mv_doc = iv_doc.
  ENDMETHOD.
ENDCLASS.

CLASS zcl_poster DEFINITION PUBLIC CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS post IMPORTING iv_doc TYPE string
                 RAISING   zcx_posting_failed.        " declared: caller MUST handle
ENDCLASS.
CLASS zcl_poster IMPLEMENTATION.
  METHOD post.
    IF iv_doc IS INITIAL.
      RAISE EXCEPTION NEW zcx_posting_failed( iv_doc = iv_doc ).   " 7.5+ inline form
    ENDIF.
    " ... real posting ...
  ENDMETHOD.
ENDCLASS.

" caller is FORCED to deal with it:
TRY.
    NEW zcl_poster( )->post( '' ).
  CATCH zcx_posting_failed INTO DATA(lx).
    cl_demo_output=>write( lx->get_text( ) ).         " message, not a swallowed code
ENDTRY.
```
**Why this is better:**
- `RAISING zcx_posting_failed` makes the caller handle or declare it — no silent ignore.
- The exception carries data (`mv_doc`) and a human-readable text.

**Step-by-Step Explanation:**
- `INHERITING FROM cx_static_check` — a *checked* exception (must be handled/declared).
- `RAISE EXCEPTION NEW zcx_posting_failed( … )` — create and throw in one statement.
- `CATCH zcx_posting_failed INTO DATA(lx)` — handle it; `lx->get_text( )` returns the message.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** Wrap a BAPI: convert its `bapiret2` failure messages into a chained, texted exception, and release a lock in `CLEANUP` whether or not it fails.

```abap
CLASS zcl_so_creator DEFINITION PUBLIC CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS create IMPORTING is_header TYPE bapisdhd1
                   RETURNING VALUE(rv_so) TYPE vbeln
                   RAISING   zcx_posting_failed.
ENDCLASS.

CLASS zcl_so_creator IMPLEMENTATION.
  METHOD create.
    DATA lt_return TYPE STANDARD TABLE OF bapiret2.

    TRY.
        " ... ENQUEUE lock, then call the BAPI ...
        CALL FUNCTION 'BAPI_SALESORDER_CREATEFROMDAT2'
          EXPORTING order_header_in = is_header
          IMPORTING salesdocument   = rv_so
          TABLES    return          = lt_return.

        " Translate BAPI error messages into a proper exception:
        LOOP AT lt_return INTO DATA(ls_ret) WHERE type CA 'EA'.   " error/abort
          ROLLBACK WORK.
          RAISE EXCEPTION NEW zcx_posting_failed(
            iv_doc = |{ ls_ret-message }| ).
        ENDLOOP.

        COMMIT WORK AND WAIT.

      CLEANUP.
        " runs if an exception leaves the TRY block — release the lock:
        " CALL FUNCTION 'DEQUEUE_...'.
    ENDTRY.
  ENDMETHOD.
ENDCLASS.

" caller:
TRY.
    DATA(lv_so) = NEW zcl_so_creator( )->create( ls_header ).
    cl_demo_output=>write( |Created { lv_so }| ).
  CATCH zcx_posting_failed INTO DATA(lx).
    cl_demo_output=>write( |Failed: { lx->get_text( ) }| ).
ENDTRY.
```

**Detailed Walkthrough:**
- **BAPI `return` table → exception** — the classic message structure becomes a single, catchable error carrying the BAPI text.
- **`CLEANUP`** — guarantees the lock is released on the exception path, the OO replacement for scattered cleanup code.
- **`RAISING` on `create`** — the failure mode is part of the method's contract.

**How This Works in Practice.** Every consumer of `create` either handles `zcx_posting_failed` or propagates it upward; the original BAPI message is preserved in the exception text, so the *why* survives.

**Why This Implementation.** Converting procedural `bapiret2` handling into class-based exceptions is the standard way to give legacy SAP APIs a clean, hard-to-ignore error surface.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Swallowing exceptions silently.**
```abap
TRY.
    risky( ).
  CATCH zcx_posting_failed.
    "  ← empty catch: error vanishes, worse than a return code
ENDTRY.
```
**Why this is wrong:** an empty catch hides failure entirely; the program proceeds in a broken state.
**Correct approach:** handle (recover), log, or re-raise — never catch-and-ignore.

**Mistake #2: Losing the root cause when re-raising.**
```abap
CATCH cx_sy_open_sql_db INTO DATA(lx_db).
  RAISE EXCEPTION TYPE zcx_posting_failed.    " original DB cause discarded
```
**Why this is wrong:** the underlying cause (the DB error) is lost, making diagnosis hard.
**Correct approach:** chain it: `RAISE EXCEPTION NEW zcx_posting_failed( previous = lx_db ).`

---

## 8. Comparison With Similar Concepts

**`CX_STATIC_CHECK` vs `CX_DYNAMIC_CHECK` vs `CX_NO_CHECK`:** static = compiler forces handling/declaration (recoverable, expected errors); dynamic = possible but declaration would be noise (e.g. arithmetic); no-check = pervasive/fatal, propagates freely (e.g. failed assertions). Pick by how callers should treat it.

**Class-based vs classic exceptions:** classic `RAISE`/`MESSAGE … RAISING` and `EXCEPTIONS` in `CALL FUNCTION` are the legacy mechanism — no objects, no chaining, limited data. New code uses class-based exceptions.

**Exception vs return code:** a return code is ignorable and dataless; an exception (especially checked) is enforced and carries message + cause. Prefer exceptions for real error conditions; reserve return values for expected non-error outcomes.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **BAPIs / function modules:** wrap `bapiret2`/`EXCEPTIONS` into class-based exceptions (Example 2).
- **Messages (T100):** exceptions can carry a T100 message via the `IF_T100_MESSAGE` interface so `get_text( )` returns translatable text.
- **RAP:** uses messages and failed/reported structures; domain logic can still raise exceptions internally.
- **Units of work:** `CLEANUP` + explicit `ROLLBACK`/`COMMIT` keep transactional integrity.

**SAP-Specific Considerations:** make custom exceptions global classes (`ZCX_…`) inheriting from the right base; add T100 texts for translation. `RESUMABLE` exceptions allow `RESUME` to continue after handling — rare, advanced. Don't let exceptions cross RFC boundaries unhandled.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: one catch-all `cx_root` everywhere.**
```abap
TRY.
    big_operation( ).
  CATCH cx_root INTO DATA(lx).    " catches EVERYTHING, including bugs you didn't mean to
ENDTRY.
```
**Why this fails:** over-broad catches hide unrelated defects (e.g. a true programming error) and make handling vague.
**Correct approach:** catch the specific exceptions you can handle; let truly unexpected ones propagate.

**Common Gotcha:** a `CX_STATIC_CHECK` exception declared with `RAISING` propagates the obligation up the call chain — every caller must handle or re-declare it. This is by design (it forces awareness), but it means changing a method's `RAISING` clause can ripple to callers.

---

## 11. Testing & Validation

**How to Verify Understanding:** For each failure path, is the error a *catchable, texted, chained* exception — not a swallowed code? Can a test assert that the right exception is raised?

**Unit Test Example:**
```abap
CLASS ltcl_poster DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS empty_doc_raises FOR TESTING.
ENDCLASS.

CLASS ltcl_poster IMPLEMENTATION.
  METHOD empty_doc_raises.
    TRY.
        NEW zcl_poster( )->post( '' ).
        cl_abap_unit_assert=>fail( 'expected zcx_posting_failed' ).
      CATCH zcx_posting_failed INTO DATA(lx).
        cl_abap_unit_assert=>assert_not_initial( lx->get_text( ) ).   " carries a message
    ENDTRY.
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the test proves the method *raises* (not returns) on bad input and that the exception carries a non-empty text — the contract a return code could not guarantee.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Class-based exceptions make failure explicit, data-carrying, and (for `CX_STATIC_CHECK`) impossible to silently ignore.
- Choose the base by caller obligation: static (must handle), dynamic (lenient), no-check (propagates).
- `TRY/CATCH/CLEANUP`; chain with `previous`; never swallow; catch specifically.

**When to Apply:** any real error condition; wrapping legacy `sy-subrc`/`bapiret2` APIs into clean, catchable failures.

**Red Flags:** empty catch blocks; `CATCH cx_root` as a habit; dropping the `previous` cause; using return codes for genuine errors.

---

## 13. Dependency Map

**Depends On:**
- `02_01_abap_class_syntax.md` — exceptions are classes.
- `02_07_inheritance_in_abap.md` — exceptions form an inheritance hierarchy under `CX_ROOT`.

**Enables:**
- `03_05_dependency_injection.md`, `03_06_abap_unit_and_test_doubles.md` — testing error paths cleanly.
- Robust BAPI/service wrappers across the system.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "RAISE EXCEPTION", "TRY", "CATCH", "CLEANUP", "CX_ROOT", "CX_STATIC_CHECK", "Exception Classes", "IF_T100_MESSAGE".

**Design Patterns & Best Practices:** Clean ABAP → *Error Handling* (use class-based exceptions; don't catch everything; throw a specific exception; document with `RAISING`) (`github.com/SAP/styleguides`).
