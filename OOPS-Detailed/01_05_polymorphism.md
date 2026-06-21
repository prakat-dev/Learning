# Polymorphism

**Learning Objective:** After this topic you can treat different types uniformly through one reference, letting each object respond in its own way — and you can replace branching `IF type = …` logic with polymorphic dispatch.

**Difficulty Level:** Foundational
**Time to Master:** 75 minutes
**Prerequisites:** `01_04_inheritance.md`, `01_06_abstraction_and_interfaces.md`
**Official Sources:**
- ABAP Keyword Documentation → *Polymorphism*, *REDEFINITION*, *Interface Reference Variables* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Inheritance* / *Interfaces* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** A report exports data to PDF, XLSX, and CSV. The export driver is one big routine: `IF format = 'PDF'. … ELSEIF format = 'XLSX'. … ELSEIF format = 'CSV'. …`. Every new format means editing that routine, retesting all branches, and risking the others.

**What Happens WITHOUT This Concept.** The branching ladder lives in many places (`export`, `preview`, `email_as`) and each must be edited for every new format. The cyclomatic complexity climbs; one change can break unrelated branches.

**Why This Matters in SAP.** "Same operation, many variants" is endemic: output formats, document categories, payment methods, tax procedures. Polymorphism removes the `IF`/`CASE` ladder and lets each variant own its behaviour.

---

## 3. Core Concept Explanation

**Definition.** **Polymorphism** ("many forms") means a single reference of a general type (an interface or a superclass) can point to objects of different specific types, and calling a method on it runs *that object's* version. The caller issues one call; the right implementation runs based on the actual object.

**Key Principles:**
- The caller depends on the general type (interface/superclass), not the specific one.
- Each concrete type provides its own method body.
- Dispatch happens at runtime based on the *actual* object, not the reference's declared type — this is **dynamic dispatch**.

**How It Works.** Hold a `REF TO lif_exporter` (or `REF TO` a superclass). Whatever concrete object it points to, `ref->export( )` calls that object's `export`. Two routes provide the variants: **interface implementation** (preferred) or **inheritance + `REDEFINITION`**.

**Why It's Designed This Way.** Polymorphism turns a "decide-then-do" branch into a "do" call. The decision (which type) is made once, at creation; from then on the code just calls the contract. Adding a variant adds a class — existing code is untouched (Open–Closed, Topic 3.1).

---

## 4. Visual Representation

```
   BRANCHING (no polymorphism)            POLYMORPHIC DISPATCH

   export( fmt ):                         lo_exp TYPE REF TO lif_exporter
     IF fmt = 'PDF'  -> pdf code            │
     ELSEIF 'XLSX'   -> xlsx code           ▼
     ELSEIF 'CSV'    -> csv code        lo_exp->export( )
     ...add a branch per format...          │  runtime picks the real object's method
     (edit this routine every time)         ├──► lcl_pdf_exporter ~export
                                            ├──► lcl_xlsx_exporter~export
                                            └──► lcl_csv_exporter ~export
                                          (add a class; caller never changes)
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Export in several formats through one contract.

**The WRONG Way (Anti-Pattern):**
```abap
METHOD export.
  CASE iv_format.
    WHEN 'PDF'.  " ...build PDF...
    WHEN 'XLSX'. " ...build XLSX...
    WHEN 'CSV'.  " ...build CSV...
    WHEN OTHERS. RAISE EXCEPTION TYPE cx_sy_dyn_call_illegal_method.
  ENDCASE.
ENDMETHOD.
```
**Problems with this code:**
- Every new format edits this method (and any other method with the same `CASE`).
- All branches re-tested for any change; high risk of breaking siblings.

**The RIGHT Way (Following Best Practice):**
```abap
INTERFACE lif_exporter.
  METHODS export IMPORTING it_data TYPE string_table
                 RETURNING VALUE(rv_out) TYPE xstring.
ENDINTERFACE.

CLASS lcl_csv_exporter DEFINITION.
  PUBLIC SECTION. INTERFACES lif_exporter.
