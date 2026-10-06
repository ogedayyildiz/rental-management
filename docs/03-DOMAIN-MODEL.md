# 03 - Domain Model

## Document Status

Status: Draft  
Version: 0.1  
Project: Rental Management and Equipment Tracking Platform

---

# 1. Purpose of This Document

This document defines the core business entities of the platform and the relationships between them.

It answers questions such as:

- What is a Machine?
- What is a Rental?
- How is a tracking device connected to a machine?
- How are users connected to rental companies?
- How are customers represented?
- How is maintenance recorded?
- How are financial records associated with machines?
- How is telemetry associated with the correct machine?
- Which information is authoritative and which information is derived?

This document defines the conceptual domain model.

It is not intended to define every database column or SQL implementation detail.

Detailed database schemas may evolve during implementation while preserving the domain concepts defined here.

---

# 2. Domain Modeling Principles

The following principles apply throughout the domain model.

## 2.1 Organization Ownership

The platform is multi-tenant.

Most business entities belong to exactly one:

`Organization`

Examples include:

- Machines.
- Customers.
- Rentals.
- Branches.
- Tracking devices.
- Maintenance records.
- Financial records.

Tenant-owned data must never exist without a clear organization owner.

---

## 2.2 Historical Information Must Be Preserved

The system should avoid overwriting information when historical relationships matter.

For example, this is not sufficient:

```text
Machine.device_id = 123
```

because the device may later be moved to another machine.

Instead:

```text
Machine
   │
   ▼
DeviceInstallation
   │
   ▼
TrackingDevice
```

records when the relationship started and ended.

The same principle applies where appropriate to:

- Device installations.
- Machine assignments to rentals.
- CAN decoder versions.
- Maintenance history.
- Financial history.

---

## 2.3 Different Types of State Must Remain Separate

A machine does not have only one meaningful status.

For example:

```text
Rental State:
AVAILABLE / RESERVED / RENTED

Service State:
OPERATIONAL / MAINTENANCE / OUT_OF_SERVICE

Connectivity State:
ONLINE / OFFLINE / UNKNOWN

Operational State:
ACTIVE / IDLE / CHARGING / UNKNOWN

Asset Lifecycle:
ACTIVE / SOLD / RETIRED
```

These states represent different concepts and must not be collapsed into one generic `machine_status` field.

---

## 2.4 Derived Information Should Not Replace Source Data

Values such as:

- ROI.
- Profit.
- Fleet utilization.
- Current availability.
- Current machine status.

may be calculated from more fundamental records.

Where performance requires it, calculated values may be cached or materialized.

The underlying source data must remain authoritative.

---

## 2.5 Historical Business Data Must Not Change Because Master Data Changed

For example, if a machine was rented for:

```text
15,000 TRY / month
```

and the standard rental price later becomes:

```text
18,000 TRY / month
```

the historical rental must remain:

```text
15,000 TRY / month
```

Rental transactions therefore store pricing snapshots rather than relying on the current price list.

---

# 3. High-Level Domain Map

The main relationships can be viewed conceptually as:

```text
                           PLATFORM

                              │
                              ▼

                        Organization
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼

     Branch                Customer               Users
                              │                     │
                              │                 Membership
                              │                     │
                              ▼                     ▼
                            Rental                Roles
                              │
                         RentalItem
                              │
                    RentalMachineAssignment
                              │
                              ▼
                           Machine
                              │
          ┌───────────────────┼───────────────────────┐
          │                   │                       │
          ▼                   ▼                       ▼

 DeviceInstallation       WorkOrder              FaultEvent
          │                                          
          ▼
   TrackingDevice
          │
          ▼
      Telemetry


Machine
   │
   ├── FinancialEntry
   ├── Documents
   ├── DecoderProfile Assignment
   └── Current State
```

---

# 4. Organization

`Organization` represents one customer company using the SaaS platform.

For example:

```text
BADAL Kiralama
```

or:

```text
Rental Company A
```

Each organization forms a tenant boundary.

Typical organization information includes:

```text
id

name

legal_name

tax_information

default_currency

default_language

timezone

country

status
```

An organization may contain:

```text
Branches

Users through Memberships

Customers

Machines

Tracking Devices

Rentals

Maintenance

Financial Records

Documents
```

All organization-owned information must be isolated from other organizations.

---

# 5. User

`User` represents a human identity in the platform.

A user is global rather than directly belonging to one organization.

Example:

```text
User
    Ogeday Yıldız
```

The same user may theoretically belong to more than one organization.

For this reason:

```text
User
```

must not directly contain:

```text
organization_id
```

Instead, organization membership is represented through:

`Membership`

---

# 6. Membership

`Membership` connects:

```text
User
   │
   ▼
Organization
```

Example:

```text
User: Ahmet Yılmaz

Organization: BADAL Kiralama

Membership:
    Active
    Role: Service Manager
```

Typical information includes:

```text
user

organization

membership_status

joined_at

roles
```

A membership may later also contain branch restrictions.

For example:

```text
User may access:

Ankara Branch
Istanbul Branch

but not:

Izmir Branch
```

Detailed access rules belong in `05-AUTH-AND-PERMISSIONS.md`.

---

# 7. Role and Permission

A `Role` groups permissions.

Example:

```text
Service Technician
```

may contain:

```text
machine.read

telemetry.read

maintenance.read

maintenance.update

fault.read
```

but not:

```text
finance.read

device.immobilize
```

Permissions represent the actual authorization capability.

Roles are convenient collections of permissions.

Conceptually:

```text
Membership
     │
     ▼
    Role
     │
     ▼
Permissions
```

The backend must ultimately authorize operations using permissions rather than simply checking role names.

---

# 8. Branch

`Branch` represents an operational location of an organization.

Examples:

```text
Ankara Branch

Istanbul Branch

Antalya Branch
```

A branch may contain or manage:

- Machines.
- Employees.
- Rentals.
- Service operations.

A machine may have a home branch.

The machine's physical GPS location must remain separate from its organizational branch assignment.

For example:

```text
Home Branch:
Ankara

Current GPS Location:
Antalya customer site
```

These are different concepts.

---

# 9. Customer

`Customer` represents the party renting equipment.

Customers belong to one organization.

A customer may be:

```text
COMPANY
```

or:

```text
INDIVIDUAL
```

Typical information includes:

```text
customer_number

type

legal_name

display_name

tax_number

tax_office

notes

status
```

A customer may have multiple:

```text
Contacts

Addresses

Rentals

Documents
```

---

# 10. Customer Contact

`CustomerContact` represents a person associated with a customer.

Example:

```text
Customer:
ABC Construction

Contact:
Mehmet Kaya
Site Manager
+90 ...
mehmet@...
```

A company may therefore have multiple contacts for:

- Purchasing.
- Accounting.
- Site management.
- Technical communication.
- Management.

---

# 11. Customer Address

A customer may have multiple addresses.

Examples include:

```text
Registered Address

Billing Address

Delivery Address

Worksite
```

Addresses should not be assumed to be identical.

Rental delivery locations may also be stored as snapshots so that historical rentals are not changed if the customer's master address changes later.

---

# 12. Machine

`Machine` represents one physical rentable asset.

Examples:

```text
Galen ES1012
Serial: ES1012-00125

JLG 600AJ
Serial: ...
```

A machine belongs to one organization.

Typical machine information includes:

```text
internal_fleet_number

serial_number

manufacturer

model

category

year

purchase_date

home_branch

asset_lifecycle_status
```

Technical attributes may include:

```text
working_height

platform_height

lifting_capacity

machine_weight

power_type

drive_type
```

Not every machine category will use the same technical attributes.

The model must therefore avoid assuming that every rentable machine is a scissor lift.

---

# 13. Machine Model

`MachineModel` represents reusable technical information shared by machines of the same model.

Example:

```text
MachineModel:

Manufacturer:
Galen

Model:
ES1012

Category:
Scissor Lift
```

Possible model-level information includes:

```text
working_height

platform_height

rated_capacity

dimensions

power_type

manufacturer documentation
```

Individual machine records contain information specific to the physical unit.

For example:

```text
Machine Model:
ES1012

Machine:
Serial ES1012-00057
Purchased 2026
Purchase price 9,500 EUR
```

Initially, organizations may create the machine models they require.

A shared platform-level equipment catalog may be introduced later if beneficial.

---

# 14. Machine Category

`MachineCategory` describes the general equipment type.

Examples include:

```text
Scissor Lift

Articulated Boom Lift

Telescopic Boom Lift

Spider Lift

Forklift

Telehandler

Other
```

The architecture must allow additional categories without requiring fundamental application changes.

---

# 15. Machine Current State

Frequently requested machine state should be available through a current-state representation.

Conceptually:

`MachineCurrentState`

may contain information such as:

```text
rental_state

service_state

connectivity_state

operational_state

latest_latitude

latest_longitude

latest_battery_soc

latest_operating_hours

last_seen_at

active_fault_count
```

This record is primarily a read model.

It is not the authoritative source of historical information.

For example:

```text
latest_battery_soc = 63
```

may be stored for fast dashboard access.

Historical battery values remain in telemetry storage.

---

# 16. Tracking Device

`TrackingDevice` represents a physical telematics unit.

The initial provider is Ruptela.

Typical information includes:

```text
provider

provider_device_id

imei

serial_number

device_model

hardware_version

firmware_version

status

last_seen_at
```

A tracking device belongs to an organization even when it is temporarily not installed on a machine.

Tracking devices and machines are separate entities.

---

# 17. Device Installation

`DeviceInstallation` records the physical installation of a tracking device on a machine.

Typical information includes:

```text
machine

tracking_device

installed_at

removed_at

installed_by

notes
```

Example:

```text
Device 123

installed on:

Machine ES1012-001

2026-01-01 → 2026-06-01
```

The device may later be installed on:

```text
Machine ES1212-014

2026-06-02 →
```

Historical telemetry must be associated with the machine on which the device was installed at the time the telemetry was recorded.

A tracking device must not have overlapping active installations on multiple machines.

---

# 18. Telemetry Sample

`TelemetrySample` represents normalized machine telemetry.

Typical values may include:

```text
recorded_at

received_at

machine

tracking_device

latitude

longitude

speed

battery_soc

operating_hours

machine_active
```

Additional signals may exist depending on the equipment.

Not every telemetry point needs to contain every signal.

Telemetry must distinguish between:

```text
recorded_at
```

and:

```text
received_at
```

because devices may buffer records while cellular connectivity is unavailable.

---

# 19. Raw CAN Data

For custom CAN integrations, raw CAN frames may be received from Ruptela before machine-specific decoding.

Conceptually:

```text
Tracking Device
      │
      ▼
Ruptela Protocol
      │
      ▼
Raw CAN Frame
      │
      ▼
Decoder Profile
      │
      ▼
Normalized Telemetry
```

Raw CAN storage may use a shorter retention period than normalized telemetry if storage volume becomes significant.

Raw CAN data is technical source data and should not be directly exposed to normal business modules.

---

# 20. Decoder Profile

`DecoderProfile` describes how proprietary machine CAN data is interpreted.

Examples:

```text
GALEN_ES_V1

GALEN_ES_V2

GALEN_RT_V1

JLG_600AJ_V1
```

A decoder profile should be versioned.

Conceptually, it defines rules such as:

```text
CAN ID:
0x200

Byte:
0

Signal:
batterySoc

Scale:
1

Unit:
%
```

Decoder profiles are technical platform configuration rather than normal rental-company business data.

---

# 21. Machine Decoder Assignment

The machine must be associated with the decoder profile appropriate for its CAN protocol.

The association should preserve history.

Conceptually:

```text
Machine
   │
   ▼
MachineDecoderAssignment
   │
   ▼
DecoderProfile
```

Typical information:

```text
machine

decoder_profile

valid_from

valid_to
```

This allows a machine's controller software or CAN protocol to change over time without changing the interpretation of older telemetry.

