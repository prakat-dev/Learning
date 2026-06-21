# Factory Pattern

**Learning Objective:** After this topic you can implement factory methods and abstract factories in ABAP, returning interface references so callers are decoupled from concrete classes, and combine `CREATE PRIVATE` + `FRIENDS` to centralize and control object creation.

**Difficulty Level:** Advanced
**Time to Master:** 75 minutes
**Prerequisites:** `02_06_interfaces_in_abap.md`, `02_11_friends.md`, `03_01_solid_principles.md`
**Official Sources:**
- ABAP Keyword Documentation → *CLASS-METHODS*, *CREATE PRIVATE*, *FRIENDS* (`help.sap.com/doc/abapdocu_latest_index_htm`)
- Clean ABAP → *Constructors* / *Object Orientation* (`github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`)

---

## 2. The Problem (Why This Concept Matters)

**Real-World Scenario.** Depending on the country, an order needs a different tax calculator. If every caller writes `CASE country … NEW zcl_tax_de … NEW zcl_tax_us …`, that decision is duplicated everywhere. Add a country, and you edit dozens of sites.

**What Happens WITHOUT This Concept.** Object-creation logic (which concrete class, with which setup) is scattered across callers. Callers couple to concrete classes; changing the selection rule means hunting down every `NEW`.

**Why This Matters in SAP.** Factories centralize "which implementation and how to build it" in one place, hand back an interface, and let the rest of the system stay ignorant of concrete classes — essential for pluggable, extensible SAP designs (and for swapping implementations in tests).

---

## 3. Core Concept Explanation

**Definition.** A **factory** encapsulates object creation. Two common forms:
- **Factory method:** a (often static) method that decides which concrete class to instantiate and returns it as an interface reference.
- **Abstract factory:** an object whose job is to create families of related objects, itself injectable and replaceable.

**Key Principles:**
- Return an **interface** (or abstract type), never a concrete class — callers depend on the abstraction (DIP, Topic 3.1).
- Put the *creation decision* in one place.
- Combine with `CREATE PRIVATE` + `FRIENDS` so products can only be built by their factory (Topic 2.11).

**How It Works.** The caller asks the factory for "a tax calculator for country X"; the factory contains the selection logic and returns `lif_tax`. The caller uses the interface and never sees the concrete type.

