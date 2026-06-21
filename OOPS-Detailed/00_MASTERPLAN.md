# 00 — ABAP Object-Oriented Programming: Curriculum Masterplan

> **Curriculum:** ABAP OOP — From Foundations to Enterprise Mastery
> **Version:** 1.0
> **Format:** Self-contained Markdown topic files, code in ABAP 7.4+ syntax
> **Primary authoritative sources:**
> - SAP **ABAP Keyword Documentation** — `https://help.sap.com/doc/abapdocu_latest_index_htm/latest/en-US/index.htm`
> - SAP **Clean ABAP** style guide — `https://github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md`
> - SAP **Code Pal for ABAP** — `https://github.com/SAP/code-pal-for-abap`
> - SAP **ABAP test isolation examples** — `https://github.com/SAP-samples/abap-test-isolation-examples`

---

## A. Curriculum Overview

### Problem Statement — What's wrong with how ABAP OOP is usually taught?

Most ABAP OOP material suffers from three recurring failures:

1. **Syntax-first, reasoning-never.** Learners are shown `CLASS … DEFINITION` before they understand *what problem a class solves*. They can write a class but cannot decide *whether* to write one.
2. **Procedural habits in OO clothing.** ABAP grew up procedural (reports, function modules, `FORM` routines). A lot of "OO ABAP" is procedural code wrapped in a class — one giant method, global-ish public attributes, no encapsulation. The syntax is OO; the design is not.
3. **Toy examples with no SAP context.** Generic `lcl_dog` / `lcl_cat` animals teach the keyword but not how OO interacts with the things ABAP developers actually touch: BAPIs, function modules, database access, exceptions, and ABAP Unit.

### Vision & Success Criteria

A learner who completes this curriculum can:

- Explain *why* a given OO concept exists in terms of a concrete maintenance or change-management problem.
- Read an existing ABAP class and identify whether it actually uses OO design or just OO syntax.
- Write classes, interfaces, and exception classes that follow Clean ABAP and pass ABAP Unit tests.
- Apply SOLID principles and a small set of design patterns to real SAP scenarios.
- Make defensible design decisions and justify them.

**Success is measured by transfer, not recall:** the learner should be able to design a new class for a problem they have never seen, not merely recite definitions.

### Target Audience & Prerequisites

- **Primary audience:** ABAP developers comfortable with procedural ABAP (data declarations, internal tables, `SELECT`, `LOOP`, function modules) who want to think and build object-orientedly.
- **Assumed knowledge:** basic ABAP types and statements, internal tables, open SQL basics, the ABAP Development Tools (ADT / "Eclipse") or SE80/SE24.
- **Not assumed:** any prior OO experience in any language. Every OO concept is taught from zero.

### Learning Philosophy & Approach

Each topic follows the same loop: **Problem → Concept → Wrong way → Right way → SAP reality → Test it.** Concepts are introduced only when the problem that motivates them has already been felt. Code is shown broken before it is shown correct, because the contrast is where the learning lives.

---

## B. Complete Learning Path

