HALAND PETCARE v2.0

COMPLETE SYSTEM BLUEPRINT

Version: 2.0

Status: Production Architecture

Architecture: Local-First Clinic Operating System


---

1. PROJECT OVERVIEW



Haland PetCare adalah Clinic Operating System untuk klinik hewan yang dirancang dengan arsitektur Full Offline Local-First.

Seluruh operasional klinik harus tetap berjalan walaupun internet tidak tersedia.

Internet hanya digunakan untuk:

Sinkronisasi data

Backup cloud

Multi-device replication

Remote access


Cloud bukan sumber data utama.

Database lokal adalah sumber data utama.


---

2. CORE PRINCIPLES



Principle 1

Local Database First

Semua operasi:

CREATE
READ
UPDATE
DELETE

harus dilakukan ke database lokal terlebih dahulu.


---

Principle 2

Cloud As Replication Layer

Cloud hanya digunakan untuk:

Backup

Sync

Monitoring


Cloud tidak boleh menjadi dependency operasional.


---

Principle 3

Offline Must Work

Semua fitur wajib berjalan tanpa internet.

Tidak ada fitur yang bergantung pada koneksi internet.


---

Principle 4

Role-Based Security

Semua akses berdasarkan role.

Owner
Doctor
Staff
Customer


---

Principle 5

Audit Everything

Semua perubahan penting harus tercatat.


---

3. SYSTEM ARCHITECTURE



┌──────────────────────┐
│ Next.js PWA Client   │
└──────────┬───────────┘
│
▼
┌──────────────────────┐
│ Application Layer    │
│ Services             │
│ Validation           │
│ Business Rules       │
└──────────┬───────────┘
│
▼
┌──────────────────────┐
│ PGlite Local DB      │
│ Primary Database     │
└──────────┬───────────┘
│
▼
┌──────────────────────┐
│ Sync Queue           │
│ Background Sync      │
└──────────┬───────────┘
│
▼
┌──────────────────────┐
│ Supabase PostgreSQL  │
│ Backup & Replication │
└──────────────────────┘


---

4. ROLE HIERARCHY



OWNER

├── STAFF
│    └── CUSTOMER
│
├── DOCTOR
│
└── CUSTOMER


---

5. USER CREATION RULES



OWNER

Can Create:

owner

doctor

staff

customer


Can Edit:

all users


Can Disable:

all users



---

STAFF

Can Create:

customer


Cannot Create:

owner

doctor

staff


Cannot Change Roles


---

DOCTOR

Cannot Create Users

Cannot Edit Users


---

CUSTOMER

Cannot Create Users

Cannot Edit Users


---

6. OWNER WORKFLOW



Login
↓
Owner Dashboard

Dashboard:

Revenue Today

Revenue This Month

Total Patients

Active Queue

Active Hospitalization

Inventory Alerts

Top Services

Top Medicines


↓

Reports

↓

User Management

↓

System Monitoring


---

7. STAFF WORKFLOW



Login
↓
Staff Dashboard

Dashboard:

Waiting Queue

Today's Appointments

Unpaid Transactions

Low Stock Alerts


↓

Customer Registration

↓

Pet Registration

↓

Appointment Management

↓

Queue Management

↓

POS

↓

Inventory


---

8. DOCTOR WORKFLOW



Login
↓
Doctor Dashboard

Dashboard:

Today's Patients

Active Queue

Active Hospitalization

Follow Ups


↓

Queue

↓

Medical Examination

↓

Medical Record

↓

Prescription

↓

Hospitalization Decision

↓

Discharge


---

9. CUSTOMER WORKFLOW



Login
↓
Customer Dashboard

Dashboard:

My Pets

Appointments

Medical History

Hospitalization Status


↓

View Pet

↓

View Medical Records

↓

Create Appointment


---

10. WALK-IN FLOW



Customer Arrives
↓
Search Customer
↓
Customer Exists?

YES
↓
Select Pet

NO
↓
Create Customer
↓
Create Pet

↓

Create Queue

↓

Status = WAITING

↓

Doctor Calls Patient

↓

Status = IN_PROGRESS

↓

Medical Examination

↓

Medical Record Created

↓

Decision

OUTPATIENT
or
HOSPITALIZATION


---

11. OUTPATIENT FLOW



Medical Record Saved
↓
Queue DONE
↓
Generate Transaction
↓
POS
↓
Payment
↓
Inventory Reduction
↓
Invoice
↓
Completed


