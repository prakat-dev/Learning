# BRF+ & NACE — Complete Notes Set

> Day 1, Topic 2 of the 5-Day SAP ABAP Learning Plan. Balanced notes plus examples. Current to S/4HANA Output Management.

## How to use

Each file is self-contained and ends with a **Quick self-test**. Read in order for a full pass, or jump to the comparison (File 05) for the interview-critical material.

## Contents

- **01 — BRF+ Fundamentals** — what BRF+ is, why it exists, the Application → Function → Ruleset → Rule → Expression hierarchy, data objects, lifecycle, simple vs expert mode
- **02 — BRF+ Expressions** — decision table (worked discount example), decision tree, formula, case, DB lookup, procedure/function call
- **03 — Calling BRF+ from ABAP** — instance API vs `CL_FDT_FUNCTION_PROCESS`, filling context, reading result, workbench-generated code, simulation/trace, performance, versioning
- **04 — NACE / Output Determination** — the condition technique: output types, access sequences, condition records (VV11/12/13), NAST, transmission mediums, form technologies
- **05 — BRF+ vs NACE & S/4HANA Output Management** — the real comparison: S/4HANA Output Management (OPD), the terminology nuance, side-by-side table, mandatory-from-version timeline, when NAST is still required, reverting to NAST

## Key takeaways

- **BRF+** is a general business-rules engine; **NACE** is output determination. They are not the same kind of thing.
- **S/4HANA Output Management** uses BRF+ only for **determination**, Adobe Forms for rendering, configured in **OPD**.
- Both frameworks **coexist** in S/4HANA: BRF+ mandatory for some documents (PO from 1511), NAST still required for EDI/ALE/workflow mediums.

## Suggested study order

01 → 02 → 03 build BRF+ as a tool; 04 covers the classic side; 05 ties it together. If short on time, read **05 first**, then 01 and 04 for the supporting detail.