```
                        ABAP OOP LEARNING PATH

  TIER 1: OOP FUNDAMENTALS            (language-independent thinking)
  ┌───────────────────────────────────────────────────────────────┐
  │  1.1 Why OOP                                                    │
  │       │                                                         │
  │       ▼                                                         │
  │  1.2 Classes & Objects                                         │
  │       │                                                         │
  │       ├──────────────┬───────────────┐                         │
  │       ▼              ▼               ▼                          │
  │  1.3 Encapsulation  1.6 Abstraction & Interfaces               │
  │       │              │                                          │
  │       ▼              │                                          │
  │  1.4 Inheritance ◄───┘                                         │
  │       │                                                         │
  │       ▼                                                         │
  │  1.5 Polymorphism                                              │
  └───────────────────────────────────────────────────────────────┘
                        │
                        ▼
  TIER 2: ABAP-SPECIFIC IMPLEMENTATION   (how ABAP expresses the above)
  ┌───────────────────────────────────────────────────────────────┐
  │  2.1 ABAP Class Syntax (local vs global)                       │
  │  2.2 Attributes (instance vs static)                           │
  │  2.3 Methods (instance / static / functional)                  │
  │  2.4 Constructors (instance & class constructor)               │
  │  2.5 Visibility Sections                                       │
  │  2.6 Interfaces in ABAP                                        │
  │  2.7 Inheritance in ABAP (REDEFINITION, SUPER->, FINAL)        │
  │  2.8 Casting & RTTI (narrowing / widening)                     │
  │  2.9 Events & Event Handlers                                   │
  │  2.10 Class-Based Exceptions                                   │
  │  2.11 FRIENDS                                                  │
  └───────────────────────────────────────────────────────────────┘
                        │
                        ▼
  TIER 3: ENTERPRISE PATTERNS & MASTERY
  ┌───────────────────────────────────────────────────────────────┐
  │  3.1 SOLID Principles in ABAP                                  │
  │  3.2 Singleton Pattern                                         │
  │  3.3 Factory Pattern                                           │
  │  3.4 Strategy Pattern                                          │
  │  3.5 Dependency Injection                                      │
  │  3.6 ABAP Unit & Test Doubles                                  │
  │  3.7 Clean ABAP in Practice                                    │
  └───────────────────────────────────────────────────────────────┘
```

**Hard dependency rule:** never start a Tier *n+1* topic before its Tier *n* prerequisites are understood. Tier 2 explains *how ABAP does* what Tier 1 explains *why*. Tier 3 combines Tier 2 mechanisms into designs.

---

## C. Module Breakdown

### Module 01 — OOP Fundamentals
- **Learning objectives:** Understand the maintenance problems OO solves; define class/object/encapsulation/inheritance/polymorphism/abstraction; reason about *when* to apply each.
- **Topics:** 1.1–1.6 (see inventory).
- **Prerequisites:** Procedural ABAP.
- **Estimated time to master:** 8–12 hours.
- **Real-world scenarios:** Replacing a 2,000-line report `FORM` jungle; isolating change in a pricing engine.
- **Official sources:** ABAP Keyword Documentation → "ABAP Objects"; Clean ABAP → "Classes", "Methods".

### Module 02 — ABAP-Specific Implementation
- **Learning objectives:** Translate every Tier-1 concept into correct, idiomatic ABAP syntax; handle exceptions and casting safely.
- **Topics:** 2.1–2.11.
- **Prerequisites:** Module 01.
- **Estimated time to master:** 15–20 hours.
- **Real-world scenarios:** Building a reusable document-processing class with interfaces; wrapping a BAPI in a class-based exception API.
- **Official sources:** ABAP Keyword Documentation → "Classes", "Interfaces", "Exceptions"; Clean ABAP.

### Module 03 — Enterprise Patterns & Mastery
- **Learning objectives:** Apply SOLID and core design patterns; write isolated, testable units; meet Clean ABAP standards.
- **Topics:** 3.1–3.7.
- **Prerequisites:** Modules 01–02.
- **Estimated time to master:** 15–25 hours.
- **Real-world scenarios:** Pluggable tax-calculation strategies; injecting a test double for a database gateway; making a 30% test-coverage legacy class testable.
- **Official sources:** Clean ABAP; *Clean ABAP* (SAP PRESS book); ABAP test isolation examples; Gang of Four (*Design Patterns*, 1994); R. C. Martin (SOLID).

---

## D. Topic Inventory