---

12. HOSPITALIZATION FLOW



Medical Record
↓
Select Empty Cage
↓
Create Hospitalization
↓
Cage Occupied
↓
Daily Monitoring
↓
Doctor Discharge
↓
Release Cage
↓
Generate Transaction
↓
POS
↓
Completed


---

13. APPOINTMENT FLOW



Create Appointment
↓
Status PENDING
↓
Staff Confirmation
↓
Status CONFIRMED
↓
Patient Arrives
↓
Convert To Queue
↓
Walk-In Flow


---

14. OFFLINE ARCHITECTURE



All Modules Must Work Offline

Supported:

✓ Login

✓ Customers

✓ Pets

✓ Queue

✓ Appointment

✓ Medical Records

✓ Hospitalization

✓ Inventory

✓ POS

✓ Reports


---

15. LOCAL DATABASE



Engine:

PGlite

Database:

haland_local

Local Database Is Primary Source Of Truth


---

16. CLOUD DATABASE



Provider:

Supabase

Purpose:

Backup

Sync

Replication


Cloud Is Secondary


---

17. SYNC ENGINE



Every Change:

INSERT
UPDATE
DELETE

↓

Stored Locally

↓

Added To Sync Queue

↓

Background Sync

↓

Supabase


---

18. SYNC QUEUE STRUCTURE



sync_queue

id

table_name

record_id

operation

payload

created_at

synced

retry_count

last_attempt_at


---

19. CONFLICT MANAGEMENT



Strategy:

Server Wins + Conflict Log

Never:

Last Write Wins


---

sync_conflicts

id

table_name

record_id

local_data

server_data

resolved

resolved_by

resolved_at


---

20. AUDIT SYSTEM



audit_logs

id

user_id

action

table_name

record_id

old_data

new_data

created_at


---

Actions:

INSERT

UPDATE

DELETE

LOGIN

LOGOUT

PAYMENT

STOCK_ADJUSTMENT

DISCHARGE

CREATE_USER

DISABLE_USER

RESET_PASSWORD


---

21. SECURITY MODEL



Every Table:

RLS Enabled

Default:

DENY ALL

Access Granted By Policy


---

22. INVENTORY RULES



Stock Reduced Only When:

payment_status = paid

Using Database Transaction

Must Prevent Race Condition


---

23. SOFT DELETE



deleted_at

Required On:

users

customers

pets

inventory_items

No Hard Delete Operational Data


---

24. DASHBOARD METRICS



Owner

Revenue Today

Revenue Month

Total Patients

Active Hospitalization


Staff

Waiting Queue

Pending Payments


Doctor

Patients Today

Follow Ups


Customer

Pets

Appointments



---

25. PWA REQUIREMENTS



manifest.json

service-worker.ts

background-sync.ts

offline-page

offline-indicator

install-prompt


---

26. GLOBAL STATUS BAR



Must Always Show:

ONLINE

OFFLINE

SYNCING

CONFLICT

LAST SYNC TIME

PENDING SYNC COUNT


---

27. PERFORMANCE TARGETS



Dashboard

< 2 sec

Customer Search

< 100 ms

Queue Load

< 300 ms

Medical Record Save

< 500 ms

POS Payment

< 500 ms

Sync Operation

Background


---

28. REPOSITORY STRUCTURE



haland-petcare/

src/

app/

components/

modules/

hooks/

stores/

services/

database/

sync/

auth/

reports/

inventory/

queue/

medical-records/

hospitalization/

customers/

pets/

appointments/

users/

types/

lib/

supabase/

public/

docs/

database-schema.md

workflow.md

sync-engine.md

rls-policies.md

api-contract.md

BLUEPRINT.md

README.md


---

29. DEPLOYMENT



Frontend

Vercel

Cloud Database

Supabase

Storage

Supabase Storage

Backup

Daily

Retention

30 Days


---

30. DEFINITION OF DONE



A feature is complete only if:

✓ Validation Exists

✓ Loading State Exists

✓ Empty State Exists

✓ Error State Exists

✓ Audit Logged

✓ Offline Supported

✓ Sync Supported

✓ Responsive

✓ Role Protected

✓ RLS Protected

✓ No Console Errors

✓ No Data Loss

✓ Tested Offline

✓ Tested Sync

✓ Production Ready