ENDCLASS.
CLASS lcl_csv_exporter IMPLEMENTATION.
  METHOD lif_exporter~export.
    " ...CSV-specific building, returns bytes...
    rv_out = cl_abap_codepage=>convert_to( concat_lines_of( table = it_data sep = |,| ) ).
  ENDMETHOD.
ENDCLASS.

CLASS lcl_xlsx_exporter DEFINITION.
  PUBLIC SECTION. INTERFACES lif_exporter.
ENDCLASS.
CLASS lcl_xlsx_exporter IMPLEMENTATION.
  METHOD lif_exporter~export.
    " ...XLSX-specific building...
  ENDMETHOD.
ENDCLASS.

" Caller depends only on the contract:
DATA lo_exporter TYPE REF TO lif_exporter.
lo_exporter = NEW lcl_csv_exporter( ).      " or NEW lcl_xlsx_exporter( )
DATA(lv_bytes) = lo_exporter->export( lt_data ).   " one call, right behaviour
```
**Why this is better:**
- Adding `lcl_pdf_exporter` is a new class; the caller is unchanged.
- Each format's logic is isolated and independently testable.

**Step-by-Step Explanation:**
- `lif_exporter` is the general type the caller holds.
- Each class implements `export` differently.
- `lo_exporter->export( )` runs the *actual* object's version — dynamic dispatch.

---

## 6. Code Example 2: Real-World Application

**Business Scenario:** A document service exports a batch in whatever format the user picked, choosing the implementation once and then looping polymorphically.

```abap
CLASS lcl_export_service DEFINITION.
  PUBLIC SECTION.
    METHODS run IMPORTING iv_format TYPE string
                          it_data   TYPE string_table
                RETURNING VALUE(rv_out) TYPE xstring.
  PRIVATE SECTION.
    METHODS exporter_for IMPORTING iv_format TYPE string
                         RETURNING VALUE(ro) TYPE REF TO lif_exporter.
ENDCLASS.

CLASS lcl_export_service IMPLEMENTATION.
  METHOD run.
    DATA(lo_exporter) = exporter_for( iv_format ).   " choose ONCE
    rv_out = lo_exporter->export( it_data ).          " then call polymorphically
  ENDMETHOD.

  METHOD exporter_for.
    " The ONLY place that maps format -> class (a tiny factory, Topic 3.3):
    ro = COND #( WHEN iv_format = 'CSV'  THEN NEW lcl_csv_exporter( )
                 WHEN iv_format = 'XLSX' THEN NEW lcl_xlsx_exporter( ) ).
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **`exporter_for`** confines the "which type" decision to one small method (a factory). Everywhere else is polymorphic.
- **`run`** does not branch on format — it just calls `export`.
- Adding a format edits only `exporter_for`, never `run` or any caller of `run`.

**How This Works in Practice.** The branching that used to be smeared across the codebase now exists exactly once, in `exporter_for`. The rest of the system speaks only `lif_exporter`.

**Why This Implementation.** This is the standard pairing: a **factory** localizes creation; **polymorphism** handles use. Together they implement the Open–Closed Principle.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Re-introducing the `CASE` by checking the concrete type.**
```abap
LOOP AT lt_exporters INTO DATA(lo).
  IF lo IS INSTANCE OF lcl_pdf_exporter.   " type-checking defeats polymorphism
    " special handling...
  ENDIF.
ENDLOOP.
```
**Why this is wrong:** inspecting concrete types re-creates the branching you eliminated and couples the caller to specifics.
**Correct approach:** put the differing behaviour *in the method* each class overrides, then just call it.

**Mistake #2: Typing the variable to the concrete class, losing polymorphism.**
```abap
DATA lo_exporter TYPE REF TO lcl_csv_exporter.   " too specific
lo_exporter = NEW lcl_xlsx_exporter( ).           " syntax error / no substitution
```
**Why this is wrong:** a concrete reference can't hold a sibling type; you lose the whole point.
**Correct approach:** type the reference to the interface: `DATA lo_exporter TYPE REF TO lif_exporter.`

---

## 8. Comparison With Similar Concepts