| ID | Filename | Level | Learning Objective (short) | Code? | Depends on |
|----|----------|-------|----------------------------|:----:|-----------|
| 1.1 | `01_01_why_oop_principles.md` | Foundational | Articulate the maintenance problems OO solves | Yes | — |
| 1.2 | `01_02_classes_and_objects.md` | Foundational | Distinguish a class (blueprint) from an object (instance) | Yes | 1.1 |
| 1.3 | `01_03_encapsulation.md` | Foundational | Hide internal state behind a stable interface | Yes | 1.2 |
| 1.4 | `01_04_inheritance.md` | Foundational | Reuse and specialize behaviour via an "is-a" hierarchy | Yes | 1.2, 1.3 |
| 1.5 | `01_05_polymorphism.md` | Foundational | Treat different types uniformly through one interface | Yes | 1.4, 1.6 |
| 1.6 | `01_06_abstraction_and_interfaces.md` | Foundational | Separate *what* from *how* via abstraction & interfaces | Yes | 1.2 |
| 2.1 | `02_01_abap_class_syntax.md` | Intermediate | Write local & global classes correctly | Yes | 1.2 |
| 2.2 | `02_02_attributes_instance_static.md` | Intermediate | Choose instance vs static attributes | Yes | 2.1 |
| 2.3 | `02_03_methods.md` | Intermediate | Write instance/static/functional methods | Yes | 2.1 |
| 2.4 | `02_04_constructors.md` | Intermediate | Use instance & class constructors correctly | Yes | 2.1, 2.2 |
| 2.5 | `02_05_visibility_sections.md` | Intermediate | Apply public/protected/private deliberately | Yes | 1.3, 2.1 |
| 2.6 | `02_06_interfaces_in_abap.md` | Intermediate | Define & implement ABAP interfaces | Yes | 1.6, 2.1 |
| 2.7 | `02_07_inheritance_in_abap.md` | Intermediate | Use REDEFINITION, SUPER->, FINAL, ABSTRACT | Yes | 1.4, 2.1 |
| 2.8 | `02_08_casting_and_rtti.md` | Advanced | Narrow/widen casts safely; use RTTI | Yes | 2.6, 2.7 |
| 2.9 | `02_09_events_and_handlers.md` | Advanced | Decouple via RAISE EVENT / SET HANDLER | Yes | 2.3, 2.6 |
| 2.10 | `02_10_class_based_exceptions.md` | Intermediate | Design class-based exception hierarchies | Yes | 2.1, 2.7 |
| 2.11 | `02_11_friends.md` | Advanced | Use FRIENDS narrowly and justifiably | Yes | 2.5 |
| 3.1 | `03_01_solid_principles.md` | Advanced | Apply the five SOLID principles in ABAP | Yes | Module 02 |
| 3.2 | `03_02_singleton_pattern.md` | Advanced | Implement & critique the Singleton | Yes | 2.4, 3.1 |
| 3.3 | `03_03_factory_pattern.md` | Advanced | Centralize object creation behind a factory | Yes | 2.6, 3.1 |
| 3.4 | `03_04_strategy_pattern.md` | Advanced | Make behaviour pluggable at runtime | Yes | 1.5, 3.1 |
| 3.5 | `03_05_dependency_injection.md` | Advanced | Invert dependencies for testability | Yes | 3.1, 3.3 |
| 3.6 | `03_06_abap_unit_and_test_doubles.md` | Advanced | Write isolated unit tests with test doubles | Yes | 3.5 |
| 3.7 | `03_07_clean_abap_in_practice.md` | Advanced | Apply the Clean ABAP rule set holistically | Yes | All |

**Total: 24 topic files + this masterplan.**

---

## E. Validation Framework

### Official sources by area
- **Language/syntax claims** → ABAP Keyword Documentation (search the keyword, e.g. `CLASS`, `INTERFACE`, `RAISE EXCEPTION`).
- **Style/design claims** → Clean ABAP style guide + Code Pal for ABAP checks.
- **Pattern claims** → Gang of Four definitions; SOLID per Robert C. Martin.
- **Testing claims** → ABAP Unit documentation + SAP test-isolation examples repo.

### Code validation strategy
- All code targets ABAP 7.4+ syntax (inline declarations `DATA(x)=`, `NEW`, constructor expressions). Legacy syntax appears *only* when explicitly contrasting old vs new.
- Each non-trivial example is structured so it could be pasted into a local test class / report and activated. SAP-delivered objects referenced (e.g. `BAPI_*`, `CL_*`) are real, commonly available APIs; learners must still confirm availability in their own system/release.

