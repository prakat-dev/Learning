<div align="center">

# RAP 01 - Data Modeling

**Tables, CDS Interface Views, Associations, Compositions and Mapping**

`CDS` `Interface Views` `Database Tables` `Draft Tables` `Compositions` `Associations` `Managed Numbering`

---

*Covers data modeling concepts in RAP. This is the foundation layer. Every other RAP concept builds on top of this.*

</div>

---

## Table of Contents

| # | Section | What it Covers |
|---|---------|---------------|
| 1 | [Where Data Comes From](#section-1---where-data-comes-from) | The different data source types available in RAP |
| 2 | [Database Tables](#section-2---database-tables) | Active tables, draft tables, naming, key design |
| 3 | [CDS Interface Views](#section-3---cds-interface-views) | What they are, root vs child, field aliasing |
| 4 | [Associations vs Compositions](#section-4---associations-vs-compositions) | Ownership vs reference, cardinality |
| 5 | [Splitting One Table into Multiple Entities](#section-5---splitting-one-table-into-multiple-entities) | The where clause pattern |
| 6 | [Annotations at the Interface Level](#section-6---annotations-at-the-interface-level) | Semantic annotations and what belongs here |
| 7 | [Field Mapping](#section-7---field-mapping) | Connecting CDS field names to DB column names |
| 8 | [Common Mistakes](#section-8---common-mistakes) | Things to watch out for |

---

## Section 1 - Where Data Comes From

The first decision in any RAP application is where the data will come from. RAP supports multiple data source types and the right choice depends on whether the app owns its data, reads from somewhere else or wraps an existing system.

### Database Table

The most common starting point for transactional apps. A custom database table is created and RAP manages all read and write operations against it. Used when the application owns its data and needs full CRUD (create, read, update, delete) support.

Example: An employee onboarding app creates a table `ZEMP_MASTER_TP3` to store employee records. A travel expense app creates `ZTRAVEL_EXPENSE` to store trip data. In both cases the app owns the data and RAP manages persistence.

### Custom Entity

No database table exists. The CDS entity is defined with the keyword `define custom entity` and a query implementation class provides the data at runtime. The data can come from anywhere: a BAPI call, an external API, a calculation, an RFC call or even hardcoded values. Custom entities are commonly used for read-only scenarios, wrapping legacy BAPIs or fetching data from external systems.

#### Defining the Custom Entity

The CDS definition looks similar to a regular view entity but uses the keyword `custom entity` and specifies the query implementation class via an annotation. Field types must be specified explicitly since there is no underlying table to infer them from.

```sql
@EndUserText.label: 'Flight Data from BAPI'
@ObjectModel.query.implementedBy: 'ABAP:ZCL_FLIGHT_QUERY'
define custom entity ZCE_FLIGHT_DATA
{
  key CarrierId   : abap.char(3);
  key ConnectionId: abap.numc(4);
  key FlightDate  : abap.dats;
      Price       : abap.dec(15,2);
      Currency    : abap.cuky;
      PlaneType   : abap.char(10);
      SeatsMax    : abap.int4;
      SeatsFree   : abap.int4;
}
```

The annotation `@ObjectModel.query.implementedBy` is the critical part. It tells RAP which ABAP class will provide the data when this entity is queried. The prefix `ABAP:` is required.

#### The Query Implementation Class

The class must implement the interface `if_rap_query_provider`. This interface has one method called `select` which RAP calls every time the entity is read (list page load, navigation, filtering, sorting, paging).

The method receives two objects. `io_request` describes what the consumer asked for. `io_response` is where the answer is placed.

```abap
CLASS zcl_flight_query DEFINITION PUBLIC FINAL
  CREATE PUBLIC.
  PUBLIC SECTION.
    INTERFACES if_rap_query_provider.
ENDCLASS.

CLASS zcl_flight_query IMPLEMENTATION.

  METHOD if_rap_query_provider~select.

    " 1. Read what the consumer asked for
    DATA(lt_filter)    = io_request->get_filter( )->get_as_ranges( ).
    DATA(lv_offset)    = io_request->get_paging( )->get_offset( ).
    DATA(lv_page_size) = io_request->get_paging( )->get_page_size( ).

    DATA(lt_carrier_range) = VALUE #( lt_filter[ name = 'CARRIERID' ]-range OPTIONAL ).

    " 2. Count first, only if requested
    IF io_request->is_total_numb_of_rec_requested( ).
      SELECT COUNT(*) FROM /dmo/flight
        WHERE carrier_id IN @lt_carrier_range
        INTO @DATA(lv_count).
      io_response->set_total_number_of_records( lv_count ).
    ENDIF.

    " 3. Fetch the page, only if data requested
    IF io_request->is_data_requested( ).
      IF lv_page_size = if_rap_query_paging=>page_size_unlimited.
        SELECT * FROM /dmo/flight
          WHERE carrier_id IN @lt_carrier_range
          ORDER BY carrier_id, connection_id
          OFFSET @lv_offset
          INTO TABLE @DATA(lt_flights).
      ELSE.
        SELECT * FROM /dmo/flight
          WHERE carrier_id IN @lt_carrier_range
          ORDER BY carrier_id, connection_id
          OFFSET @lv_offset
          UP TO @lv_page_size ROWS
          INTO TABLE @lt_flights.
      ENDIF.

      io_response->set_data( lt_flights ).
    ENDIF.

  ENDMETHOD.

ENDCLASS.
```

#### Every Method on io_request

The `io_request` object (interface `IF_RAP_QUERY_REQUEST`) is how RAP tells the query class what the consumer wants. The query class is responsible for honoring each one. Ignoring them leads to a broken UI (no paging, search bar does nothing, sorting ignored).

| Method | Returns | Purpose |
|--------|---------|---------|
| `get_entity_id( )` | Entity name | Which entity is being queried. Useful when one class serves multiple custom entities |
| `get_filter( )` | `IF_RAP_QUERY_FILTER` | Filter conditions from the search fields and URL `$filter` |
| `get_paging( )` | `IF_RAP_QUERY_PAGING` | Offset (skip) and page size (top) |
| `get_sort_elements( )` | Sort table | Which fields to sort by and the direction |
| `get_requested_elements( )` | Field name table | Which fields the consumer actually needs |
| `get_search_expression( )` | String | The free text search string typed in the search box |
| `get_parameters( )` | Parameter table | Values for a parameterized custom entity |
| `get_aggregation( )` | `IF_RAP_QUERY_AGGREGATION` | Aggregation and grouping info for analytical queries |
| `is_data_requested( )` | Boolean | Whether the consumer needs the actual data rows |
| `is_total_numb_of_rec_requested( )` | Boolean | Whether the consumer needs the total count |

#### Filtering

There are three ways to read filters. Choose based on the data source.

```abap
" Option A: as ranges (select-options style) - good for ABAP logic and BAPIs
DATA(lt_filter) = io_request->get_filter( )->get_as_ranges( ).
DATA(lt_carrier_range) = VALUE #( lt_filter[ name = 'CARRIERID' ]-range OPTIONAL ).

SELECT * FROM /dmo/flight
  WHERE carrier_id IN @lt_carrier_range
  INTO TABLE @DATA(lt_flights).
```

```abap
" Option B: as a ready SQL string - good for a dynamic WHERE on a DB table
DATA(lv_where) = io_request->get_filter( )->get_as_sql_string( ).
" e.g. returns: CARRIER_ID = 'LH' AND PRICE BETWEEN '100' AND '500'

SELECT * FROM /dmo/flight
  WHERE (lv_where)
  INTO TABLE @DATA(lt_flights2).
```

```abap
" Option C: as a tree - for complex parsing of nested AND/OR conditions
DATA(lo_tree) = io_request->get_filter( )->get_as_tree( ).
DATA(lo_root) = lo_tree->get_root_node( ).
" walk the tree with get_children( ), get_value( ) etc.
```

Use ranges for BAPIs and ABAP logic, the SQL string for direct database SELECTs, and the tree only when the filter logic is complex enough to need node-by-node parsing.

#### Paging

Paging has two values: offset (how many to skip) and page size (how many to return).

```abap
DATA(lv_offset)    = io_request->get_paging( )->get_offset( ).      " skip
DATA(lv_page_size) = io_request->get_paging( )->get_page_size( ).   " top
```

The critical gotcha: when no limit is requested `get_page_size( )` returns `-1`, the constant `if_rap_query_paging=>page_size_unlimited`. This must be handled or the SELECT fails.

```abap
IF lv_page_size = if_rap_query_paging=>page_size_unlimited.
  SELECT * FROM /dmo/flight
    ORDER BY carrier_id, connection_id
    OFFSET @lv_offset
    INTO TABLE @DATA(lt_flights).
ELSE.
  SELECT * FROM /dmo/flight
    ORDER BY carrier_id, connection_id
    OFFSET @lv_offset
    UP TO @lv_page_size ROWS
    INTO TABLE @lt_flights.
ENDIF.
```

If the data comes from a BAPI that returns everything at once, apply paging manually on the internal table:

```abap
DATA(lv_end) = lv_offset + lv_page_size.
LOOP AT lt_all INTO DATA(ls_row) FROM lv_offset + 1 TO lv_end.
  APPEND ls_row TO lt_page.
ENDLOOP.
```

#### Sorting

```abap
DATA(lt_sort) = io_request->get_sort_elements( ).

LOOP AT lt_sort INTO DATA(ls_sort).
  IF ls_sort-descending = abap_true.
    SORT lt_flights BY (ls_sort-element_name) DESCENDING.
  ELSE.
    SORT lt_flights BY (ls_sort-element_name) ASCENDING.
  ENDIF.
ENDLOOP.
```

#### Free Text Search

When the entity supports search, the search box value comes through `get_search_expression( )`. This is separate from filters. Filters target specific fields, search is a free text term across the entity.

```abap
DATA(lv_search) = io_request->get_search_expression( ).

IF lv_search IS NOT INITIAL.
  " apply the search term across relevant fields
  SELECT * FROM /dmo/flight
    WHERE carrier_id LIKE @( |%{ lv_search }%| )
       OR plane_type LIKE @( |%{ lv_search }%| )
    INTO TABLE @DATA(lt_flights).
ENDIF.
```

#### Requested Elements

For performance, the consumer often needs only a few fields. `get_requested_elements( )` tells the class which ones. There is no need to compute or fetch fields nobody asked for.

```abap
DATA(lt_requested) = io_request->get_requested_elements( ).
" lt_requested might contain only: CARRIERID, PRICE
" so expensive calculations for other fields can be skipped
```

#### Parameters

If the custom entity is defined `with parameters`, the parameter values are read with `get_parameters( )`.

```abap
DATA(lt_params) = io_request->get_parameters( ).
DATA(lv_param)  = VALUE #( lt_params[ name = 'P_DATE' ]-value OPTIONAL ).
```

#### Data and Count Requested

The consumer does not always want both data and count. Sometimes only the count (to show "247 items"), sometimes only the data. Check before doing expensive work. The count is the total matching the filter, not the number on the current page.

```abap
IF io_request->is_total_numb_of_rec_requested( ).
  " calculate full count matching the filter
  io_response->set_total_number_of_records( lv_count ).
ENDIF.

IF io_request->is_data_requested( ).
  " fetch only the current page of data
  io_response->set_data( lt_flights ).
ENDIF.
```

#### Important Points About Custom Entities

Custom entities do not support draft. There is no draft table and no draft lifecycle. If the user navigates away their work is lost. For scenarios that need draft, use a database table with managed or unmanaged RAP instead.

Custom entities are read-only by default. To make them transactional (create, update, delete) additional work is needed with an unmanaged behaviour definition. This is covered in the advanced scenarios documents.

The query class is called on every request. If the data source is slow (like an external API call) it impacts the UI responsiveness directly. Caching strategies or background data loading should be considered for slow data sources.

Paging must be handled correctly. If `set_total_number_of_records` is not called, the Fiori table shows no scroll indicator and the user cannot page through the data. Always check `is_total_numb_of_rec_requested` and set the count when requested.

### Existing SAP CDS Views

No custom table is created. The interface CDS view is built on top of a released SAP CDS view like `I_BusinessPartner`, `I_Product` or `I_SalesOrder`. The data already exists in SAP standard tables. This approach is used when building apps that reshape or extend existing SAP data without creating a separate copy.

### External Entity via Service Consumption

The data lives in another system entirely. A service consumption model is created in Eclipse which generates proxy classes from the remote API metadata. This is the pattern used in side by side extensibility on BTP when the app needs to fetch data from an S/4HANA system.

### A Note on Abstract Entities

Abstract entities are not a data source. They cannot be queried and cannot serve as the root of a business object. They are purely type definitions used for action parameters (input and output structures for RAP actions), function import parameters and service consumption model type generation. They are covered in more detail in the behaviour definition document where action parameters are discussed.

### When to Use What

| Scenario | Data Source |
|----------|------------|
| New transactional app that owns its data | Database table with managed RAP |
| Read-only data from a BAPI or legacy function module | Custom entity with query class |
| Wrapping an external API | Custom entity or service consumption model |
| Extending or reshaping existing SAP data | CDS view on released SAP views |
| Read and write to a legacy system | Database table with unmanaged save or unmanaged RAP |

---

## Section 2 - Database Tables

### Active Tables

The active table holds production data. This is the real data that all users see and work with. Every transactional RAP app that owns its data has at least one active table.

A travel booking app might have:

| Table | Purpose |
|-------|---------|
| `ZTRAVEL` | Travel header (traveler, dates, status) |
| `ZBOOKING` | Individual bookings under a travel (flights, hotels) |
| `ZBOOKING_SUPPL` | Supplements for each booking (meals, baggage) |

An employee onboarding app might have:

| Table | Purpose |
|-------|---------|
| `ZEMP_MASTER_TP3` | Employee master data |
| `ZEMP_ADDRESS_TP3` | Employee address data |
| `ZEMP_ATTACHMENT` | Employee photo or document attachments |

### Draft Tables

When the behaviour definition declares `with draft`, each entity in the composition tree needs a corresponding draft table. Draft tables hold work-in-progress data. When a user starts editing a record the changes go to the draft table first. Only when the user clicks Save/Activate does the data move from draft to active.

| Draft Table | Active Table | Purpose |
|-------------|-------------|---------|
| `ZEMP_MASTER_DRFT` | `ZEMP_MASTER_TP3` | Employee master drafts |
| `ZEMP_ADDR_CDRFT` | `ZEMP_ADDRESS_TP3` | Current address drafts |
| `ZEMP_ADDR_PDRFT` | `ZEMP_ADDRESS_TP3` | Permanent address drafts |

Every draft table includes the standard draft admin fields through the include structure `SYCH_BDL_DRAFT_ADMIN_INC`. This adds fields like `%_DRAFTUUID`, `%_DRAFTENTITYTYPE`, `%_DRAFTGROUPID` and others that RAP uses internally to manage the draft lifecycle. These fields should never be filled manually.

The draft table does not need to be created by hand. A quick fix in the behaviour definition generates it automatically from the CDS view entity. The generated table contains one field per view element plus the draft admin include. If the entity changes later the same quick fix regenerates the table to match.

### Naming Conventions

Database table field names in SAP use underscores and uppercase. CDS view field names use CamelCase (also called PascalCase). The mapping between the two is defined in the behaviour definition.

```
Database table:  FIRST_NAME
CDS view:        FirstName
```

The draft table is not named manually field by field. It is generated automatically using a quick fix in the behaviour definition. The draft database table contains exactly one field for each element of the CDS view entity plus the technical draft admin fields. Because it is generated from the CDS view entity, its field structure follows the entity definition rather than being hand-crafted. If the CDS view entity changes later, the same quick fix regenerates the draft table to match.

### Key Design

RAP supports different key strategies. The most common for managed scenarios is UUID with managed numbering.

**UUID with managed numbering** is the standard pattern. RAP generates a UUID (`SYSUUID_X16`) automatically when a record is created. A human readable ID can be generated separately in a determination.

```sql
key empid as Empid,    -- UUID, generated by RAP automatically
    id_no as IdNo,     -- human readable ID like "260001", generated by custom logic
```

This separation of technical key (UUID) and business key (readable ID) is recommended. The UUID keeps RAP operations efficient (draft handling, locking, associations) while the business key gives users a meaningful identifier.

**Semantic keys** are used in some scenarios where the business key itself is the primary key. For example a country code table where the ISO code is the key. This is less common in transactional apps because draft handling works more cleanly with UUIDs.

**Late numbering** is used when wrapping legacy systems where the key is generated during save (for example by a BAPI). The entity has no key during draft/modify phase and receives its key only at save time. This is covered in detail in the numbering patterns document.

### Standard Administrative Field Types

| ABAP Type | Used For | Example |
|-----------|----------|---------|
| `SYSUUID_X16` | UUID primary key (16 byte raw) | Empid, TravelUUID |
| `ABAP_BOOLEAN` | Boolean flag (single char X or space) | Active, Terminated |
| `ABP_CREATION_USER` | User who created the record | CreatedBy |
| `ABP_CREATION_TSTMPL` | Timestamp of creation | CreatedAt |
| `ABP_LASTCHANGE_USER` | User who last changed the record | LastChangedBy |
| `ABP_LASTCHANGE_TSTMPL` | Timestamp of last change (used for total ETag) | LastChangedAt |
| `ABP_LOCINST_LASTCHANGE_TSTMPL` | Local instance last change (used for ETag master) | LocalLastChangedAt |

The `ABP_*` types are standard RAP types. Using them ensures that RAP automatically manages these administrative fields without custom code. RAP fills `CreatedBy` and `CreatedAt` on create, and `LastChangedBy`, `LastChangedAt` and `LocalLastChangedAt` on every update.

---

## Section 3 - CDS Interface Views

### What They Are

The CDS interface view (prefixed `ZI_` by convention) sits between the database table and the projection view. It is the technical foundation of the RAP business object. It defines the data structure, the relationships between entities and the semantic meaning of fields.

The interface view is not meant for direct UI consumption. It is consumed by the projection view which adds UI-specific annotations, value helps and text elements. This separation allows the same interface view to serve multiple projection views for different use cases. For example one projection for an HR manager who sees all fields and another projection for an employee self-service portal that shows only personal details.

### Root Entity vs Child Entity

A RAP business object has one root entity and optionally one or more child entities forming a composition tree.

**Root entity** is declared with `define root view entity`. It is the entry point of the business object.

```sql
define root view entity ZI_TRAVEL
  as select from ztravel
```

**Child entity** is declared with `define view entity` (no `root` keyword) and has an association back to the parent:

```sql
define view entity ZI_BOOKING
  as select from zbooking
  association to parent ZI_TRAVEL as _travel
    on $projection.TravelUUID = _travel.TravelUUID
```

The `association to parent` tells RAP that this entity is a child. A composition tree can go multiple levels deep. In the SAP standard travel example the structure is:

```
ZI_TRAVEL (root)
    |-- ZI_BOOKING (child)
            |-- ZI_BOOKING_SUPPLEMENT (grandchild)
```

In the employee master app the structure is:

```
ZI_EMPLOYEE_MASTER (root)
    |-- ZI_EMP_ADDRESS_DATA (child - current address)
    |-- ZI_EMP_PERM_ADDRESS_DATA (child - permanent address)
```

### Field Aliasing

The interface view aliases database column names from underscored format to CamelCase:

```sql
first_name       as FirstName,
last_name        as LastName,
date_of_birth    as DateOfBirth,
booking_date     as BookingDate,
connection_id    as ConnectionId,
```

This aliasing is important because all subsequent layers (projection view, behaviour definition, behaviour implementation, UI annotations) use the CamelCase names. The database column names are never referenced directly again after the interface view.

### Exposing Compositions and Associations

Compositions and associations declared at the top of the CDS view must also appear in the field list at the bottom. If they are not listed they are invisible to projection views and behaviour implementations.

```sql
define root view entity ZI_EMPLOYEE_MASTER
  as select from zemp_master_tp3
  composition [0..1] of ZI_EMP_ADDRESS_DATA      as _caddress
  composition [0..1] of ZI_EMP_PERM_ADDRESS_DATA as _paddress
  association [1..1] to ZI_GENDER_VALUEHELP      as _gendervh
    on $projection.Gender = _gendervh.value
{
  key empid         as Empid,
      first_name    as FirstName,
      ...

      -- these MUST appear here or they are inaccessible
      _caddress,
      _paddress,
      _gendervh
}
```

---

## Section 4 - Associations vs Compositions

This is one of the most important concepts in RAP data modeling. Getting it wrong breaks draft handling, delete cascades and the entire business object lifecycle.

### Composition (Ownership)

A composition means the parent owns the child. The lifecycle of the child is completely tied to the parent:
- If the parent is deleted the children are deleted too
- If the parent enters draft mode the children enter draft mode too
- If the parent is activated the children are activated too
- Creating a child always requires specifying which parent it belongs to

```sql
-- Travel owns its bookings
composition [0..*] of ZI_BOOKING as _booking

-- Employee owns its addresses
composition [0..1] of ZI_EMP_ADDRESS_DATA as _caddress
```

Compositions are declared on the parent entity pointing down to children. The child entity declares the reverse relationship using `association to parent`.

### Association (Reference)

An association is a lookup. No ownership. No lifecycle dependency. The associated entity exists independently and is not affected by what happens to the referencing entity.

```sql
-- Travel references an agency but does not own it
association [1..1] to /DMO/I_Agency as _agency
    on $projection.AgencyID = _agency.AgencyID

-- Employee references a gender value help but does not own it
association [1..1] to ZI_GENDER_VALUEHELP as _gendervh
    on $projection.Gender = _gendervh.value
```

Deleting a travel record does not delete the agency. Deleting an employee does not delete the gender values. The association exists purely for data lookup and joins.

### Cardinality

The numbers in brackets define how many records the relationship can return.

| Cardinality | Meaning | Example |
|-------------|---------|---------|
| `[0..1]` | Zero or one | An employee has zero or one current address |
| `[1..1]` | Exactly one | A booking has exactly one carrier (for the join) |
| `[0..*]` | Zero or many | A travel has zero or many bookings |
| `[1..*]` | One or many | At least one record always exists |

Cardinality matters for compositions. Using `[1..1]` for a composition where the child might not exist yet causes errors. If the child record is created later (via an action for example) use `[0..1]` or `[0..*]`.

### Quick Rule

```
Does the parent own the child's lifecycle?
    YES -> composition
    NO  -> association

Will deleting the parent delete this related data?
    YES -> composition
    NO  -> association
```

---

## Section 5 - Splitting One Table into Multiple Entities

Sometimes one physical database table needs to serve multiple separate CDS entities. This is done using a `where` clause in the CDS view to filter records by a type or category field.

### The Pattern

In the employee master app both the current address and permanent address live in the same physical table `ZEMP_ADDRESS_TP3`. A field called `addr_type` distinguishes between them. Two separate CDS entities filter on this field:

```sql
-- Current address: only records where addr_type = 'C'
define view entity ZI_EMP_ADDRESS_DATA
  as select from zemp_address_tp3
  association to parent ZI_EMPLOYEE_MASTER as _employee
    on $projection.Empid = _employee.Empid
{
  key empid      as Empid,
  key addr_type  as AddrType,
      street     as Street,
      city1      as City1,
      ...
}
where addr_type = 'C'
```

```sql
-- Permanent address: only records where addr_type = 'P'
define view entity ZI_EMP_PERM_ADDRESS_DATA
  as select from zemp_address_tp3
  association to parent ZI_EMPLOYEE_MASTER as _employee
    on $projection.Empid = _employee.Empid
{
  key empid      as Empid,
  key addr_type  as AddrType,
      street     as Street,
      city1      as City1,
      ...
}
where addr_type = 'P'
```

### Why This Works

The `where` clause filters at the CDS level. When RAP reads the current address entity it only sees records where `addr_type = 'C'`. When it reads the permanent address entity it only sees records where `addr_type = 'P'`. Each entity gets its own behaviour, its own draft table and its own UI section despite sharing one physical table.

### Advantages

- One physical table to maintain instead of two identical tables
- The same database constraints and indexes apply to both types
- Each type gets independent behaviour, draft table and UI treatment
- Actions can target specific types (InitCurrentAddress creates `addr_type = 'C'`, InitPermanentAddress creates `addr_type = 'P'`)

### The Draft Table Requirement

Even though both entities share one active table, each entity must have its own draft table. RAP manages draft separately per entity. The employee master app uses `ZEMP_ADDR_CDRFT` for current address drafts and `ZEMP_ADDR_PDRFT` for permanent address drafts.

### Other Use Cases for This Pattern

- Shipping address vs billing address (split by address type)
- Primary bank account vs secondary bank account (split by account type)
- Home phone vs work phone as separate entities (split by phone type)
- Different document categories stored in one table (split by document type)

The pattern is useful whenever the data structure is identical but the business meaning differs based on a category field.

---

## Section 6 - Annotations at the Interface Level

The interface view uses semantic annotations to tell the RAP framework what each field represents. These are not UI annotations. They describe the data meaning and enable RAP to handle fields correctly.

### Common Semantic Annotations

| Annotation | Purpose | Example |
|-----------|---------|---------|
| `@Semantics.eMail.address: true` | Renders as a clickable email link in the UI | EmailId |
| `@Semantics.telephone.type: [#HOME]` | Renders as a phone number with call options | PhoneNumberHom |
| `@Semantics.telephone.type: [#CELL]` | Renders as a mobile number | PhoneNumberPer |
| `@Semantics.mimeType: true` | Identifies the MIME type field for a binary attachment | MimeType |
| `@Semantics.largeObject` | Marks binary data (image, PDF) and links to MIME type and filename | ImageData |
| `@Semantics.amount.currencyCode` | Links an amount field to its currency field | TotalPrice |
| `@Semantics.quantity.unitOfMeasure` | Links a quantity field to its unit field | FlightDistance |
| `@Semantics.user.createdBy: true` | RAP auto-fills with current user on create | CreatedBy |
| `@Semantics.systemDateTime.createdAt: true` | RAP auto-fills with timestamp on create | CreatedAt |
| `@Semantics.user.lastChangedBy: true` | RAP auto-fills on every update | LastChangedBy |
| `@Semantics.systemDateTime.lastChangedAt: true` | RAP auto-fills on every update, used as total ETag | LastChangedAt |
| `@Semantics.systemDateTime.localInstanceLastChangedAt: true` | RAP auto-fills, used as ETag master for optimistic concurrency | LocalLastChangedAt |
| `@Semantics.address.street: true` | Identifies the field as a street address component | Street |
| `@Semantics.address.city: true` | Identifies the field as a city | City |
| `@Semantics.address.country: true` | Identifies the field as a country | Country |
| `@Semantics.address.zipCode: true` | Identifies the field as a postal/zip code | PostalCode |

### Example: Large Object (File Upload)

Handling file uploads like employee photos requires three fields working together:

```sql
@Semantics.mimeType: true
mime_type as MimeType,                -- e.g. 'image/jpeg'

file_name as FileName,                -- e.g. 'photo.jpg'

@Semantics.largeObject: {
  mimeType: 'MimeType',
  fileName: 'FileName',
  contentDispositionPreference: #INLINE
}
image_data as ImageData,              -- the actual binary data
```

The `@Semantics.largeObject` annotation links the binary field to its MIME type and filename fields. RAP and the Fiori UI use this to render an upload/download control automatically.

### Example: Currency and Unit Fields

Amount and quantity fields must be linked to their currency or unit fields:

```sql
@Semantics.amount.currencyCode: 'CurrencyCode'
total_price as TotalPrice,

currency_code as CurrencyCode,

@Semantics.quantity.unitOfMeasure: 'DistanceUnit'
flight_distance as FlightDistance,

distance_unit as DistanceUnit,
```

Without these annotations the UI would show a raw number without currency symbol or unit label.

### What Does NOT Belong on the Interface View

UI annotations should never be placed on the interface view. These belong on the projection view or in a metadata extension:

```sql
-- DO NOT put these on ZI_ views
@UI.lineItem
@UI.fieldGroup
@UI.facet
@UI.selectionField
@UI.headerInfo
@Consumption.valueHelpDefinition
```

Keeping the interface view free of UI annotations makes it reusable across multiple projection views and keeps a clean separation between data semantics and UI concerns.

### Other Common Interface Level Annotations

```sql
@AccessControl.authorizationCheck: #NOT_REQUIRED
```
Controls whether CDS access control (DCL) is enforced. `#NOT_REQUIRED` means no access control object is needed. For production apps this would typically be `#CHECK` with a corresponding DCL object.

```sql
@Metadata.ignorePropagatedAnnotations: true
```
Prevents annotations from underlying views being inherited. This keeps the annotation scope clean and explicit.

---

## Section 7 - Field Mapping

The behaviour definition contains a mapping block that connects CDS field names to database column names. This is required whenever the CDS alias differs from the physical database column name.

### How Mapping Works

The left side is the CDS field name (CamelCase). The right side is the database column name (underscored). RAP uses this mapping when reading from and writing to the database.

```abap
mapping for zemp_master_tp3
{
  Empid        = empid;
  FirstName    = first_name;
  LastName     = last_name;
  DateOfBirth  = date_of_birth;
  EmailId      = email_id;
  Gender       = gender;
  ...
}
```

### Each Entity Needs Its Own Mapping

Every entity in the composition tree that has its own persistent table needs a mapping block. The root entity maps to the root table. Each child entity maps to its own table.

```abap
-- Root entity mapping
mapping for zemp_master_tp3
{
  Empid     = empid;
  FirstName = first_name;
  ...
}

-- Child entity mapping
mapping for zemp_address_tp3
{
  Empid      = empid;
  AddrType   = addr_type;
  Street     = street;
  City1      = city1;
  Country    = country;
  PostalCode = postal_code;
  ...
}
```

### When Mapping is Not Needed

If the CDS field name matches the database column name exactly (case-insensitive), no mapping entry is required for that field. For example if both the CDS view and the database use `EMPID`, mapping is optional for that field.

In practice most RAP applications define the mapping explicitly for all fields to be clear and maintainable, even when some fields would technically match without it.

---

## Section 8 - Common Mistakes

### Using Composition When Association is Appropriate

Composition ties the child lifecycle to the parent. If the related entity should exist independently, using composition creates unwanted coupling. For example a Country value help table should never be a composition child of an employee. It is an independent lookup and should be an association.

```sql
-- WRONG: country data gets deleted when employee is deleted
composition [1..1] of ZI_COUNTRY as _country

-- CORRECT: country is just a lookup, exists independently
association [1..1] to ZI_COUNTRY as _country
    on $projection.CountryCode = _country.CountryCode
```

### Missing Currency or Unit Links on Amount and Quantity Fields

If an amount field is not linked to its currency field via `@Semantics.amount.currencyCode`, the Fiori UI shows a raw number without context. Same applies to quantities without `@Semantics.quantity.unitOfMeasure`. This also causes issues with OData metadata and analytical queries.

```sql
-- WRONG: no link between price and currency
total_price   as TotalPrice,
currency_code as CurrencyCode,

-- CORRECT: price is explicitly linked to its currency
@Semantics.amount.currencyCode: 'CurrencyCode'
total_price   as TotalPrice,
currency_code as CurrencyCode,
```

### Exposing All Fields with SELECT *

CDS views should only expose the fields that are needed. Exposing everything leads to unnecessary data transfer, poor performance and a bloated OData service. Every extra field increases the payload size on every request.

```sql
-- AVOID: pulling everything from the table
define view entity ZI_ORDER as select from ztable { * }

-- BETTER: only expose what the business object needs
define view entity ZI_ORDER as select from ztable
{
  key order_id    as OrderId,
      customer_id as CustomerId,
      order_date  as OrderDate,
      status      as Status
}
```

### Deeply Nested CDS View Stacks

Building CDS views on top of CDS views on top of CDS views creates deep stacks that are hard to debug and slow to execute. SAP recommends keeping the stack shallow. Interface view directly on the table, projection view on the interface view. Avoid adding intermediate views unless there is a clear reuse reason.

### Forgetting to Expose Associations in the Field List

An association or composition declared at the top of the CDS view but not listed in the field list at the bottom is invisible to projection views and behaviour implementations. This is a common cause of "association not found" errors.

```sql
composition [0..*] of ZI_BOOKING as _booking

{
  key travel_uuid as TravelUUID,
  ...
  _booking    -- if this line is missing the composition cannot be used
}
```

### Incorrect Cardinality on Compositions

Using `[1..1]` for a composition where the child might not exist yet causes runtime errors. If the child record is created later (for example via an action or by the user manually adding it) use `[0..1]` or `[0..*]`.

### Missing association to parent on Child Entity

The child entity must declare `association to parent` pointing back to the root. Without this the RAP framework cannot establish the composition tree and draft handling, locking and delete cascading will not work.

```sql
-- WRONG: child has no link back to parent
define view entity ZI_BOOKING as select from zbooking
{
  key booking_uuid as BookingUUID,
      ...
}

-- CORRECT: child explicitly declares its parent
define view entity ZI_BOOKING as select from zbooking
  association to parent ZI_TRAVEL as _travel
    on $projection.TravelUUID = _travel.TravelUUID
{
  key booking_uuid as BookingUUID,
      travel_uuid  as TravelUUID,
      ...
      _travel
}
```

### Draft Table Out of Sync with Active Table

The draft table contains one field per CDS view element plus the draft admin include `SYCH_BDL_DRAFT_ADMIN_INC`. If the CDS view entity changes (a field is added or removed) the draft table must be regenerated to match. Use the quick fix in the behaviour definition to regenerate it. If the draft table is left out of sync, draft operations fail or the new field cannot be edited in draft mode.

### Forgetting @Metadata.allowExtensions: true

Without this annotation on the interface or projection view, metadata extensions cannot be created for that view. This blocks the ability to add UI annotations via a separate metadata extension file which is the recommended approach.

```sql
-- Always include this on interface and projection views
@Metadata.allowExtensions: true
define root view entity ZI_TRAVEL as select from ztravel
```

---

## References

| Source | Topic |
|--------|-------|
| SAP Help Portal - ABAP CDS Entities | Official CDS syntax and annotation reference |
| SAP Help Portal - RAP Business Object Structure | Root entity, child entity, composition tree |
| SAP Learning - Developing RAP Applications | Data modeling patterns and best practices |
| SAP Help Portal - Semantic Annotations | Complete list of @Semantics annotations |
| SAP Help Portal - Draft Handling in RAP | Draft table structure and lifecycle |
| SAP Community - RAP Data Modeling Best Practices | Naming conventions, key design patterns |

---

<div align="center">

*Last updated: June 2026*

</div>
