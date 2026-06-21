<div align="center">

# SAP RAP Learning Notes

**ABAP RESTful Application Programming Model**

`RAP` `CDS` `Behaviour Definition` `OData` `Fiori` `Clean Core` `ABAP Cloud`

---

*The overview and index page for my RAP learning series. Start here, then jump into the detailed documents for each layer.*

</div>

---

## Table of Contents

| # | Section | What it Covers |
|---|---------|---------------|
| 1 | [What is RAP and What Does It Do](#section-1---what-is-rap-and-what-does-it-do) | The purpose of RAP and the problem it solves |
| 2 | [Clean Core and Why It Matters](#section-2---clean-core-and-why-it-matters) | Tiers, extensibility types, where RAP fits |
| 3 | [The RAP Architecture](#section-3---the-rap-architecture) | The layered structure end to end |
| 4 | [The Development Objects](#section-4---the-development-objects) | Every file in a RAP app and what it does |
| 5 | [Implementation Types](#section-5---implementation-types) | Managed, unmanaged, managed with unmanaged save |
| 6 | [Key Concepts](#section-6---key-concepts) | Draft, locking, ETag, determinations, validations, actions |
| 7 | [Document Index](#section-7---document-index) | Links to every detailed note |

---

## Section 1 - What is RAP and What Does It Do

RAP stands for ABAP RESTful Application Programming Model. It is a framework from SAP for building transactional applications and APIs on S/4HANA and BTP. RAP can be used to build Fiori apps but that is only one use case. It is equally used to build OData APIs consumed by external systems, web APIs for system to system integration, local APIs consumed within the same SAP system and business events for event driven scenarios. In short RAP is how anything transactional is built in the modern ABAP world.

Before RAP, building an OData service meant doing everything manually. Creating the service in SEGW, implementing DPC_EXT methods for every operation, handling drafts with custom code, managing locks manually and wiring all the layers together by hand. A simple CRUD service could take weeks, every developer solved the same problems differently and none of it was cloud ready.

RAP changes that. The data model is described using CDS views, the allowed operations are declared in a behaviour definition and only the business logic unique to the use case needs to be written. Everything else like OData service generation, draft handling, database persistence, locking and concurrency is handled by RAP automatically. It is an opinionated framework. Follow its structure and patterns and in return it provides a production-ready service with far less code and a consistent design.

RAP is also the preferred development model for clean core. Whether it is side by side extensions on BTP or developer extensions on S/4HANA Cloud, RAP is how SAP expects it to be built.

---

## Section 2 - Clean Core and Why It Matters

In the classic SAP world there were no boundaries. Developers could modify SAP standard code directly, call any internal function module, write native SQL against any table and use APIs that SAP never intended to be public. Over time systems became so heavily customized that upgrading to a new SAP version turned into a massive project. Companies would spend months testing and fixing custom code before they could upgrade. Some stopped upgrading entirely.

SAP realized that for S/4HANA Cloud to work there needed to be a clear boundary between SAP standard code and customer code. That is what clean core is about. Keep the SAP core untouched and build all custom development using only released and stable APIs and extension points.

### ABAP Cloud and the Three Tier Model

To enforce clean core SAP introduced a tier based development model.

Tier 1 is ABAP Cloud or ABAP for Cloud Development. This is the strictest tier. Only released SAP APIs, released CDS views and released objects can be used. No access to internal SAP tables or unreleased function modules. This is mandatory on BTP and S/4HANA Cloud Public Edition. RAP with strict(2) falls into this tier.

Tier 2 is ABAP Cloud with relaxed rules. Same as Tier 1 but unreleased SAP APIs can be wrapped behind a clean interface. If data is needed from a SAP table that has no released CDS view yet, a wrapper class is created in Tier 2 that reads it and exposes it through a released interface. Tier 1 code then calls the wrapper instead of the unreleased API directly.

Tier 3 is Classic ABAP. The old way with full access to everything. Available on on-premise and private cloud only. Not available on BTP or public cloud at all. SAP is discouraging new development here.

The idea is that over time everything moves to Tier 1. Tier 2 exists as a bridge for things SAP has not released yet. Tier 3 is legacy.

### Types of Extensibility

SAP defines three types of extensibility under clean core.

Key User Extensibility requires no code. Business users use SAP provided tools to add custom fields, create custom logic with BAdIs and adjust UIs. It is all done through the browser without needing Eclipse.

Developer Extensibility is where RAP lives. ABAP developers build custom applications or extend existing SAP RAP business objects using released extension points. CDS views, behaviour definitions and ABAP classes are used but only through released APIs.

Side by Side Extensibility means building completely separate applications on BTP that connect to S/4HANA through APIs. These can be built in ABAP using RAP on BTP or in other languages like Node.js using CAP. The app runs outside S/4HANA and calls back into it via OData or events.

A separate set of notes covers clean core and extensibility in depth. This page only summarizes how RAP relates to it.

---

## Section 3 - The RAP Architecture

RAP is built in layers. Each layer has a clear responsibility and sits on top of the one below it. Understanding this layering is the key to understanding RAP, because every development object fits into one of these layers.

```
                          CONSUMER
              (Fiori app, external system, API client)
                              |
                              | OData V2 / V4
                              v
        +-------------------------------------------------+
        |              SERVICE LAYER                      |
        |   Service Definition  -> what is exposed        |
        |   Service Binding     -> protocol and version   |
        +-------------------------------------------------+
                              |
                              v
        +-------------------------------------------------+
        |             PROJECTION LAYER                    |
        |   Projection CDS View -> tailored field subset  |
        |   Projection BDEF     -> exposed operations     |
        |   Metadata Extension  -> UI annotations         |
        +-------------------------------------------------+
                              |
                              v
        +-------------------------------------------------+
        |             BUSINESS OBJECT LAYER               |
        |   Interface CDS Views -> data model             |
        |   Behaviour Definition-> operations and rules   |
        |   Behaviour Class     -> business logic in ABAP |
        +-------------------------------------------------+
                              |
                              v
        +-------------------------------------------------+
        |             PERSISTENCE LAYER                   |
        |   Active Table        -> production data        |
        |   Draft Table         -> work in progress data  |
        +-------------------------------------------------+
```

### What Each Layer Does

The persistence layer is the database. Active tables hold the real data and draft tables hold work in progress when the app is draft enabled.

The business object layer is the heart of RAP. The interface CDS views define the data model and relationships. The behaviour definition declares what operations are allowed (create, update, delete, actions, determinations, validations). The behaviour class holds the ABAP code for the custom logic.

The projection layer tailors the business object for a specific use case. The projection CDS view selects a subset of fields. The projection behaviour definition exposes a subset of operations. The metadata extension holds all the UI annotations that control how a Fiori app renders.

The service layer exposes everything to the outside world. The service definition lists which entities are exposed. The service binding sets the protocol (OData V2 or V4) and generates the actual consumable service.

### Why So Many Layers

The layering exists so the same business object can serve different consumers. One projection might expose all fields for an HR manager Fiori app. Another projection on the same interface views might expose only a few fields for a public API. The core business logic is written once in the business object layer and reused by every projection above it.

---

## Section 4 - The Development Objects

A complete RAP application is made up of several development objects. Here is every object type, what it does and which layer it belongs to. Each has its own detailed note in this series.

| Object | Extension | Layer | Purpose |
|--------|-----------|-------|---------|
| Database Table | `.tabl` | Persistence | Stores active and draft data |
| Interface CDS View | `.ddls` | Business Object | Defines the data model, fields and relationships |
| Behaviour Definition | `.bdef` | Business Object | Declares operations, determinations, validations, actions |
| Behaviour Class | `.clas` | Business Object | ABAP implementation of the custom logic |
| Projection CDS View | `.ddls` | Projection | Tailored field subset for a specific consumer |
| Projection Behaviour Definition | `.bdef` | Projection | Exposes a subset of operations for the projection |
| Metadata Extension | `.ddlx` | Projection | UI annotations (facets, field groups, list columns) |
| Service Definition | `.srvd` | Service | Lists which entities are exposed in the service |
| Service Binding | `.srvb` | Service | Sets the protocol (OData V2/V4) and activates the service |

### Supporting Objects

| Object | Purpose |
|--------|---------|
| Domain | Defines fixed value lists and data characteristics (e.g. gender, marital status) |
| Data Element | Reusable field definition with labels |
| Value Help CDS View | Provides dropdown values for a field |
| Number Range Object | Generates sequential numbers for business keys |
| Virtual Element Class | Calculates field values at runtime that are not stored in the database |

### How They Connect

```
Database Table
    |
    v
Interface CDS View  <----  Behaviour Definition  ---->  Behaviour Class
    |                              |
    v                              v
Projection CDS View <----  Projection Behaviour Definition
    |                              ^
    v                              |
Metadata Extension                 |
    |                              |
    v                              |
Service Definition  ---------------+
    |
    v
Service Binding
    |
    v
OData Service (consumed by Fiori or external systems)
```

---

## Section 5 - Implementation Types

When a behaviour definition is created, one of three implementation types is chosen. This decides how much RAP does automatically versus how much the developer handles manually.

### Managed

RAP does everything. It handles create, read, update, delete, draft, locking and database persistence automatically. The developer only writes the business logic unique to the app (determinations, validations, actions). This is the default and most common type, used when the app owns its data and the data lives in a custom table.

Used when: building a new transactional app on a custom table. This is what most apps use, including the employee master app.

### Unmanaged

RAP does almost nothing for persistence. The developer implements every operation manually, including how create, update and delete write to the database. This is used when wrapping a legacy system that already has its own persistence logic, such as an existing BAPI or function module that must be called to save data.

Used when: integrating with legacy code that already has its own save logic that cannot be bypassed.

### Managed with Unmanaged Save

A hybrid. RAP manages everything during the interaction phase (draft, locking, buffering) but at the final save step the developer takes over and writes to the database manually, often by calling a BAPI. This gives the convenience of managed for the user interaction while still routing the actual save through legacy logic.

Used when: the app should behave like a modern managed app but the final save must go through an existing BAPI or legacy persistence.

### Quick Comparison

| Aspect | Managed | Unmanaged | Managed + Unmanaged Save |
|--------|---------|-----------|--------------------------|
| Who handles CRUD | RAP | Developer | RAP for interaction, developer for save |
| Who handles persistence | RAP | Developer | Developer (at save only) |
| Draft support | Yes, automatic | Manual, with effort | Yes, automatic |
| Typical use | New custom app | Wrapping legacy persistence | Modern UX over a BAPI save |
| Effort | Lowest | Highest | Medium |

A detailed note covers each implementation type with full examples.

---

## Section 6 - Key Concepts

These concepts appear across every RAP app. Each has its own detailed note. This is just the high level summary.

### Draft

Draft lets users save incomplete work without committing it to the active table. When draft is enabled, every entity gets a draft table. Work in progress goes to the draft table first and only moves to the active table when the user activates it. Draft also enables features like recovering unsaved work after closing the browser.

### Locking and Concurrency

RAP uses pessimistic locking (exclusive locks) to ensure only one user edits an instance at a time, and optimistic concurrency control using an ETag to detect if data changed between read and save. The total ETag covers the whole business object and is used during draft resume to confirm the underlying data has not changed.

### Determinations

A determination is logic that runs automatically when specified fields change or at save time. It is used to calculate or derive values. For example calculating a full name from first and last name, or generating a sequential ID. Determinations run on modify or on save.

### Validations

A validation is logic that checks data and raises an error if something is wrong. It runs on save or on specified field changes. For example checking that a date of birth makes the employee at least 18 years old. A failed validation blocks the save and shows a message.

### Actions

An action is a custom operation beyond standard create, update and delete. It appears as a button in the Fiori app. For example an Approve button, a Terminate button or an action that initializes related data. Actions can have input parameters and can return results.

### Field Control

Field control governs whether fields are mandatory, read-only or hidden. It can be static (always read-only) declared in the behaviour definition, or dynamic (read-only based on a condition) calculated at runtime.

### EML (Entity Manipulation Language)

EML is the ABAP language used inside behaviour implementations to read and modify RAP entities. Instead of SELECT and UPDATE on tables, EML works through the RAP framework so it respects draft, locking and the transactional buffer. It uses statements like READ ENTITIES and MODIFY ENTITIES.

---

## Section 7 - Document Index

This series is split into focused documents. Each is self contained and can be read on its own.

### Foundations

| Document | Topic |
|----------|-------|
| `rap-01-data-modeling` | Tables, CDS interface views, associations, compositions, data source types |
| `rap-02-behaviour-definition` | Managed, strict, draft, field controls, determinations, validations, actions |
| `rap-03-behaviour-implementation` | Handler classes, local classes, method signatures |
| `rap-04-eml` | Entity Manipulation Language, READ and MODIFY ENTITIES, %tky, %cid |
| `rap-05-error-handling` | reported, failed, messages, state areas, field specific errors |
| `rap-06-projection-layer` | Projection views, provider contract, value helps, text elements |
| `rap-07-metadata-extensions` | UI annotations, facets, field groups, list and object pages |
| `rap-08-service-layer` | Service definition, service binding, OData V2 and V4 |
| `rap-09-draft` | Draft tables, draft lifecycle, draft actions |
| `rap-10-virtual-elements` | SADL exit, calculated fields |
| `rap-11-side-effects` | Field triggers, UI refresh |

### Advanced Scenarios

| Document | Topic |
|----------|-------|
| `rap-12-implementation-types` | Managed, unmanaged, managed with unmanaged save |
| `rap-13-app-types` | Transactional, analytical, read-only |
| `rap-14-extending-standard-bos` | Extending SAP RAP business objects |
| `rap-15-odata-apis` | Building OData APIs, provider contracts |
| `rap-16-custom-entity` | Custom entities and the query implementation class |
| `rap-17-bapi-in-rap` | Wrapping BAPIs, unmanaged save pattern |
| `rap-18-released-apis` | Finding released APIs and BAPIs, clean core checks |
| `rap-19-authorization` | Global and instance authorization |
| `rap-20-feature-control` | Static and dynamic feature control |
| `rap-21-numbering-patterns` | Managed, early and late numbering |
| `rap-22-actions-with-parameters` | Action input, output, factory actions |
| `rap-23-transactional-buffer` | How RAP manages data in memory before save |
| `rap-24-testing-rap` | ABAP Unit, test doubles, CDS test doubles |
| `rap-25-debugging-rap` | Breakpoints, tracing, common issues |
| `rap-26-best-practices` | Naming, error handling, performance |

### Related Notes

| Document | Topic |
|----------|-------|
| `clean-core-extensibility-notes` | Clean core, tiers, BTP, transports |
| `key-user-extensibility` | No-code extensibility tools and walkthroughs |

---

<div align="center">

*Last updated: June 2026*

</div>