### Fact-checking & hallucination-prevention checklist (run per file)
- [ ] Every syntax statement maps to a real ABAP keyword.
- [ ] No invented SAP table, BAPI, or class names presented as standard without a "verify in your system" caveat.
- [ ] No fabricated deep-link URLs — cite the stable doc roots above + the keyword to search.
- [ ] Design claims trace to Clean ABAP / GoF / SOLID, not to opinion stated as fact.
- [ ] Wrong-way examples are labelled as anti-patterns so they're never copied by mistake.

### Quality gates before "publishing" a topic file
1. All 14 mandatory sections present.
2. At least 3 code examples (anti-pattern, correct, real-world) + 1 unit test where meaningful.
3. Cross-references resolve to filenames in this inventory.
4. Reads correctly standalone *and* in path order.

---

## F. Implementation Roadmap

- **File naming:** `MM_TT_descriptive_name.md` where `MM` = module number, `TT` = topic number. Masterplan is `00_MASTERPLAN.md`.
- **File format:** GitHub-flavored Markdown; code fenced as ```abap; ASCII diagrams for visuals; bold on first mention of a key term; relative links for cross-references.
- **Reading order:** masterplan → follow the Tier 1 → 2 → 3 path; within a module, ascending topic number, respecting the dependency column.
- **Code standard:** syntactically correct, commented to explain *why*, error-handled, single-concept-focused.
- **Per-file proofread checklist:** structure complete → code valid → sources cited → cross-refs valid → standalone-readable.

### Mandatory section template (every topic file)
1. Header & Metadata · 2. The Problem · 3. Core Concept · 4. Visual · 5. Code Ex.1 (wrong vs right) · 6. Code Ex.2 (real-world) · 7. Code Ex.3 (common mistakes) · 8. Comparison with similar concepts · 9. SAP ecosystem integration · 10. Anti-patterns & gotchas · 11. Testing & validation · 12. Quick reference · 13. Dependency map · 14. Official sources.

---

## Build Status

| File | Status |
|------|:------:|
| `00_MASTERPLAN.md` | ✅ Complete |
| `01_01_why_oop_principles.md` | ✅ Complete |
| `01_02_classes_and_objects.md` | ✅ Complete |
| `01_03_encapsulation.md` | ✅ Complete |
| `01_04_inheritance.md` | ✅ Complete |
| `01_05_polymorphism.md` | ✅ Complete |
| `01_06_abstraction_and_interfaces.md` | ✅ Complete |
| `02_01_abap_class_syntax.md` | ✅ Complete |
| `02_02_attributes_instance_static.md` | ✅ Complete |
| `02_03_methods.md` | ✅ Complete |
| `02_04_constructors.md` | ✅ Complete |
| `02_05_visibility_sections.md` | ✅ Complete |
| `02_06_interfaces_in_abap.md` | ✅ Complete |
| `02_07_inheritance_in_abap.md` | ✅ Complete |
| `02_08_casting_and_rtti.md` | ✅ Complete |
| `02_09_events_and_handlers.md` | ✅ Complete |
| `02_10_class_based_exceptions.md` | ✅ Complete |
| `02_11_friends.md` | ✅ Complete |
| `03_01_solid_principles.md` | ✅ Complete |
| `03_02_singleton_pattern.md` | ✅ Complete |
| `03_03_factory_pattern.md` | ✅ Complete |
| `03_04_strategy_pattern.md` | ✅ Complete |
| `03_05_dependency_injection.md` | ✅ Complete |
| `03_06_abap_unit_and_test_doubles.md` | ✅ Complete |
| `03_07_clean_abap_in_practice.md` | ✅ Complete |
| `02_12_shared_memory_objects.md` (proposed gap-fill) | ⏳ Proposed — not yet generated |
| `02_13_persistent_objects.md` (optional gap-fill) | ⏳ Optional — not yet generated |

> **Status:** All three tiers are delivered in full — **24 topic files plus this masterplan (25 files total)**. The core curriculum (Modules 01–03) is complete: OOP fundamentals, ABAP-specific implementation, and enterprise patterns. Two optional gap-fill topics remain available on request: a **Shared Memory Objects** topic (proposed `02_12`, covering shared-memory-enabled classes, the `SHMA` area, the root class, and the attach/detach lifecycle) and an optional **Persistent Objects** topic (`02_13`).