Telemetry should use the decoder profile valid when the CAN frame was recorded.

---

# 22. Fault Definition

`FaultDefinition` describes the meaning of a known machine fault.

Example:

```text
Machine Model:
ES1012

Fault Code:
17

Description:
Platform overload sensor disagreement

Severity:
HIGH
```

Fault definitions may later reference:

- Troubleshooting instructions.
- Service manuals.
- Recommended actions.
- AI knowledge sources.

---

# 23. Fault Event

`FaultEvent` represents an actual fault detected on a physical machine.

Typical information includes:

```text
machine

fault_code

source

severity

first_seen_at

last_seen_at

cleared_at

status

occurrence_count
```

A fault event is different from a fault definition.

Conceptually:

```text
FaultDefinition
"What does code 17 mean?"
```

versus:

```text
FaultEvent
"Machine ES1012-001 has code 17 active right now."
```

---

# 24. Device Command

`DeviceCommand` represents a requested remote action.

Examples include:

```text
IMMOBILIZE

ENABLE

REQUEST_STATUS
```

Typical information includes:

```text
organization

machine

tracking_device

command_type

requested_by

requested_at

sent_at

acknowledged_at

status

failure_reason

expires_at
```

Possible statuses include:

```text
REQUESTED

QUEUED

SENT

ACKNOWLEDGED

FAILED

EXPIRED
```

Remote machine state must not be changed simply because a command was requested.

The actual result must be based on acknowledgement or confirmed machine/device state.

Every sensitive device command must create an audit record.

---

# 25. Rental

`Rental` represents one commercial rental transaction with a customer.

A rental belongs to:

```text
Organization

Customer
```

and may contain multiple rental items.

This is important because one customer contract may include:

```text
2 × Scissor Lifts

1 × Boom Lift

Transport

Additional Equipment
```

A rental should therefore not represent only one machine.

Typical rental information includes:

```text
rental_number

customer

branch

start_date

end_date

currency

status

billing_information

delivery_information

collection_information

notes
```

Possible lifecycle states may include:

```text
DRAFT

QUOTED

RESERVED

ACTIVE

COMPLETED

CANCELLED
```

Detailed lifecycle rules belong in `07-RENTAL-BUSINESS-RULES.md`.

---

# 26. Rental Item

`RentalItem` represents one commercial line inside a rental.

For example:

```text
Rental R-2026-001

Item 1:
12 m Scissor Lift
1 month
15,000 TRY

Item 2:
Transport
5,000 TRY
```

A machine rental item may initially describe a requested:

```text
Machine Category

Machine Model

Specific Machine
```

This allows a company to accept a reservation before deciding exactly which physical unit will be delivered.

Rental items must store agreed pricing as a historical snapshot.

Changing the standard price later must not change historical rentals.

---

# 27. Rental Machine Assignment

A `RentalMachineAssignment` connects a physical machine to a rental item.

This relationship should be separate from the rental item itself because machines may be allocated or replaced.

Example:

```text
Rental Item:
12 m Scissor Lift

Initially assigned:
ES1012-004

Machine develops fault.

Replacement:
ES1012-009
```

The system should preserve both assignments.

Typical information includes:

```text
rental_item

machine

assigned_at

released_at

assignment_reason
```

This provides accurate rental and machine history.

---

# 28. Rental Charges

Charges that do not directly represent a physical machine should remain separately identifiable.

Examples include:

```text
Transport

Collection

Insurance

Damage Waiver

Cleaning

Fuel

Operator

Discount

Other
```

A `RentalCharge` may therefore be associated with the rental.

Detailed pricing rules will be defined later.

---

# 29. Machine Availability

Machine availability should not be stored as an arbitrary manually controlled boolean such as:

```text
available = true
```

Availability should be determined from relevant business conditions.

For example:

```text
Machine Asset Lifecycle
        +
Machine Service State
        +
Rental Assignments
        +
Reservations
        ↓
Current Availability
```

A machine may therefore be unavailable because it is:

```text
Reserved

Rented

Under Maintenance

Out of Service

Sold

Retired
```

