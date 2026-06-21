# 11 — Clean ABAP & OOP Best Practices

> How to write OO code that is maintainable, testable, and ready for S/4HANA, RAP, and ABAP Cloud.

---

## When to use OOP vs procedural

Use OOP for new development: business logic, reusable services, anything you intend to unit test, and anything feeding RAP or Fiori. Procedural code still appears in reports, older function groups, and simple scripts, but wrap meaningful logic in classes where it pays off. Modern SAP (RAP, BOPF, Web Dynpro, Fiori backends) is object-oriented end to end, so procedural-only skills are no longer sufficient.

---

## SOLID applied to ABAP

| Principle | Meaning | ABAP application |
|-----------|---------|------------------|
| **S**ingle Responsibility | One class, one reason to change | Don't mix DB access, business logic, and UI in one class |
| **O**pen/Closed | Open to extension, closed to modification | Add subclasses or BADI implementations instead of editing working code |
| **L**iskov Substitution | A subclass must work wherever its parent does | A `REDEFINITION` must honor the parent's contract |
| **I**nterface Segregation | Prefer small, focused interfaces | Several narrow interfaces over one fat one |
| **D**ependency Inversion | Depend on abstractions | Inject interfaces, not concrete classes |

---

## Naming conventions

Classes follow `ZCL_<area>_<thing>` (e.g. `ZCL_SD_ORDER_READER`); interfaces follow `ZIF_<area>_<thing>` (e.g. `ZIF_SD_ORDER_REPOSITORY`). Methods read as verb phrases — `get_total`, `calculate_tax`, `is_valid`. Boolean-returning methods use `is_`, `has_`, or `can_` prefixes. Be consistent with your project's standards above all.

---

## Method design

Keep methods short and single-purpose, with few parameters — prefer passing a structure over many scalars. Favor functional methods (a single `RETURNING` parameter) so they can be chained and used inside expressions. Keep one level of abstraction per method, so a method either orchestrates high-level steps or does low-level detail, not both.

```abap
" functional, chainable, expression-friendly
DATA(lv_total) = lo_order->get_items( )->sum_net_value( ).
```

---

## Class design

Make classes `FINAL` by default and open them for inheritance only intentionally. Keep attributes private and expose them through methods. Favor composition over inheritance, and put interfaces at boundaries (database, RFC, external services) so those boundaries can be mocked in tests. One class should represent one concept.

---

## Error handling

Use class-based exceptions and throw specific types. Never swallow an exception in an empty `CATCH` block. Chain with `previous` when re-raising a lower-level exception, and validate inputs early so the code fails fast and close to the cause.

---

## Common pitfalls

A "god class" that does everything should be split by responsibility. Making everything static produces hidden state and hard-to-test code — prefer instances with dependency injection. Deep inheritance trees become fragile; prefer composition and interfaces. Business logic embedded in ALV or UI event handlers belongs in a service class instead. Hard-coded `NEW` of dependencies should become injection. And overly clever one-liners (deeply nested `REDUCE`/`FOR`) that hurt readability should be unpacked.

---

## Clean Core / ABAP Cloud readiness

Clean Core means your custom code stays decoupled from SAP internals so upgrades stay clean. In practice: use only **released** APIs and objects; never access non-released SAP tables directly — go through released CDS views or APIs; extend via released BADIs rather than modifications; keep custom code in the customer namespace; and make sure it passes ATC (ABAP Test Cockpit) cloud-readiness checks. The OO foundation in these notes — interfaces, dependency injection, clean exceptions, and unit tests — is exactly what RAP and ABAP Cloud assume.

---

## The path forward

This OOP foundation feeds directly into the rest of the learning plan: RAP behavior classes and EML, CDS consumption through OO services, Fiori/OData backends built on clean OO logic, and unit-tested custom code that survives upgrades.

---

## Quick self-test

1. State each SOLID letter and one ABAP application.
2. Why make classes `FINAL` by default?
3. Why prefer composition over inheritance?
4. Name three signs of a "god class" and the fix for each.
5. What does Clean Core require for table and API access?
6. How does clean OO design connect to RAP and ABAP Cloud?
