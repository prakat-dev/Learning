<div align="center">

# Key User Extensibility

**SAP S/4HANA Cloud - In-App / No-Code Extensions**

`Key User` `In-App Extensibility` `Custom Fields` `Custom Logic` `Custom CDS Views` `Custom Business Objects` `UI Adaptation` `Business Context`

---

*Covers the complete set of key user in-app extensibility tools available in S/4HANA Cloud.*
*Validated against official SAP documentation, SAP Learning and SAP Community content.*

</div>

---

## Table of Contents

| # | Section | What it Covers |
|---|---------|---------------|
| 1 | [What is Key User Extensibility](#section-1---what-is-key-user-extensibility) | Overview, who it is for, where to find the tools |
| 2 | [Business Contexts](#section-2---business-contexts) | What they are and how to find the right one |
| 3 | [The Complete Toolbox](#section-3---the-complete-toolbox) | Every tool available and what it does |
| 4 | [Walkthrough - Adding a Custom Field](#section-4---walkthrough---adding-a-custom-field) | Step by step example |
| 5 | [Walkthrough - Adding Custom Logic](#section-5---walkthrough---adding-custom-logic) | Step by step BAdI example |
| 6 | [Walkthrough - UI Adaptation](#section-6---walkthrough---ui-adaptation) | Step by step Adapt UI example |
| 7 | [Checking if an App is Extensible](#section-7---checking-if-an-app-is-extensible) | All the ways to verify extensibility support |
| 8 | [SAP GUI Transactions for Extensibility](#section-8---sap-gui-transactions-for-extensibility) | SCFD_REGISTRY, SCFD_EUI and others |
| 9 | [Transporting Extensions to Production](#section-9---transporting-extensions-to-production) | Public Cloud vs Private Cloud transport |
| 10 | [Limitations](#section-10---limitations-and-when-to-switch-to-developer-extensibility) | When to switch to Developer Extensibility |
| 11 | [References](#references) | Official SAP sources |

---

## Section 1 - What is Key User Extensibility

Key User Extensibility (also called In-App Extensibility) is a set of low-code/no-code tools provided by SAP that allow business users and administrators to extend and adapt SAP S/4HANA without writing ABAP code and without involving a developer. The changes are made directly in the Fiori Launchpad using dedicated Fiori apps. Both terms refer to the same set of tools. "Key User" describes who does it and "In-App" describes how it is done.

The target audience is not developers. It is power users, business process owners, administrators and anyone with deep knowledge of the business process but not necessarily deep technical coding skills. SAP calls these people "Key Users".

### Where to Find the Tools

All key user extensibility tools are Fiori apps available on the SAP Fiori Launchpad. The key user needs the business catalog `SAP_CORE_BC_EXT` (Cloud) or `SAP_BASIS_BC_EXT` (on-premise) assigned to their role to access these apps. SAP is gradually replacing `SAP_CORE_BC_EXT` with more granular catalogs like `SAP_CORE_BC_EXT_FLD` for custom fields and `SAP_CORE_BC_EXT_BLE` for custom logic.

The main apps can be found under the Extensibility group on the Fiori Launchpad:

```
Custom Fields and Logic        - create fields and implement BAdIs
Custom Business Objects        - build simple standalone apps
Custom CDS Views               - create analytical data views
Custom Communication Scenarios - configure APIs for custom objects
Extensibility Cockpit          - search and explore all extension options
Extensibility Inventory        - view all existing extensions with dependencies
Export Software Collection     - transport extensions (public cloud)
```

---

## Section 2 - Business Contexts

### What is a Business Context

A business context is a grouping that represents a business object or entity in S/4HANA. It is the central concept behind key user extensibility. When a custom field is created in a business context, SAP automatically propagates that field to all related database tables, CDS views, OData services and Fiori apps under that context. One field creation, everything updated end to end.

Examples of business contexts:

| Business Context | What it Covers |
|-----------------|----------------|
| `PRODUCT` | MARA, MARC and all product/material related tables, CDS views, OData services, Fiori apps |
| `BUSINESS_PARTNER` | Business Partner tables, views and apps |
| `FINS_CODING_BLOCK` | ACDOCA and all accounting coding block related objects |
| `CUSTOMER` | Customer Master related objects |
| `PURCHASE_ORDER_ITEM` | Purchase Order item tables and apps |
| `MM_PURREQN_ITEM` | Purchase Requisition item tables and apps |
| `COST_CENTER` | Cost Center master data |
| `PROFIT_CENTER` | Profit Center master data |

There are over 350 business contexts available in S/4HANA. Each business context has a defined capacity. Different extension types consume varying amounts of this capacity. For example a 300-character free-text field takes up significantly more capacity than a 30-character field.

One business context can serve multiple Fiori apps. For example the `PRODUCT` business context is used by Manage Product Master Data (F1602), Create Material and several other product related apps. So the relationship is not one app to one context but one context to many apps.

### How to Find the Right Business Context

There are three approaches depending on the situation.

#### Approach 1 - Start from inside the app (easiest, no lookup needed)

The Custom Fields app can be opened directly from within an extensible Fiori application by choosing "Add Field" in the UI Adaptation mode. When opened this way the correct business context is pre-selected automatically.

```
Open the Fiori app to extend (e.g. Manage Customer Master)
    |
Click user icon (top right) -> Adapt UI
    |
Click "Add Field"
    |
Custom Fields app opens with the correct business context pre-selected
    |
No need to know the business context name at all
```

This is the most direct path. The app itself tells the system which business context it belongs to.

#### Approach 2 - Search in the Extensibility Cockpit

The Extensibility Cockpit app lets you search across all available extension options. It can be searched by type of extension, solution scope, scope item (business process name in plain English) or business context name.

```
Open Extensibility Cockpit on Fiori Launchpad
    |
Search: "purchase order"
    |
Results show all related business contexts:
    - PURCHASE_ORDER_HEADER
    - PURCHASE_ORDER_ITEM
    - PURCHASE_ORDER_SCHEDULE_LINE
    |
Click on one to see which apps, CDS views, OData services use it
    |
Navigate directly to create the extension
```

The Extensibility Cockpit covers over 2000 extensible CDS views, 700+ extensible OData services and 350+ extensible business contexts. It also shows the remaining capacity for each context.

#### Approach 3 - Look up by database table name

If the database table is known but the business context is not, use transaction `SCFD_REGISTRY` (SAP GUI, on-premise/private cloud only) to look it up.

```
I want to add a field to MARA table
    |
Open SCFD_REGISTRY -> search for MARA
    |
Result: Business Context = PRODUCT
    |
Now use PRODUCT when creating the custom field
```

### What Happens Inside a Business Context

When a business context is opened inside the Custom Fields app, it has multiple tabs showing all the objects connected to it:

| Tab | What it Shows |
|-----|---------------|
| UIs and Reports | Which Fiori apps use this context |
| OData API | Which OData services use this context |
| SOAP API | Which SOAP services use this context |
| BAPI | Which BAPIs use this context |
| Email Templates | Which email templates use this context |
| Form Templates | Which forms use this context |
| Business Scenarios | Which business scenarios use this context |

The number in each tab heading indicates how many objects are assigned. This gives a complete picture of where a custom field will appear once it is enabled for each object.

---

## Section 3 - The Complete Toolbox

Here is every key user extensibility tool available, grouped by category.

### 3.1 UI Flexibility (Adapt UI)

Allows key users to change the layout of existing SAP Fiori apps without coding. Changes can be made for specific user roles or for all users.

| What Can Be Done | Example |
|------------------|---------|
| Hide or show fields | Hide "Payment Terms" field for a specific role |
| Rearrange field positions | Move "Email" field above "Phone" |
| Rename labels | Change "BP Number" to "Customer ID" |
| Add or remove sections | Remove an unused tab from the object page |
| Embed external content | Add an iframe with an external dashboard |

How to access: Open any supported Fiori app, click the user icon (top right), select "Adapt UI". The app enters edit mode and fields become draggable.

### 3.2 Custom Fields

Allows adding new fields to existing SAP standard apps. When a custom field is created, SAP automatically updates the underlying database structures, CDS views, OData services and the UI to make the field available end to end.

| Supported Field Types | Description |
|-----------------------|-------------|
| Text | Free text, configurable length |
| Number | Integer or decimal |
| Quantity / Amount | With unit of measure or currency |
| Date | Date picker |
| Time | Time picker |
| Checkbox | Boolean true/false |
| Code List | Dropdown with fixed values |
| Code List (CDS based) | Dropdown from a CDS view |

Each business context has a capacity limit for custom fields. Complex fields (like long text) consume more capacity than simple ones.

After creating a custom field it needs to be enabled on the relevant UIs using the Adapt UI feature. It also needs to be enabled for search relevance and reporting if required.

#### Modifying a Custom Field After Publishing

Once a custom field is published its core technical properties are locked and cannot be changed.

| Can Change | Cannot Change |
|------------|---------------|
| Field label | Data type (Text, Number, Date, Code List etc.) |
| Tooltip | Field length |
| Where it is enabled (UIs, reports, APIs) | Business context |
| Search relevance | |
| Aggregation behaviour | |
| Code list values (if type is Code List) | |

If a different data type or field length is needed the only option is to delete the old field and create a new one. This means all data in the old field will be lost. This is why SAP emphasizes getting the field definition right at the time of creation.

#### Deleting a Custom Field

Custom fields can be deleted but there is a specific order of operations. The field must be detached from everything before it can be removed.

| Step | Action |
|------|--------|
| 1 | Remove the field from any custom CDS views or custom logic that reference it |
| 2 | Open each tab (UIs and Reports, Email Templates, Form Templates, Business Scenarios, OData APIs, SOAP APIs, BAPIs, IDOCs) and click "Disable Usage" on every enabled object |
| 3 | Save and publish |
| 4 | Click "Delete" |
| 5 | Confirm permanent data deletion |
| 6 | If deletion fails check transaction SCFD_LOG for detailed error information |

When the system asks "Do you want to permanently delete data?" it means exactly that. All data stored in that field across every record in the database is permanently deleted. Not archived, not hidden, not recoverable. If a field had values across 50,000 records, all 50,000 values are wiped.

Applications can become unusable while the deletion process is running.

If the goal is to stop showing the field without losing the data, do not delete it. Instead disable it from all UI tabs and hide it using Adapt UI. The field and its data remain in the database but users can no longer see it.

| Goal | Action |
|------|--------|
| Hide temporarily, keep data | Disable usage on UI tabs + hide with Adapt UI |
| Remove permanently, lose data | Delete the field (data is permanently lost) |

### 3.3 Custom Logic (BAdI Implementations)

Allows key users to add business logic to existing SAP processes by implementing pre-delivered BAdI (Business Add-In) definitions. The logic is written in a simplified ABAP editor within the browser.

| When Logic Runs | Use Case |
|----------------|----------|
| Before Save (Validation) | Check that a custom field is filled before saving |
| After Modification (Determination) | Auto-fill a custom field based on other field values |
| Custom Calculation | Calculate a derived value like tax or discount |

The editor supports basic ABAP operations, field access and simple control flow. It does not support the full ABAP language. SAP provides the XCO (Extension Components) library for common operations.

Custom logic also supports tracing. The Custom Logic Tracing app allows administrators to debug and trace the execution of custom logic implementations to troubleshoot issues.

Important: Key user BAdI implementations are done through the Custom Fields and Logic Fiori app (Custom Logic tab) or the separate Custom Logic app. They are not done through SAP GUI transactions SE18/SE19. Those classic transactions are for traditional developer BAdIs which are a completely different mechanism.

### 3.4 Custom Business Objects

Allows creating simple standalone applications with their own database tables, CDS views and OData services. All generated automatically from a definition in the browser.

A custom business object is essentially a table with fields that the key user defines. SAP generates:
- The database table
- A CDS view on top of the table
- An OData service for CRUD operations
- A basic Fiori UI for maintaining the data

Custom business objects can include:
- Determinations (logic that runs after modification)
- Validations (logic that runs before save)
- OData exposure (for external consumption)

Example use case: A key user creates a "Product Discount Hierarchy" business object to manage discount rules. The object has fields like Product Category, Discount Percentage and Valid From/To dates. SAP generates everything needed to maintain this data through a Fiori app.

### 3.5 Custom CDS Views

Allows creating analytical CDS views through a browser-based tool without writing DDL source code. These views can be used for:

| Usage | Description |
|-------|-------------|
| Analytical reports | Consumed by SAP Analytics Cloud or embedded analytics |
| OData services | Exposed for external consumption |
| Forms and templates | Used as data sources in form templates |
| Integration | Consumed by external systems via API |

The Custom CDS Views app provides a visual editor where the key user selects a data source (an existing released CDS view), picks the fields needed and optionally adds filters, calculated fields or associations.

### 3.6 Custom Communication Scenarios

Allows configuring inbound and outbound API communication for custom objects. Used when:

- A custom business object needs to be filled with data from an external system via OData
- A custom CDS view needs to be exposed as an API
- Custom logic needs to call an external service

The key user creates a communication scenario, adds the relevant OData services and then creates a communication arrangement to link it to a communication system with credentials.

### 3.7 Other Tools

| Tool | What it Does |
|------|-------------|
| Custom Tiles | Create Fiori Launchpad tiles with URL links to external applications |
| Custom Catalog Extensions | Add custom apps to existing Fiori app catalogs |
| Custom Forms | Create or modify output forms (e.g. purchase order printouts) using Adobe Forms Designer |
| E-Mail Template Designer | Create email templates based on CDS view data sources |
| KPI Design | Define custom KPIs based on CDS views |
| Custom Analytical Queries | Build ad-hoc analytical reports using the Query Builder |
| View Browser | Browse available CDS views for analytics |
| Extensibility Cockpit | View all available extension options per business context with capacity info |
| Extensibility Inventory | View all custom extensions in the system with dependency information |
| Maintain Translations | Translate custom field labels and descriptions |

---

## Section 4 - Walkthrough - Adding a Custom Field

Scenario: Adding a "VAT Country" field to the Manage Customer Master app.

### Step 1 - Find the business context (easiest way)

Open the Manage Customer Master app on the Fiori Launchpad. Click user icon (top right) and select "Adapt UI". Click "Add Field". The Custom Fields app opens with the correct business context (CUSTOMER) pre-selected. No lookup needed.

Alternatively, open the Custom Fields and Logic app directly from the Fiori Launchpad and browse or search the business context list.

### Step 2 - Create the custom field

Click "New" to create a new field. Fill in:

```
Business Context : CUSTOMER (pre-selected if opened from the app)
Field Label      : VAT Country
Tooltip          : Country for VAT registration
Field Type       : Text
Field Length     : 3
```

Click "Create". SAP now automatically:
- Adds the field to the underlying database structure
- Adds the field to the relevant CDS views
- Adds the field to the OData service
- Makes the field available for UI placement

### Step 3 - Publish the field

Click "Publish" to activate the field. Until published the field exists only as a draft.

### Step 4 - Check the tabs to see where the field can be used

Open the field details and review the tabs (UIs and Reports, OData API, etc.) to see all the objects this field is now available for. Enable the field for each object where it is needed.

### Step 5 - Enable the field on the UI

The field is created but not yet visible on any app screen. To make it visible:

1. Open the Manage Customer Master app
2. Click user icon (top right) and select "Adapt UI"
3. Navigate to the section where the field should appear
4. Click "Add" or drag the new "VAT Country" field into position
5. Save the UI adaptation

### Step 6 - Enable for reports and search (optional)

Back in the Custom Fields and Logic app, open the field details. Enable "Search Relevance" if the field should be searchable. Enable "Report Usage" if it should be available in analytical reports.

---

## Section 5 - Walkthrough - Adding Custom Logic

Scenario: When the VAT Country field is filled, automatically validate that the VAT number is also provided.

### Step 1 - Open Custom Fields and Logic app

Navigate to the Custom Fields and Logic app on the Fiori Launchpad.

### Step 2 - Switch to the Custom Logic tab

Click on the "Custom Logic" tab. This shows all available BAdI definitions for the business context.

### Step 3 - Find the relevant BAdI

Search for the BAdI related to Customer Master validation. For example "Modify Business Partner" or "Check Business Partner". The BAdI definition determines when the logic runs (before save, after modification, etc.).

### Step 4 - Create a new implementation

Click "New" on the BAdI definition. Give the implementation a name, for example `YY1_VALIDATE_VAT_COUNTRY`.

### Step 5 - Write the logic

The browser-based editor opens. Write the validation logic:

```abap
" Check if VAT Country is filled
IF vat_country IS NOT INITIAL.
  " Validate that VAT number is also filled
  IF vat_number IS INITIAL.
    " Raise an error message
    RAISE EXCEPTION TYPE cx_raise_message
      EXPORTING
        message_text = 'VAT Number is required when VAT Country is specified'.
  ENDIF.
ENDIF.
```

The editor supports basic ABAP syntax. Complex operations are available through the XCO library.

### Step 6 - Publish

Click "Publish" to activate the logic. The BAdI implementation now runs every time a customer master record is saved.

### Step 7 - Trace and debug (if needed)

If the logic does not behave as expected, open the "Custom Logic Tracing" app. Enable tracing for the BAdI implementation, reproduce the scenario and review the trace log to see exactly what happened during execution.

---

## Section 6 - Walkthrough - UI Adaptation

Scenario: For the Accounts Payable team, hide the "Sales" tab on the Customer Master screen since they never use it.

### Step 1 - Open the target app

Open the Manage Customer Master app on the Fiori Launchpad.

### Step 2 - Enter Adapt UI mode

Click the user icon in the top right corner. Select "Adapt UI". The app switches to adaptation mode with a toolbar at the top.

### Step 3 - Select the scope

Choose who this adaptation applies to:

```
For Everyone    - all users see the change
For This Role   - only users with a specific role see the change
For Me Only     - only the current user sees the change (personal variant)
```

For this scenario, select "For This Role" and choose the Accounts Payable role.

### Step 4 - Make the changes

Click on the "Sales" tab. The adaptation toolbar shows options:
- Hide - removes the tab from the screen
- Rename - changes the label
- Move - repositions it

Select "Hide". The Sales tab disappears from the screen.

### Step 5 - Save and activate

Click "Save" in the adaptation toolbar. The change is now active for all users with the Accounts Payable role. Users with other roles still see the Sales tab.

### Step 6 - Manage adaptations

All UI adaptations can be viewed and managed in the "Manage Launchpad Settings" area. Adaptations can be deleted or modified later if requirements change.

---

## Section 7 - Checking if an App is Extensible

Not every SAP Fiori app supports key user extensibility. Here are all the ways to check.

### Method 1 - From inside the app (quickest)

Open the Fiori app and click user icon then "Adapt UI". If the "Add Field" option appears, the app supports custom fields. If Adapt UI is not available at all, the app does not support UI adaptation.

### Method 2 - Extensibility Cockpit (most comprehensive)

Open the Extensibility Cockpit app and search by scope item or business context. It shows all extensible objects with details on what types of extensions are supported and remaining capacity.

### Method 3 - SAP Fiori Apps Reference Library (before system access)

Go to the new SAP Fiori Apps Reference Library at `fal.cloud.sap`. Search for the app by name or App ID. Open the detail view and go to the Product Features tab. Scroll to the bottom and click "Read more in App Documentation" to open the SAP Help Portal.

In the SAP Help Portal navigation menu look for the "App Extensibility" section. If this section is missing, the app does not support custom fields. If present, it lists the supported extensibility types, business contexts, UI elements (for custom fields) and BAdI definitions (for custom logic).

### Method 4 - Extensibility Assistant (GenAI powered, Public Cloud only)

SAP has introduced a GenAI-driven assistant that helps key users figure out which business context, OData service or CDS view to extend. It is available on S/4HANA Cloud Public Edition only and needs to be activated through Joule for Development (J4D). After activation it takes one day before the "Open Assistant" button becomes visible.

---

## Section 8 - SAP GUI Transactions for Extensibility

These transactions are available on S/4HANA on-premise and private cloud only. Public cloud has no SAP GUI access.

| Transaction | Name | Purpose |
|-------------|------|---------|
| `SCFD_REGISTRY` | Custom Fields Registry UI | Shows all business contexts in the system and which objects (tables, CDS views, OData services, BAdIs) can be enhanced under each context. Read-only lookup tool. |
| `SCFD_EUI` | Custom Fields Enable UI | Enables existing database table fields for use in the Custom Fields and Logic Fiori app. More powerful than the Fiori app for advanced scenarios like legacy field enablement and F4 value help with custom check tables. |
| `SCFD_MD_UPLOAD` | Custom Fields Metadata Upload | Imports custom field definitions using an XML format file. |
| `SCFD_RECOVERY` | Custom Fields Recovery | Displays archive information for custom fields. |

### When to use which

```
I want to see which business contexts exist and what can be extended
    -> SCFD_REGISTRY (GUI) or Extensibility Cockpit (Fiori)

I want to create a simple custom field
    -> Custom Fields and Logic Fiori app

I want to enable an existing database field for custom field use
    -> SCFD_EUI (GUI) - more powerful for legacy fields and F4 help

I want to add custom validation logic
    -> Custom Fields and Logic Fiori app (Custom Logic tab)

I want to implement a traditional developer BAdI
    -> SE19 (classic) or ADT in Eclipse (ABAP Cloud)
```

Note: `SE18` (BAdI definition) and `SE19` (BAdI implementation) are for traditional developer BAdIs. They are a completely different mechanism from key user custom logic. Key user BAdI implementations are always done through the Fiori app, never through SE18/SE19.

---

## Section 9 - Transporting Extensions to Production

Key user extensions need to be transported from the development system to quality and production, just like developer objects. The mechanism differs between Public Cloud and Private Cloud.

### 9.1 S/4HANA Cloud Public Edition

Transport is handled through the "Export Software Collection" Fiori app.

```
Step 1:  Open "Export Software Collection" app
Step 2:  Create a new software collection
Step 3:  Add the custom fields, logic and business objects to the collection
Step 4:  Check for dependencies using the Extensibility Inventory app
Step 5:  Release the collection for export
Step 6:  The system automatically transports to the test tenant
Step 7:  After testing, release to production tenant
```

Important: If a developer extensibility object (like a CDS view) depends on a custom field created by a key user, both must be transported together or in the correct order. The system checks dependencies during export and blocks the transport if a required object is missing.

### 9.2 S/4HANA Cloud Private Edition and On-Premise

Transport uses the classical SAP transport system (CTS), the same as developer objects.

```
Step 1:  Open "Configure Software Packages" app
Step 2:  Register the extensions for transport
Step 3:  Add them to a transport request
Step 4:  Release the transport request
Step 5:  Import into quality system via TMS
Step 6:  After testing, import into production via TMS
```

### 9.3 Transport via abapGit (Partners and ISVs)

For SAP partners who ship functionality to customers, key user custom fields can also be transported using abapGit. This is useful when the partner already uses abapGit for developer extensibility artifacts (RAP business objects, classes, CDS views) and wants to include key user extensions in the same Git repository.

---

## Section 10 - Limitations and When to Switch to Developer Extensibility

Key user extensibility is powerful for simple scenarios but has clear boundaries.

### What Key User Extensibility Cannot Do

| Limitation | Alternative |
|-----------|------------|
| Complex business logic with multiple entity interactions | Developer Extensibility with RAP |
| Deep composition trees (parent-child-grandchild) | Developer Extensibility with RAP |
| Draft enabled applications with complex workflows | Developer Extensibility with RAP |
| Custom UI layouts beyond what Adapt UI supports | Fiori Elements or freestyle Fiori with Developer Extensibility |
| Full ABAP language features in custom logic | Developer Extensibility with ABAP classes |
| Access to non-released SAP objects | Tier 2 wrappers with Developer Extensibility |
| Integration with external systems beyond basic OData | Side by Side Extensibility on BTP |

### The Decision Rule

```
Can a key user do it with the browser tools?
    |
    YES -> use Key User Extensibility
    |
    NO  -> does it need tight coupling to S/4HANA data?
              |
              YES -> use Developer Extensibility (RAP on-stack)
              |
              NO  -> use Side by Side Extensibility (BTP)
```

### Combining Key User and Developer Extensibility

The two types are not mutually exclusive. A common pattern is:

1. Key user creates a custom field on a standard business object
2. Developer uses that custom field in a custom CDS view or RAP business object
3. Both are transported together (the custom field first, then the developer objects that depend on it)

The Extensibility Inventory app shows all dependencies between key user and developer objects, helping to plan the correct transport sequence.

---

## References

| Source | Topic |
|--------|-------|
| SAP Fiori Apps Reference Library (`fal.cloud.sap`) | Check which apps support extensibility |
| SAP Learning - Key User In-App Extensibility Tools | Step by step walkthroughs for custom fields and logic |
| SAP Learning - Using Key User In-App Extensibility Tools in S/4HANA Cloud | Extensibility Cockpit, business contexts, capacity |
| SAP Community - In-App Extension / Key User Extension (Sep 2024) | SCFD_REGISTRY, business context lookup, custom fields walkthrough |
| SAP Community - Adding Fields in Standard Fiori Apps with CFL (Jul 2025) | Business context tabs, target objects, data source extensions |
| SAP Community - Extensibility Simplified Guide for Beginners (Apr 2026) | Overview of all extensibility types |
| SAP Help Portal - Custom Fields and Logic | Official documentation for custom field creation and management |
| SAP Help Portal - Handling of Legacy Fields in S/4HANA | SCFD_EUI usage for existing database fields |
| SAP KBA 3431894 | How to find relevant business context for a database table |

---

<div align="center">

*Last updated: June 2026*

</div>
