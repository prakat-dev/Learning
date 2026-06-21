# 02 — BRF+ Expressions

> Expressions are the computational units that produce results. The decision table is the workhorse; this file covers it in depth plus the other common types.

---

## What an expression is

An expression is a **self-contained unit of logic** that takes input (usually from the function's context) and returns a value. A rule delegates its decision to an expression. BRF+ ships many expression types; you pick the one that fits the shape of your logic.

---

## Expression types (overview)

| Expression type | Use it for |
|-----------------|------------|
| **Decision Table** | Tabular IF-conditions → result; the most common type |
| **Decision Tree** | Nested, hierarchical conditions (branching) |
| **Formula** | Arithmetic / string / date calculations |
| **Case** | Switch on one value, like ABAP `CASE` |
| **Boolean** | Combine conditions with AND/OR/NOT |
| **Database Lookup** | Read values from a DB table |
| **Loop** | Iterate over a table data object |
| **Function Call** | Call another BRF+ function |
| **Procedure Call** | Call an ABAP function module or class method |
| **Value Range** | Map a value to a range band |
| **Step Sequence / Search Tree** | Ordered multi-step evaluation |

A key strength: **Procedure Call** and **Function Call** mean you can drop back into full ABAP whenever the rules engine isn't enough — call an FM or OO method and use its result inside the rule.

---

## Decision Table — the workhorse

A decision table reads like a spreadsheet: **condition columns** on the left, **result columns** on the right. Each row is one rule. The engine evaluates rows top to bottom and returns the result of the matching row(s).

### Worked example: customer discount

**Business requirement:** derive a discount percentage from customer group, product category, and order value.

**Context (input) data objects:**
- `CUSTOMER_GROUP` (element, char 2)
- `PRODUCT_CATEGORY` (element, char 4)
- `ORDER_VALUE` (element, amount)

**Result data object:**
- `DISCOUNT_PERCENT` (element, decimal)

**Decision table:**

| # | CUSTOMER_GROUP | PRODUCT_CATEGORY | ORDER_VALUE | → DISCOUNT_PERCENT |
|---|----------------|------------------|-------------|--------------------|
| 1 | `VIP` | `*` (any) | `>= 10000` | `15` |
| 2 | `VIP` | `*` (any) | `< 10000` | `10` |
| 3 | `STD` | `ELEC` | `>= 5000` | `8` |
| 4 | `STD` | `*` (any) | `*` (any) | `5` |
| 5 | `*` (any) | `*` (any) | `*` (any) | `0` |

How it evaluates: for a `VIP` customer ordering anything worth 12,000, row 1 matches → 15%. For an `STD` customer buying `ELEC` worth 6,000, row 3 matches → 8%. The catch-all row 5 guarantees a result even when nothing else matches.

**Important behaviors to know:**
- Conditions can be single values, ranges (`>=`, `<`, `between`), or wildcards (any).
- Row order matters — by default the table can be set to return the **first match** or to **evaluate all matching rows** (accumulating results). Know which mode your table uses.
- Always include a **catch-all / default row** so the function never returns an undefined result.
- Columns map directly to context (conditions) and result (outcomes) data objects.

---

## Decision Tree

Use when the logic is naturally **hierarchical / nested** rather than a flat table. You branch on one condition, and each branch can branch further.

```
ORDER_VALUE >= 10000 ?
   |- yes -> CUSTOMER_GROUP = VIP ? -> yes: 20%
   |                                 -> no : 12%
   |- no  -> 5%
```

A decision tree can express the same logic as a decision table, but trees are clearer when branches are deep and asymmetric; tables are clearer when conditions are uniform across all rows.

---

## Formula

For calculations. Supports arithmetic, string, and date/time functions.

```
Example formula for a line total:
  QUANTITY * UNIT_PRICE * ( 1 - DISCOUNT_PERCENT / 100 )
```

Formulas are used when the result is computed, not looked up.

---

## Case / Boolean

- **Case** — switch on a single value and return per branch (like ABAP `CASE`).
- **Boolean** — returns true/false by combining conditions with AND/OR/NOT; often used as the condition part of a rule.

---

## Database Lookup

Reads data from a DB table directly inside the rule — for example, look up a customer's risk class from a master table to feed a later decision, without writing ABAP.

---

## Procedure Call / Function Call

- **Procedure Call** — invoke an ABAP **function module** or **class method** and use its returning value in the rule. This is the escape hatch to full ABAP.
- **Function Call** — invoke another **BRF+ function**, enabling modular, reusable rule logic.

---

## How a rule ties it together

A rule reads like: **"If `<condition>` then set `<result>` to the value of `<expression>`."** For the discount example, the ruleset's rule is essentially:

```
THEN  process expression DT_DISCOUNT  (the decision table)
      and set DISCOUNT_PERCENT to its result.
```

The function is called from ABAP, the context is filled, the ruleset runs the rule, the rule evaluates the decision table, and the result flows back. (Calling from ABAP is File 03.)

---

## Quick self-test

1. What is the difference between a decision table and a decision tree, and when do you prefer each?
2. Why should a decision table always have a catch-all row?
3. What do "first match" vs "all matches" modes change?
4. Which expression type lets you call an ABAP function module or method?
5. Which expression type would you use for `QUANTITY * UNIT_PRICE`?
6. How does a rule connect to an expression?