A current availability value may be materialized for performance, but it remains derived from authoritative records.

---

# 30. Maintenance Plan

`MaintenancePlan` defines recurring maintenance requirements.

Examples:

```text
Every 500 operating hours
```

or:

```text
Every 12 months
```

A maintenance plan may contain both triggers.

Example:

```text
Service every:

500 operating hours

OR

12 months

whichever occurs first
```

A plan may be designed for:

```text
Machine Model

Machine Category

Specific Machine
```

Detailed maintenance rules belong in `08-MAINTENANCE.md`.

---

# 31. Machine Maintenance Plan

`MachineMaintenancePlan` assigns a maintenance plan to a physical machine.

Conceptually:

```text
Machine
   │
   ▼
MachineMaintenancePlan
   │
   ▼
MaintenancePlan
```

This allows different machines to use different service schedules even when they belong to the same category.

---

# 32. Work Order

`WorkOrder` represents maintenance or repair work performed or planned on a machine.

A work order may be:

```text
PREVENTIVE

CORRECTIVE

INSPECTION

OTHER
```

Typical information includes:

```text
machine

type

status

created_at

scheduled_at

started_at

completed_at

assigned_technician

operating_hours

description

diagnosis

work_performed

cost
```

Completed work orders form the maintenance history.

A separate duplicate `MaintenanceHistory` entity is therefore not required for normal cases.

Historical maintenance imported from another system may be represented as completed work orders.

---

# 33. Maintenance and Machine Availability

Maintenance may affect machine availability.

For example:

```text
OPEN Work Order
        │
        ▼
Machine Service State
        │
        ▼
MAINTENANCE
```

However, not every work order necessarily requires taking the machine out of service.

The maintenance domain must explicitly determine whether the work prevents rental or operation.

---

# 34. Financial Entry

`FinancialEntry` represents a financial amount relevant to operational fleet analysis.

The purpose of this entity is management accounting and machine profitability analysis.

It is not intended initially to replace full accounting software.

Examples include:

```text
Rental Revenue

Machine Purchase Cost

Transport Cost

Maintenance Cost

Repair Cost

Financing Cost

Insurance Cost

Other Operating Cost
```

Typical information includes:

```text
organization

type

category

amount

currency

recognized_at

machine

rental

customer

source

notes
```

Relationships may be optional depending on the entry.

For example:

```text
Machine Purchase Cost
    → Machine

Rental Revenue
    → Rental + Machine

Corporate Expense
    → Organization
```

---

# 35. Financial Source Records

Where a financial entry is generated from another domain entity, the relationship should be preserved.

Example:

```text
WorkOrder
    │
    ▼
Maintenance Cost
    │
    ▼
FinancialEntry
```

or:

```text
Rental
    │
    ▼
Rental Revenue
    │
    ▼
FinancialEntry
```

This allows financial reports to explain where amounts came from.

Financial totals should not be manually duplicated across many tables.

---

# 36. Machine Acquisition

Machine purchase information is important enough to model explicitly.

A `MachineAcquisition` may contain:

```text
machine

supplier

purchase_date

purchase_price

currency

transport_cost

customs_cost

other_acquisition_cost
```

These records provide the basis for:

- Machine investment cost.
- ROI.
- Payback-period calculations.

Relevant amounts may also generate financial entries.

---

# 37. Document

`Document` represents a stored file known to the platform.

Examples include:

```text
Machine Manual

Service Manual

Certificate

Rental Contract

Customer Document

Maintenance Report

Invoice

Photo
```

Typical metadata includes:

```text
organization

file_name

mime_type

storage_key

uploaded_by

uploaded_at

document_type
```

Documents may be associated with entities such as:

```text
Machine

Customer

Rental

WorkOrder

Organization
```

The physical file remains in object storage.

---

# 38. Notification

`Notification` represents information delivered to a user because something happened in the system.

Examples:

```text
Critical machine fault

Maintenance due

Rental ending soon

Machine offline

Remote command failed
```

Typical information includes:

```text
organization

recipient

notification_type

created_at

read_at

related_entity

severity
```

External delivery through:

```text
Push

Email

SMS
```

should remain separate from the underlying notification event.

---

# 39. Audit Event

`AuditEvent` records sensitive user or system actions.

Examples:

```text
USER_PERMISSION_CHANGED

MACHINE_CREATED

FINANCIAL_ENTRY_CHANGED

DEVICE_INSTALLED

DEVICE_REMOVED

IMMOBILIZATION_REQUESTED

IMMOBILIZATION_ACKNOWLEDGED

RENTAL_CANCELLED
```

Typical audit information includes:

```text
organization

actor_user

action

entity_type

entity_id

timestamp

metadata
```

Audit history should not behave like ordinary editable business data.

---

# 40. Core Relationship Summary

The main domain relationships are:

```text
User
 │
 └── Membership
       │
       └── Organization
              │
              ├── Branch
              │
              ├── Customer
              │      │
              │      └── Rental
              │             │
              │             ├── RentalItem
              │             │      │
              │             │      └── RentalMachineAssignment
              │             │                    │
              │             │                    ▼
              │             │                 Machine
              │             │
              │             └── RentalCharge
              │
              ├── Machine
              │      │
              │      ├── MachineModel
              │      │
              │      ├── DeviceInstallation
              │      │       │
              │      │       └── TrackingDevice
              │      │
              │      ├── MachineDecoderAssignment
              │      │       │
              │      │       └── DecoderProfile
              │      │
              │      ├── Telemetry
              │      ├── FaultEvent
              │      ├── WorkOrder
              │      ├── FinancialEntry
              │      ├── Document
              │      └── MachineCurrentState
              │
              ├── MaintenancePlan
              │
              ├── FinancialEntry
              │
              ├── Notification
              └── AuditEvent
```

---

# 41. Important Domain Rules

The following rules are considered fundamental.

## 41.1 Tenant Ownership

Every tenant-owned business record must resolve to exactly one organization.

---

## 41.2 Device Installation History

A tracking device cannot be considered permanently attached to a machine.

Installation history must be preserved.

A device must not have overlapping active installations on different machines.

---

## 41.3 Telemetry Machine Resolution

Telemetry should be associated with the machine based on the device installation valid at:

```text
recorded_at
```

rather than:

```text
received_at
```

because telemetry may arrive late.

---

## 41.4 Decoder Version History

Raw CAN information must be decoded using the decoder profile valid when the data was generated.

Historical decoder assignments must therefore be preserved.

---

## 41.5 Rental History

Changing the current customer, price, machine assignment, or standard rate must not modify historical rental records.

---

## 41.6 Financial Currency

Every monetary value must have a clear currency.

Financial values must not use JavaScript floating-point numbers as the authoritative representation.

---

## 41.7 Machine Availability

Machine availability must be derived from business state rather than manually maintained as an independent truth.

---

## 41.8 Remote Commands

A requested remote command is not equivalent to a successfully executed remote command.

Device acknowledgement must be recorded separately.

---

## 41.9 Auditability

Sensitive actions must remain attributable to the user or system process responsible for them.

---

# 42. Deletion and Historical Data

Business entities with historical significance should generally not be physically deleted simply because they are no longer active.

Examples include:

```text
Machines

Customers with rental history

Rentals

Tracking device installations

Work orders

Financial entries
```

Instead, entities may use concepts such as:

```text
ACTIVE

INACTIVE

ARCHIVED

CANCELLED

RETIRED
```

depending on the domain.

Physical deletion may still be appropriate for records created accidentally when no dependent historical information exists.

Exact deletion rules will be defined per module.

---

# 43. Identifiers

Internal entities should use stable system-generated identifiers.

Business-facing entities may additionally have human-readable numbers.

Example:

```text
Internal ID:
01J...

Rental Number:
R-2026-00124

Machine Fleet Number:
AWP-0042

Customer Number:
CUS-00125
```

Business numbers must not be relied upon as database primary keys.

---

# 44. Time

Historical entities should preserve relevant timestamps.

