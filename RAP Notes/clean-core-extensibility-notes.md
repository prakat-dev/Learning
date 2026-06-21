<div align="center">

# SAP Clean Core & Extensibility

**A Personal Learning Reference**

`Clean Core` `Extensibility` `ABAP Cloud` `RAP` `BTP` `S/4HANA`

---

*Built section by section from hands-on learning. Validated against official SAP documentation.*

</div>

---

## Table of Contents

| # | Section | What it Covers |
|---|---------|---------------|
| 1 | [Why Clean Core](#section-1---why-clean-core) | The problem clean core solves |
| 2 | [Types of Extensibility](#section-2---types-of-extensibility) | Key User, Developer, Side by Side |
| 3 | [The Three Tier Model](#section-3---the-three-tier-model) | Tier 1, 2, 3 and the A-D evolution |
| 4 | [Developer Extensibility in Detail](#section-4---developer-extensibility-in-detail) | On-stack development inside S/4HANA |
| 5 | [Side by Side Extensibility in Detail](#section-5---side-by-side-extensibility-in-detail) | BTP, Service Consumption, Transports |
| 6 | [Quick Reference](#section-6---quick-reference) | When to use what |
| 7 | [References](#references) | Official SAP sources |

---

## Section 1 - Why Clean Core

In the classic SAP world there were no boundaries. Developers could modify SAP standard code directly, call any internal function module, write native SQL against any table and use APIs that SAP never intended to be public. Over time systems became so heavily customized that upgrading to a new SAP version turned into a massive project. Companies would spend months testing and fixing custom code before they could upgrade. Some stopped upgrading entirely because it was too risky.

SAP realized that for S/4HANA Cloud to work there needed to be a clear boundary between SAP standard code and customer code. That is what clean core is about. Keep the SAP core untouched and build all custom development using only released and stable APIs and extension points. This way SAP can release upgrades without breaking customer code and customers can adopt innovations without fear of regression.

> **Key Takeaway**
> Clean core = untouched SAP standard + all custom code built on released APIs only.

---

## Section 2 - Types of Extensibility

SAP defines three types of extensibility under clean core. Each serves a different audience and use case.

### 2.1 Key User Extensibility (Type 1)

This requires no code. Business users use SAP provided Fiori tools to add custom fields, create custom logic with BAdIs, adjust UIs and manage form templates. Everything is done through the browser without needing Eclipse or any development tools. The main advantage is that simple extensions can be realized quickly without involving a developer. SAP refers to these tools as extensibility apps.

### 2.2 Developer Extensibility (Type 2)

This is where RAP lives. ABAP developers build custom applications or extend existing SAP RAP business objects using released extension points. The development happens inside the S/4HANA system itself on the application server. Since the code runs inside S/4HANA it has access to released SAP CDS views, released APIs and released BAdIs. Only released objects can be used. Unreleased function modules, internal tables and non-released APIs are not accessible.

There are two things that can be done here:

1. **Extend an existing SAP RAP business object** by adding custom fields, validations or actions using released extension points.
2. **Build a completely new custom RAP application** inside S/4HANA with its own tables and CDS views. If the app needs standard SAP data like business partner or cost center it can read from released SAP CDS views directly since it runs in the same system.

### 2.3 Side by Side Extensibility (Type 3)

This means building a completely separate application on SAP BTP that connects to S/4HANA through APIs. The app runs outside S/4HANA on BTP infrastructure. It can be built in ABAP using SAP BTP ABAP Environment (also known as Steampunk), or in other languages like Node.js or Java using SAP CAP.

If built with ABAP on BTP, the developer gets a blank ABAP runtime. It does not have any S/4HANA tables, no SAP standard function modules and no SAP standard CDS views. It is an empty ABAP system. Custom tables, CDS views and RAP objects are created there. If the app needs data from S/4HANA it has to call back into S/4HANA through OData APIs or events. There is no direct CDS view access to S/4HANA data because the two systems are completely separate.

It is also completely valid to build a standalone app on BTP that does not connect to S/4HANA at all. Side by side just means the app runs on BTP. The connection to S/4HANA is optional.

> **At a Glance**
>
> | Type | Who | Where | Code Required |
> |------|-----|-------|---------------|
> | Key User | Business users | Inside S/4HANA | No |
> | Developer | ABAP developers | Inside S/4HANA | Yes (ABAP Cloud) |
> | Side by Side | Developers | On BTP (separate) | Yes (ABAP, CAP, etc.) |

---

## Section 3 - The Three Tier Model

SAP introduced a tier based development model to enforce clean core, primarily for S/4HANA Cloud Private Edition and on-premise systems where both classic ABAP and ABAP Cloud need to coexist.

### 3.1 Tier 1 - ABAP Cloud

This is the strictest and preferred tier. Only released SAP APIs, released CDS views and released objects can be used. The ABAP language version is set to `ABAP for Cloud Development` and the syntax check enforces the restrictions. RAP with `strict(2)` falls into this tier. On S/4HANA Cloud Public Edition this is the only tier available. On BTP ABAP Environment this is also the only option. All new development should target Tier 1.

### 3.2 Tier 2 - Cloud API Enablement

This exists as a bridge for situations where a released API is not yet available in Tier 1. A custom wrapper class is created in Tier 2 that accesses the unreleased SAP object and exposes it through a released interface. Tier 1 code then calls the wrapper instead of the unreleased API directly. The wrapper uses `Standard ABAP` language version (not ABAP Cloud) so it can access unreleased objects. Once SAP releases a proper API for that object the wrapper can be removed.

### 3.3 Tier 3 - Classic ABAP

This is the old way with no restrictions. Full access to all SAP objects, internal tables, unreleased APIs, user exits and modifications. Available on on-premise and private cloud only. Not available on BTP or public cloud. SAP is strongly discouraging new development in this tier. Existing Tier 3 code should be gradually migrated towards Tier 1.

### 3.4 Update: A to D Level Classification (2025)

SAP has recently evolved the 3-tier model into a four level A to D classification system for rating extension compliance.

| Level | Description | Clean Core? |
|-------|-------------|-------------|
| **A** | ABAP Cloud on-stack or side by side on BTP (SAP Build, CAP, ABAP Cloud) | Yes - target for all extensions |
| **B** | Released classical extensibility techniques, still supported and upgrade stable | Mostly - acceptable |
| **C** | Unreleased SAP objects with wrappers (similar to old Tier 2) | Partially - temporary measure |
| **D** | Classic ABAP with no restrictions (similar to old Tier 3) | No - highest risk |

> **Note**
> The 3-tier model is still widely referenced in documentation and training material but the A to D levels are the current SAP recommendation for classifying extensions.

---

## Section 4 - Developer Extensibility in Detail

Developer extensibility happens inside the S/4HANA application server. The code runs in the same system as SAP standard.

### 4.1 What is Accessible

- Released SAP CDS views like `I_BusinessPartner` or `I_Product`
- Released BAdIs for extending standard business logic
- Released RAP business object extension points (custom fields, validations, actions)
- All released APIs listed in the SAP Business Accelerator Hub

### 4.2 What is Not Accessible

- Unreleased function modules
- Unreleased database tables
- Unreleased BAdIs
- SAP GUI dynpro extensions
- Any object not explicitly released by SAP

The ABAP syntax check enforces this when the language version is set to `ABAP for Cloud Development`.

### 4.3 Development and Transport

Development is done in Eclipse ADT connected to the S/4HANA development system. Objects are transported through the standard SAP transport system (CTS) from development to quality to production, the same way as classic ABAP. The only difference is that the ABAP language version is set to `ABAP for Cloud Development` which restricts what can be used.

---

## Section 5 - Side by Side Extensibility in Detail

Side by side extensibility means the app runs on BTP, completely separate from S/4HANA.

### 5.1 BTP ABAP Environment (Steampunk)

This is a full ABAP application server provisioned as a managed service on BTP. SAP manages the infrastructure, the ABAP runtime and the HANA Cloud database underneath. From a developer perspective it looks like a normal ABAP system. Eclipse ADT connects to it, CDS views and RAP objects are created in it and objects are activated the same way.

The key difference from S/4HANA is that BTP ABAP Environment is empty. There are no SAP standard tables, no standard CDS views and no standard function modules. Only custom objects exist there.

### 5.2 Landscape and Transport

Side by side apps have their own separate landscape independent of the S/4HANA landscape.

```
S/4HANA landscape:   S/4 Dev  ──>  S/4 Quality  ──>  S/4 Production
                        (independent, no cross-connection)
BTP landscape:       BTP Dev  ──>  BTP Quality  ──>  BTP Production
```

Code on BTP moves through the BTP landscape using **gCTS** (Git enabled Change and Transport System). Each software component in BTP has a 1:1 relationship with a Git repository managed by SAP. When a transport request is released in the development system it creates a commit in the Git repository. The quality and production systems pull the changes from Git.

### 5.3 Connecting to S/4HANA

If the BTP app needs data from S/4HANA it calls S/4HANA through OData APIs. The connection is set up using Communication Arrangements and Destinations.

```
BTP ABAP System                              S/4HANA System
┌───────────────────────┐                   ┌───────────────────────┐
│                       │                   │                       │
│  Communication        │    OData API      │  Communication        │
│  Arrangement     ────────────────────>    │  Arrangement          │
│  (scenario name)      │    via HTTP       │  (exposes the API)    │
│                       │                   │                       │
│  Destination          │                   │  Communication        │
│  (in BTP Cockpit)     │                   │  User                 │
│                       │                   │  (authorizes BTP)     │
└───────────────────────┘                   └───────────────────────┘
              │                                       │
              └────── Cloud Connector (if on-prem) ───┘
```

**How each environment connects to the right S/4 system:**

The code never hardcodes URLs or system addresses. It references a logical Communication Scenario name. The framework resolves it to the right destination.

```
Code says:        call scenario 'Z_MY_S4_SCENARIO'

BTP Dev has:      Z_MY_S4_SCENARIO  ──>  https://s4dev.company.com
BTP Quality has:  Z_MY_S4_SCENARIO  ──>  https://s4qa.company.com
BTP Prod has:     Z_MY_S4_SCENARIO  ──>  https://s4prod.company.com
```

The scenario name is the same everywhere. The destination behind it is configured per environment. The code is identical across all systems.

### 5.4 Service Consumption Model

To call an S/4HANA OData API from BTP ABAP, a Service Consumption Model (SRVC) is created in Eclipse. The steps are:

| Step | Action | Result |
|------|--------|--------|
| 1 | Download the `$metadata.xml` from the S/4HANA OData service or SAP Business Accelerator Hub | XML file describing every entity, field and operation |
| 2 | Create a Service Consumption Model in Eclipse ADT and import the metadata | Eclipse processes the XML |
| 3 | Eclipse generates artifacts | Abstract CDS entities (data shape) and proxy classes (HTTP handling) |
| 4 | Use the proxy in ABAP code with a Communication Arrangement reference | Typed ABAP data returned, no JSON parsing needed |

**Why the abstract entity is needed:**
There is no local database table for remote data. In a normal CDS view the table definition tells CDS what fields exist. For remote data the abstract entity serves the same purpose. It describes the data shape so that ABAP code can work with it using standard types.

**Generated consumption code pattern:**

```abap
DATA(lo_destination) = cl_http_destination_provider=>create_by_comm_arrangement(
    comm_scenario  = 'Z_MY_S4_SCENARIO'
    service_id     = 'Z_BUPA_ODATA' ).

DATA(lo_http_client) = cl_web_http_client_manager=>create_by_http_destination(
    i_destination = lo_destination ).
```

The communication scenario name `Z_MY_S4_SCENARIO` is the logical reference. It resolves to different S/4HANA systems depending on which BTP environment the code is running in.

### 5.5 When No S/4HANA Connection is Needed

If the app is standalone with its own data, the Service Consumption Model, Communication Arrangements and Destinations are not needed at all. Just create custom tables, CDS views, RAP behaviour and a service binding. Everything is local on BTP.

---

## Section 6 - Quick Reference

> **When to Use What**

| Situation | Approach |
|-----------|----------|
| Simple UI change or custom field, no coding needed | Key User Extensibility |
| Tight coupling to S/4HANA data, released extension points available | Developer Extensibility (on-stack) |
| Standalone app or loosely coupled integration through APIs | Side by Side Extensibility (BTP) |
| Released API does not exist yet for Tier 1 | Tier 2 wrappers as temporary bridge |
| S/4HANA Cloud Public Edition | Key User + Developer Extensibility only. No Tier 3, no modifications |
| BTP ABAP Environment | ABAP Cloud (Tier 1 equivalent) only. No classic ABAP |

---

## References

| Source | Topic |
|--------|-------|
| SAP ABAP Extensibility Guide (Aug 2025 Update) | Tier model, A-D levels, clean core guidelines |
| SAP Help Portal - Integrating ABAP Environment with S/4HANA Cloud | Communication Arrangements, Service Consumption |
| SAP Community - Service Consumption Model for OData | SRVC walkthrough, proxy generation |
| SAP Help Portal - Software Components in BTP ABAP Environment | gCTS, transport, landscape management |
| SAP Learning - Exploring Clean Core Compliance | Extension types, decision framework |
| SAP Learning - Practicing Clean Core Extensibility | Best practices, governance |

---

<div align="center">

*Last updated: June 2026*

</div>