**Why It's Designed This Way.** Separating *what to use* (the caller's concern) from *how to build it* (the factory's concern) localizes change and removes concrete dependencies from business code.

---

## 4. Visual Representation

```
   caller                         FACTORY (one place to decide)        products
   ─────                          ───────────────────────────         ────────
   factory->create( 'DE' ) ─────► CASE country                        zcl_tax_de  ┐
                                    'DE' → NEW zcl_tax_de              zcl_tax_us  ├─ all implement
                                    'US' → NEW zcl_tax_us              zcl_tax_xx  ┘   lif_tax
                                  ENDCASE
        ◄──── REF TO lif_tax ─────  (returns the ABSTRACTION)
   caller uses lif_tax only; never references a concrete class
```

---

## 5. Code Example 1: Basic Concept

**Scenario:** Choose a tax calculator by country.

**The WRONG Way (Anti-Pattern):**
```abap
" scattered in every caller:
DATA lo_tax TYPE REF TO lif_tax.
CASE iv_country.
  WHEN 'DE'. lo_tax = NEW zcl_tax_de( ).
  WHEN 'US'. lo_tax = NEW zcl_tax_us( ).
ENDCASE.
" ...this same CASE appears in 12 places...
```
**Problems with this code:**
- The creation decision is duplicated; callers couple to every concrete class.
- Adding a country means editing all 12 sites.

**The RIGHT Way (Following Best Practice):**
```abap
INTERFACE lif_tax.
  METHODS calc IMPORTING iv_base TYPE p LENGTH 13 DECIMALS 2
               RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2.
ENDINTERFACE.

CLASS zcl_tax_factory DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    CLASS-METHODS create IMPORTING iv_country TYPE land1
                         RETURNING VALUE(ro)  TYPE REF TO lif_tax
                         RAISING   cx_sy_create_object_error.
ENDCLASS.
CLASS zcl_tax_factory IMPLEMENTATION.
  METHOD create.
    ro = SWITCH #( iv_country
      WHEN 'DE' THEN NEW zcl_tax_de( )
      WHEN 'US' THEN NEW zcl_tax_us( )
      ELSE THROW cx_sy_create_object_error( ) ).   " decision lives HERE, once
  ENDMETHOD.
ENDCLASS.

" every caller now:
DATA(lo_tax) = zcl_tax_factory=>create( iv_country = 'DE' ).
DATA(lv)     = lo_tax->calc( '100.00' ).      " uses lif_tax only
```
**Why this is better:**
- The selection logic exists once; callers depend only on `lif_tax`.
- Adding a country edits one method, not every caller.

**Step-by-Step Explanation:**
- `CLASS-METHODS create … RETURNING REF TO lif_tax` — the factory method returns the abstraction.
- `SWITCH #( … THROW … )` — the single creation decision, with a clean failure for unknown input.
- Callers receive `lif_tax` and never name a concrete class.

---

## 6. Code Example 2: Real-World Application (controlled creation + injectable abstract factory)

**Business Scenario:** Products may only be created by the factory (`CREATE PRIVATE` + `FRIENDS`), and the factory itself is hidden behind an interface so it can be injected and faked.

```abap
" The factory is an injectable abstraction, not just a static method:
INTERFACE lif_tax_factory.
  METHODS create IMPORTING iv_country TYPE land1
                 RETURNING VALUE(ro)  TYPE REF TO lif_tax
                 RAISING   cx_sy_create_object_error.
ENDINTERFACE.

" A product that ONLY its factory may instantiate:
CLASS zcl_tax_de DEFINITION PUBLIC CREATE PRIVATE FRIENDS zcl_tax_factory.
  PUBLIC SECTION.
    INTERFACES lif_tax.
  PRIVATE SECTION.
    METHODS constructor.
ENDCLASS.
CLASS zcl_tax_de IMPLEMENTATION.
  METHOD constructor. ENDMETHOD.
  METHOD lif_tax~calc.
    rv = iv_base * '0.19'.
  ENDMETHOD.
ENDCLASS.

CLASS zcl_tax_factory DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES lif_tax_factory.
ENDCLASS.
CLASS zcl_tax_factory IMPLEMENTATION.
  METHOD lif_tax_factory~create.
    ro = SWITCH #( iv_country
      WHEN 'DE' THEN NEW zcl_tax_de( )      " allowed only because factory is a FRIEND
      ELSE THROW cx_sy_create_object_error( ) ).
  ENDMETHOD.
ENDCLASS.

" A service depends on the factory ABSTRACTION (injected), so tests can fake it:
CLASS zcl_invoice_service DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    METHODS constructor IMPORTING io_factory TYPE REF TO lif_tax_factory.
    METHODS gross IMPORTING iv_country TYPE land1 iv_base TYPE p LENGTH 13 DECIMALS 2
                  RETURNING VALUE(rv) TYPE p LENGTH 13 DECIMALS 2
                  RAISING   cx_sy_create_object_error.
  PRIVATE SECTION.
    DATA mo_factory TYPE REF TO lif_tax_factory.
ENDCLASS.
CLASS zcl_invoice_service IMPLEMENTATION.
  METHOD constructor.
    mo_factory = io_factory.
  ENDMETHOD.
  METHOD gross.
    DATA(lo_tax) = mo_factory->create( iv_country ).
    rv = iv_base + lo_tax->calc( iv_base ).
  ENDMETHOD.
ENDCLASS.
```

**Detailed Walkthrough:**
- **`CREATE PRIVATE FRIENDS zcl_tax_factory`** — products are uninstantiable except through their factory (controlled creation, Topic 2.11).
- **`lif_tax_factory`** — the factory is itself an abstraction, so a service can inject a real or fake factory (DIP).
- **`zcl_invoice_service`** — never names a concrete tax class *or* the concrete factory; fully decoupled.

**How This Works in Practice.** This is the full enterprise shape: a factory centralizes creation, products are locked to their factory, and the factory is injectable — so production wires the real one and tests wire a fake.

**Why This Implementation.** It satisfies DIP and OCP (Topic 3.1): adding a country touches only the factory; nothing downstream changes, and everything stays testable.

---

## 7. Code Example 3: Common Mistakes

**Mistake #1: Factory returning a concrete class.**
```abap
CLASS-METHODS create RETURNING VALUE(ro) TYPE REF TO zcl_tax_de.   " concrete!
```
**Why this is wrong:** callers now depend on `zcl_tax_de`; the decoupling benefit is lost and you can't return a different implementation.
**Correct approach:** return the interface `lif_tax`.

**Mistake #2: Business logic leaking into the factory.**
```abap
METHOD create.
  ro = NEW zcl_tax_de( ).
  DATA(lv) = ro->calc( iv_base ).   " factory is now also doing the calculation
  " ...
ENDMETHOD.
```
**Why this is wrong:** a factory's single responsibility is *creation*; mixing in domain logic violates SRP and makes it a god object.
**Correct approach:** the factory only builds and returns; the caller uses the product.

---

## 8. Comparison With Similar Concepts

**Factory method vs constructor (Topic 2.4):** a constructor always builds *this exact class*; a factory can choose among subtypes, return cached/singleton instances, validate, or fail cleanly. Use a factory when creation needs a decision or control.

**Factory vs Singleton (Topic 3.2):** different intents — a factory *creates* (possibly many) objects; a singleton guarantees *one*. They combine: a singleton factory, or a factory that hands back singletons.

**Static factory method vs abstract (injectable) factory:** a static method is simple but not substitutable; an injectable factory object (behind an interface) can be faked in tests and swapped at runtime. Prefer the injectable form when testability/variation matters.

---

## 9. Integration With the SAP Ecosystem

**How this works with:**
- **BAdIs / `GET BADI`:** the enhancement framework is a factory mechanism returning interface implementations.
- **`CL_*=>create( )` / `=>get_instance( )`:** many SAP classes expose factory methods instead of public constructors.
- **DI (Topic 3.5):** factories are commonly injected; the composition root wires the real factory.
- **Strategy (Topic 3.4):** a factory often produces the strategy object a service will use.

**SAP-Specific Considerations:** factory methods that read configuration/customizing to decide the implementation are a clean way to make behaviour table-driven. Combine with `CREATE PRIVATE` + `FRIENDS` to enforce that products are only built correctly.

---

## 10. Anti-Patterns & Gotchas

**Anti-Pattern: the "factory" that's just `NEW` with extra steps.**
```abap
CLASS-METHODS create RETURNING VALUE(ro) TYPE REF TO zcl_thing.
METHOD create. ro = NEW zcl_thing( ). ENDMETHOD.   " no decision, no abstraction
```
**Why this fails:** with no selection logic and a concrete return type, it adds indirection for zero benefit.
**Correct approach:** introduce a factory when there's a real creation decision or a need to return an abstraction/control instantiation; otherwise just `NEW`.

**Common Gotcha:** a factory that grows a giant `CASE`/`SWITCH` can itself become an OCP problem. If creation rules get complex or table-driven, consider a registry the factory consults, so new products register themselves rather than editing the factory.

---

## 11. Testing & Validation

**How to Verify Understanding:** Can a consumer be tested by injecting a fake factory that returns a fake product — with no concrete production class involved? If the consumer names a concrete class, the factory isn't doing its job.

**Unit Test Example:**
```abap
" Fake factory + fake product → test the service in isolation:
CLASS ltd_tax DEFINITION.
  PUBLIC SECTION. INTERFACES lif_tax.
ENDCLASS.
CLASS ltd_tax IMPLEMENTATION.
  METHOD lif_tax~calc. rv = 1. ENDMETHOD.        " predictable stub
ENDCLASS.

CLASS ltd_factory DEFINITION.
  PUBLIC SECTION. INTERFACES lif_tax_factory.
ENDCLASS.
CLASS ltd_factory IMPLEMENTATION.
  METHOD lif_tax_factory~create. ro = NEW ltd_tax( ). ENDMETHOD.
ENDCLASS.

CLASS ltcl_service DEFINITION FOR TESTING DURATION SHORT RISK LEVEL HARMLESS.
  PRIVATE SECTION. METHODS gross_adds_tax FOR TESTING RAISING cx_static_check.
ENDCLASS.
CLASS ltcl_service IMPLEMENTATION.
  METHOD gross_adds_tax.
    DATA(lo) = NEW zcl_invoice_service( io_factory = NEW ltd_factory( ) ).
    cl_abap_unit_assert=>assert_equals( act = lo->gross( iv_country = 'DE' iv_base = '10' )
                                        exp = CONV p( '11' ) ).   " 10 + stub(1)
  ENDMETHOD.
ENDCLASS.
```
**Test Explanation:** injecting a fake factory (returning a fake tax) lets the service be tested with no real tax classes — the payoff of returning and depending on abstractions.

---

## 12. Quick Reference Summary

**Key Takeaways:**
- A factory centralizes the "which class and how to build it" decision and returns an *interface*.
- Combine `CREATE PRIVATE` + `FRIENDS` so products are only built by their factory.
- Prefer an injectable factory (behind an interface) over a static method when you need substitution/testing.

**When to Apply:** creation requires a decision, configuration, control, or an abstraction; or callers must be decoupled from concrete classes.

**Red Flags:** returning concrete types; domain logic inside the factory; a "factory" with no real decision; an ever-growing creation `CASE`.

---

## 13. Dependency Map

**Depends On:**
- `02_06_interfaces_in_abap.md` — factories return interface references.
- `02_11_friends.md` — `CREATE PRIVATE` + friend factory for controlled creation.
- `03_01_solid_principles.md` — DIP/OCP motivate the pattern.

**Enables:**
- `03_04_strategy_pattern.md` — factories produce strategies.
- `03_05_dependency_injection.md` — factories are injected and wired at the composition root.

---

## 14. Official Source References

**SAP Documentation:** ABAP Keyword Documentation — search "CLASS-METHODS", "CREATE PRIVATE", "FRIENDS", "SWITCH".

**Design Patterns & Best Practices:** Pattern origins: Factory Method & Abstract Factory (Gang of Four). Clean ABAP → *Constructors* (prefer multiple creation methods to optional parameters; use factories for complex construction) (`github.com/SAP/styleguides`).