Examples include:

```text
created_at

updated_at

installed_at

removed_at

assigned_at

completed_at

recorded_at

received_at
```

System timestamps should be stored consistently.

Business dates should be interpreted using the organization's relevant timezone.

---

# 45. Current State vs History

The platform should intentionally separate:

```text
Current State
```

from:

```text
Historical Events
```

For example:

```text
MachineCurrentState

batterySoc = 74
```

answers:

> What is the battery level now?

while:

```text
TelemetrySample
```

answers:

> What has the battery level been over the last three months?

Similarly:

```text
MachineCurrentState.rentalState
```

answers:

> Is the machine currently rented?

while:

```text
RentalMachineAssignment
```

answers:

> Who rented this machine in February?

This distinction should remain consistent throughout the application.

---

# 46. Initial Core Domain

The first release should prioritize the following domain entities:

```text
Organization

User

Membership

Role

Permission

Branch

Customer

CustomerContact

CustomerAddress

Machine

MachineModel

MachineCategory

MachineCurrentState

TrackingDevice

DeviceInstallation

DecoderProfile

MachineDecoderAssignment

TelemetrySample

FaultEvent

DeviceCommand

Rental

RentalItem

RentalMachineAssignment

RentalCharge

MaintenancePlan

MachineMaintenancePlan

WorkOrder

FinancialEntry

MachineAcquisition

Document

Notification

AuditEvent
```

Additional entities should be introduced only when required by defined product behavior.

---

# 47. Important Design Decisions Made in This Model

This domain model currently assumes:

### A rental can contain multiple machines.

A rental is therefore an order/transaction rather than a direct connection between one customer and one machine.

### A rental item and a physical machine assignment are separate.

This allows machines to be selected later or replaced during a rental.

### A machine and a tracking device are separate.

Device installation history is preserved.

### CAN decoder assignments are versioned.

Changing a CAN protocol does not change the interpretation of historical telemetry.

### Current machine state is separate from telemetry history.

Dashboards do not need to repeatedly scan the entire telemetry history.

### Maintenance history is represented by completed work orders.

Separate duplicate maintenance-history records are unnecessary.

### Financial entries provide operational financial analysis.

The platform is not initially intended to become a full accounting system.

### Machine availability is derived.

It is not an independently controlled boolean.

---

# 48. Open Domain Decisions

The following questions should be resolved before the domain model is considered final.

## 48.1 Rental Allocation

Should a reservation always allocate a specific machine immediately, or should the system allow:

```text
"1 × 12 m scissor lift"
```

to be reserved first and the exact machine assigned later?

The current model supports both.

---

## 48.2 Machine Replacement During Rental

The current model assumes that a rented machine may be replaced without creating a completely new rental.

The assignment history preserves both machines.

This should be confirmed as desired business behavior.

---

## 48.3 Multiple Tracking Devices

The current model allows the architecture to support more than one device on a machine if required later.

The initial product may enforce one primary tracking device per machine.

---

## 48.4 Customer Credit and Payment Tracking

The current domain includes operational financial entries but does not yet define:

```text
Invoices

Payments

Receivables

Credit Limits

Outstanding Balances
```

These should be added only if the platform is expected to perform accounting/receivables functions rather than integrating with an external accounting system.

---

## 48.5 Inventory and Accessories

The current model focuses primarily on serialized machines.

If the platform later manages:

```text
Safety harnesses

Chargers

Cables

Attachments

Consumables

Spare parts
```

a more general inventory/asset model may be required.

---

# 49. Related Documentation

System architecture:

`02-SYSTEM-ARCHITECTURE.md`

User interface:

`04-UI-UX.md`

Authentication and permissions:

`05-AUTH-AND-PERMISSIONS.md`

Telemetry architecture:

`06-TELEMETRY-IOT.md`

Rental business rules:

`07-RENTAL-BUSINESS-RULES.md`

Maintenance:

`08-MAINTENANCE.md`

Financial calculations:

`09-FINANCE-ANALYTICS.md`

Security:

`10-SECURITY.md`