# 04 — NACE / Output Determination (the Condition Technique)

> The classic ECC-era output mechanism, still relevant in S/4HANA for certain scenarios. Transaction: `NACE`.

---

## What NACE is

NACE is the entry transaction for **Message Control** (also called **output determination** or the **condition technique** for output). It decides, when a business document is saved, **which output to issue** (a printed form, an EDI/IDoc message, an email, a fax), **to whom**, and **how**.

It is the same **condition technique** used in pricing — the same building blocks (access sequences, condition tables, condition records, procedures), applied to output instead of prices.

---

## The core building blocks

| Object | Role |
|--------|------|
| **Output type** (condition type) | The kind of output, e.g. order confirmation, invoice print |
| **Access sequence** | The ordered search strategy for finding a condition record |
| **Condition tables** | The field combinations to check (e.g. sales org + customer) |
| **Condition records** | The actual data: "for this key, send this output via this medium" |
| **Output procedure** | The collection of output types assigned to a document type |
| **NAST** | The central table where determined output records are stored |

---

## How determination works (the flow)

1. A document is created/saved (sales order, delivery, invoice, PO, ...).
2. The document type has an **output procedure** assigned.
3. The procedure lists **output types**; each output type has an **access sequence**.
4. The access sequence checks **condition tables** in order, looking for a matching **condition record**.
5. On a match, an entry is written to **NAST** with the output type, medium, partner, and timing.
6. The output is **processed** — the relevant **form** is rendered and sent via the **transmission medium**.

---

## Condition records — VV11 / VV12 / VV13

Condition records are the actual maintained data that drive determination. They are maintained per application area with create/change/display transactions:

| Transaction | Action | Area |
|-------------|--------|------|
| `VV11` / `VV12` / `VV13` | Create / Change / Display | Sales (SD) output |
| `MN04` / `MN05` / `MN06` | Create / Change / Display | Purchasing (MM) output |

A condition record says, for example: "For sales organisation 1000 and customer 1, issue output type ZORD (order confirmation) by medium 1 (print), to partner function SH, using print program/form Zxxx, immediately."

---

## Transmission mediums

The medium determines how the output is sent. Common mediums:

| Medium | Meaning |
|--------|---------|
| 1 | Print output |
| 2 | Fax |
| 5 | External send (email) |
| 6 | EDI |
| 8 | Special function |
| 9 | Events (SAP Business Workflow) |
| A | Distribution (ALE) |
| T | Tasks (workflow) |

> Remember this list — it matters for S/4HANA (File 05). The new BRF+-based output **cannot** handle several of these mediums (EDI, ALE, workflow, special function), so those scenarios stay on NAST.

---

## The form technologies

NACE-determined output is rendered by a **form**, historically one of:
- **SAPscript** — the oldest form technology (transaction SE71).
- **Smartforms** — graphical, more flexible (transaction SMARTFORMS).
- **Adobe Forms / PDF-based forms** — interactive and print forms (transaction SFP).

The output type's processing routine points to a **print program** plus the **form** to use.

---

## Where you configure it (NACE menu)

Inside transaction NACE you select an **application** (e.g. V1 = Sales, V2 = Shipping, V3 = Billing, EF = Purchasing) and then maintain:
- Output types (condition types)
- Access sequences
- Condition tables
- Procedures
- Assignment of procedures to document types
- The output programs and forms per output type

---

## Monitoring and reprocessing

- Output status sits in **NAST** (and is visible in the document's output/messages screen).
- Failed or pending output can be reprocessed; status values indicate whether output was successfully processed.
- For SD, the document's "Output" screen shows determined messages and their processing status (green/yellow/red).

---

## Strengths and weaknesses

**Strengths:** extremely flexible, well understood, handles complex multi-receiver scenarios easily through condition records, supports every transmission medium (print, EDI, ALE, fax, email, workflow).

**Weaknesses:** the condition technique is complex to set up, custom logic often required ABAP routines (leading to modifications), and it is not aligned with the modification-free, cloud-ready direction of S/4HANA.

---

## Quick self-test

1. What does NACE / Message Control decide when a document is saved?
2. Name the building blocks of the condition technique for output.
3. What is the NAST table?
4. Which transactions maintain SD output condition records?
5. Which transmission mediums are relevant to remember, and why (hint: File 05)?
6. Name the three form technologies an output type can use.
