# 05 — BRF+ vs NACE & S/4HANA Output Management

> The comparison the topic is really about: how output worked in ECC (NACE) vs how it works in S/4HANA (the new Output Management framework, which uses BRF+ for determination).

---

## First, get the framing right

"BRF+ vs NACE" is slightly apples-to-oranges, and saying so is a strong answer:

- **BRF+** is a **general-purpose business rules engine** (Files 01–03). It is not an output tool — it derives values for release strategies, discounts, validations, and many other things.
- **NACE** is specifically **output determination** via the condition technique (File 04).

What people actually mean is: **output determination in ECC (NACE) vs output determination in S/4HANA (the new Output Management framework)** — and the new framework happens to use **BRF+ as its determination engine**.

### The terminology nuance SAP insists on

Do **not** call it "BRFplus Output Management." SAP's own position: BRFplus is only used for **configuration/determination**, not for processing the output. The framework's name is **SAP S/4HANA Output Management** (it includes a reuse service called **Output Control**). BRF+ is just the brain that decides *what* to output; the framework renders and sends it.

---

## How output works in S/4HANA (the new framework)

By design the new output management has **cloud qualities**: extensibility, multitenancy, and **modification-free** configuration. The differences from NACE:

- **Determination** uses **BRF+ rules** (typically decision tables) instead of access sequences and condition records. Rule input is the "context"; rule output is the "result" (output type, channel, form, receiver, etc.).
- **Configuration** is centralized in the **Output Parameter Determination** app — transaction **OPD**.
- **Forms** are standardized on **Adobe Forms** rendered via the **Adobe Document Server** (master form templates / fragments), not the SAPscript/Smartforms mix.
- Output types are no longer NACE "message types" but **abstract output definitions** (e.g. an order confirmation, an invoice print).

### Typical S/4HANA setup steps (OPD)

1. **Application Object Type Activation** — activate the business document (Sales Order, Billing Document, Purchase Order, ...) for the new output management.
2. **Define Output Types** — the abstract outputs to issue.
3. **Assign Form Templates / Email Templates** — Adobe form per output type, email body templates.
4. **Define channels** (print, email, ...) and receiver/role determination.
5. **Maintain BRF+ determination rules** — the decision tables that derive the output parameters.

### The processing chain

```
Document created
   -> S/4HANA Output Management asked to issue output
      -> calls BRF+ to DETERMINE output type, channel, form template, receiver
      -> renders the Adobe form
      -> sends via the channel (print/email/...)
```

BRF+ does only the "decide" step; the framework does "render and send."

---

## Side-by-side comparison

| Aspect | NACE / NAST (ECC & still in S/4) | S/4HANA Output Management (BRF+) |
|--------|----------------------------------|----------------------------------|
| Determination | Condition technique (access seq + condition records) | BRF+ rules (decision tables) |
| Config transaction | `NACE`, `VV11/12/13`, `MN04...` | `OPD` (Output Parameter Determination) |
| Central store | NAST table | Output management framework tables |
| Forms | SAPscript / Smartforms / Adobe | Adobe Forms only (Adobe Document Server) |
| Modification-free | No (often needs ABAP routines) | Yes (cloud-ready, extensible) |
| Multi-receiver / complex cases | Easy via condition records | Possible but can need complex rules |
| Transmission mediums | All (print, email, fax, EDI, ALE, workflow) | Limited (no EDI/ALE/workflow/special function) |
| Direction | Legacy / backward compatibility | SAP's strategic target |

---

## The crucial real-world point: they coexist

This is what makes a strong, senior-sounding answer. In S/4HANA **both frameworks exist side by side**; which one applies depends on the document type and configuration.

- **BRF+-based output is mandatory for some documents:** Purchase Orders from S/4HANA **1511**; NAST-based output is not available for new SD Billing documents from **1511**, extended to sales order management from **1602**.
- **NAST is still required for some scenarios:** BRF+-based output **cannot** handle transmission mediums like **EDI, distribution (ALE), workflow events, and special function** — those stay on NAST. So EDI/IDoc output is typically still NAST.
- **You can revert to NAST** where needed: SAP Note **2267376** and the "Manage Application Object Type Activation" customizing let you enable/disable NAST vs BRF+ output per document. Many early projects stayed on NAST while BRF+ matured.

### An honest practitioner caveat

Newer does not always mean easier. Classic requirements like sending different versions of the same invoice to different receivers — trivial with NAST condition records — can require complex rule modeling in BRF+. For greenfield S/4HANA, BRF+ output is the strategic choice; for some intricate legacy scenarios, teams still lean on NAST.

---

## The one-paragraph interview answer

NACE is the ECC-era output determination using the condition technique — output types, access sequences, condition records in NAST, rendering SAPscript/Smartforms/Adobe forms. S/4HANA replaced this for many documents with a new, modification-free, cloud-ready **Output Management** framework that uses **BRF+** as the determination engine (decision tables derive the output parameters) and **Adobe Forms** for rendering, configured centrally in the **OPD** app. BRF+ handles only determination, not processing. The two coexist in S/4HANA — BRF+ is mandatory for some documents (PO from 1511), but NAST is still needed for transmission mediums like EDI/ALE/workflow and can be reactivated via SAP Note 2267376 where required.

---

## Quick self-test

1. Why is "BRF+ vs NACE" technically apples-to-oranges?
2. Why does SAP say not to call it "BRFplus Output Management"?
3. What transaction centralizes S/4HANA output configuration, and what does BRF+ do within it?
4. Which form technology is the S/4HANA target?
5. From which release is BRF+ output mandatory for Purchase Orders?
6. Name transmission mediums that still require NAST in S/4HANA.
7. How can you revert a document to NAST-based output?
