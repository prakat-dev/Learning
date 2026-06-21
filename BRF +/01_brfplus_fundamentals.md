# 01 — BRF+ Fundamentals

> What BRF+ is, why it exists, and the object hierarchy you build in it. Transaction: `BRF+` or `BRFPLUS`.

---

## What BRF+ is

BRF+ (Business Rule Framework plus) is an **ABAP/NetWeaver framework for defining business rules declaratively** instead of hardcoding them in ABAP. Business logic — decisions, derivations, validations — is maintained as configurable rules that key users can change without a developer touching code.

Key points:
- Part of the **ABAP NetWeaver stack** (not a separate server).
- Aimed at key users / business experts as well as developers; the workbench has a **Simple mode** and an **Expert mode**.
- Rules can call function modules and ABAP OO methods, so the full power of ABAP is still available when needed.
- Supports optional **versioning** of rules (useful for legal traceability and "what would the system have decided on date X").

**Typical use cases:** purchase order release strategy, discount/pricing derivation, data validation on entry, approval routing, and — in S/4HANA — output determination (covered in File 05).

---

## Why use it instead of ABAP IF/CASE

The same rules could be written as `IF`/`CASE` in ABAP, but then every change needs a developer, a transport, and a deployment. With BRF+, the logic lives in maintainable rules: the business changes a decision table entry, activates it, and the new logic is live — no ABAP change. It moves volatile business logic out of code and into configuration.

---

## The object hierarchy

BRF+ objects nest in a fixed structure. Understanding this hierarchy is the foundation of everything else.

```
Application                  (top-level container for all related objects)
  |- Data Objects            (the "variables": elements, structures, tables)
  |- Function                (the entry point called from ABAP; has a signature)
  |     |- Context           (input parameters - the data the rules work on)
  |     |- Result            (the output the function returns)
  |     |- Ruleset(s)        (collection of rules, assigned to the function)
  |           |- Rule(s)     (IF-THEN logic; the actual business decisions)
  |                 |- Expression(s)   (the computational units: decision table, etc.)
```

### Application
The top-level container that holds all related objects. Default settings defined at application level (like the storage type) are inherited by the objects inside it. You create it first, give it a name, a description, and a development package (use `$TMP` / local if you don't want to transport it).

### Data Objects
The "variables" of BRF+ — the data carriers used in signatures, rules, and as building blocks for decision tables. Three kinds:
- **Element** — a single value (like an ABAP elementary type). Can be bound to a DDIC data element.
- **Structure** — a group of elements (like an ABAP structure).
- **Table** — multiple rows of a structure (like an ABAP internal table).

You can define them by hand (type, length, decimals) or bind directly to existing DDIC types.

### Function
The **point of contact** between ABAP and BRF+. ABAP calls the function, passing input; the function processes its rules and returns a result. A function has a signature made of two parts:
- **Context** — the input parameters (the data objects the rules read). Unsupplied context parameters get initial values.
- **Result** — the single output data object the function returns.

### Ruleset
A **collection of rules** assigned to a function. A ruleset is assigned to exactly one function. You can set a **priority** at ruleset level to control execution order when multiple rulesets apply. Rulesets are processed one after another in sequence.

### Rule
The central unit of business logic — essentially **IF-THEN(-ELSE)** logic. A rule evaluates a condition and performs actions (set the result, call an expression, trigger an action like sending an email or starting a workflow).

### Expression
The **computational unit** that produces a value. The most common is the **decision table**, but there are many types (covered in File 02). A rule typically delegates its "how do I decide" to an expression.

---

## The lifecycle: build → activate → call

1. Create the **Application**.
2. Create **Data Objects** for the context and result (or reuse DDIC-bound ones).
3. Create the **Function** and define its **context** and **result** signature.
4. Create a **Ruleset** and assign it to the function.
5. Add **Rules**, backed by **Expressions** (e.g. a decision table).
6. **Activate** every object (objects must be active to be usable — the workbench has an Activate button).
7. **Test** the function in the workbench (simulation), then **call** it from ABAP (File 03).

> Everything must be **activated** before use. A common beginner mistake is building the rules but forgetting to activate one of the objects.

---

## Simple mode vs Expert mode

The workbench can be switched between modes (in your personalization settings):
- **Simple mode** — guided, fewer options; good for key users and straightforward rules.
- **Expert mode** — exposes all object types and settings; needed for advanced work (custom expression types, detailed signatures, technical settings).

---

## Where it lives technically

BRF+ objects are stored in the framework's own tables (the `FDT_*` tables — "Formula and Derivation Tool", the technical name behind BRF+). You rarely touch these directly; you work through the workbench and the API classes (`CL_FDT_*`, `IF_FDT_*`).

---

## Quick self-test

1. What is BRF+ in one sentence, and which stack is it part of?
2. Why use BRF+ instead of hardcoded ABAP `IF`/`CASE`?
3. List the object hierarchy from Application down to Expression.
4. What are the two parts of a function's signature?
5. How many functions can a ruleset be assigned to?
6. What must you do to every object before it can be used?
7. Name the three kinds of data objects.