**Polymorphism via interface vs via inheritance:** interfaces give polymorphism across *unrelated* classes that share only a contract (preferred, more flexible); inheritance + `REDEFINITION` gives polymorphism within an "is-a" family that also shares code. ABAP supports both; choose interface unless you genuinely need shared implementation.

**Polymorphism vs `CASE`/`IF`:** branching decides behaviour at the call site every time; polymorphism decides it once (at creation) and then dispatches automatically. The branch count grows with variants; polymorphic code does not.

**Static vs dynamic dispatch:** ABAP method calls on instance/interface references are dynamically dispatched — the runtime object decides. (Static-typed *helpers* are resolved at compile time and are not polymorphic.)

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **BAdIs:** the framework calls your implementation polymorphically through the BAdI interface.
- **Strategy pattern (Topic 3.4):** strategy *is* polymorphism applied to "pluggable algorithms."
- **Output/format frameworks, payment methods, tax procedures:** all natural polymorphism candidates.

**SAP-Specific Considerations:** dynamic dispatch has negligible overhead for typical business logic; do not avoid polymorphism for "performance" in normal code. RTTI/casting (Topic 2.8) exists for the rare cases where you must inspect the actual type — use it sparingly, as frequent casting signals a missing polymorphic method.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the "switch on type" that should have been a method.**
```abap
METHOD area.
  CASE mo_shape->kind( ).
    WHEN 'CIRCLE'. rv = ...
    WHEN 'SQUARE'. rv = ...
  ENDCASE.
ENDMETHOD.
```
**Why this fails:** the shape should compute its own area; the `CASE` belongs inside each shape as an overridden/implemented `area( )`.
**Correct approach:** give each shape an `area( )` and call `mo_shape->area( )`.

**Common Gotcha:** an unbound interface reference still type-checks at compile time but raises `CX_SY_REF_IS_INITIAL` at runtime if you call a method before assigning a real object. Always assign before calling.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can you add a new variant without editing any caller (only adding a class and one factory line)? If callers must change, you don't yet have polymorphism.

**Unit Test Example:**
```abap
CLASS ltcl_export DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION.
    METHODS dispatch_runs_concrete_impl FOR TESTING.
ENDCLASS.

CLASS ltcl_export IMPLEMENTATION.
  METHOD dispatch_runs_concrete_impl.
    DATA lt_data TYPE string_table.
    APPEND 'a' TO lt_data. APPEND 'b' TO lt_data.

    " One reference type, two concrete objects, two behaviours:
    DATA lo TYPE REF TO lif_exporter.

    lo = NEW lcl_csv_exporter( ).
    DATA(lv_csv) = cl_abap_codepage=>convert_from( lo->export( lt_data ) ).
    cl_abap_unit_assert=>assert_equals( act = lv_csv exp = 'a,b' ).
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** the test holds the general `lif_exporter` type but exercises the concrete CSV behaviour — confirming the right implementation is dispatched through the contract.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- Polymorphism = one general reference, many concrete behaviours, chosen at runtime.
- It replaces growing `IF type = …`/`CASE` ladders with a single call to a contract method.
- Reach it via interfaces (preferred) or inheritance + `REDEFINITION`; type the reference to the *general* type.

**When to Apply:** "same operation, many variants," especially when variants will be added over time.

**Red Flags:** `IS INSTANCE OF`/type-checking to decide behaviour; references typed to concrete classes for varying behaviour; `CASE` ladders duplicated across methods.

---

## 13. Dependency Map

**Depends On:**
- `01_04_inheritance.md` — inheritance + `REDEFINITION` is one route to polymorphism.
- `01_06_abstraction_and_interfaces.md` — interfaces are the preferred route.

**Enables:**
- `03_04_strategy_pattern.md` — strategy is polymorphism for pluggable algorithms.
- `03_01_solid_principles.md` — the Open–Closed and Liskov principles rest on polymorphism.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "Polymorphism", "REDEFINITION", "Interface Reference Variables", and "IS INSTANCE OF" (for the rare, deliberate type test).

**Design Patterns & Best Practices:** Clean ABAP → prefer polymorphism to branching over types (`github.com/SAP/styleguides`). Conceptual basis: the Open–Closed and Liskov Substitution principles (Topic 3.1).
