HALAND PETCARE v2.1

SINGLE SOURCE OF TRUTH (SSOT)

Version: 2.1

Status: Production Architecture

Architecture: Local-First Clinic Operating System

Last Updated: 2026-06-23

---

1. PURPOSE

Haland PetCare adalah Clinic Operating System untuk klinik hewan yang dibangun menggunakan arsitektur Local-First.

Tujuan utama sistem:

- Seluruh operasional klinik tetap berjalan tanpa internet.
- Cloud bukan sumber data utama.
- Database lokal adalah sumber data utama.
- Seluruh fitur wajib mendukung mode offline.
- Sinkronisasi dilakukan ketika koneksi tersedia.

---

2. NON-NEGOTIABLE RULES

Seluruh pengembangan wajib mematuhi aturan berikut.

RULE 01

Local Database Is Source Of Truth.

Tidak boleh ada fitur yang langsung bergantung pada cloud.

Semua operasi:

- Create
- Read
- Update
- Delete

harus dilakukan ke database lokal terlebih dahulu.

---

RULE 02

Cloud Is Replication Layer.

Cloud hanya digunakan untuk:

- Backup
- Synchronization
- Monitoring
- Remote Access

Cloud tidak boleh menjadi dependency operasional.

---

RULE 03

Offline Must Always Work.

Jika internet mati:

- Login tetap berjalan
- POS tetap berjalan
- Queue tetap berjalan
- Medical Record tetap berjalan
- Inventory tetap berjalan
- Hospitalization tetap berjalan

---

RULE 04

Every Important Action Must Be Audited.

Tidak boleh ada perubahan data penting tanpa audit log.

---

RULE 05

No Hard Delete.

Semua data operasional menggunakan soft delete.

---

RULE 06

Security First.

Seluruh endpoint:

- Authentication Protected
- Role Protected
- Audit Logged

---

3. SYSTEM ARCHITECTURE

Next.js PWA
      │
      ▼
Application Core
      │
      ▼
PGlite Local Database
      │
 ┌────┼────┐
 │    │    │
 ▼    ▼    ▼
Audit Backup Sync
 │    │    │
 └────┼────┘
      ▼
Replication Engine
      │
      ▼
Supabase PostgreSQL

---

4. PRIMARY TECHNOLOGY STACK

Frontend:

- Next.js 15
- React
- TypeScript
- Tailwind CSS
- shadcn/ui

Offline Layer:

- PWA
- Service Worker
- IndexedDB
- Background Sync

Local Database:

- PGlite

Cloud Database:

- Supabase PostgreSQL

ORM:

- Drizzle ORM

Validation:

- Zod

State Management:

- Zustand

Authentication:

- Local Authentication
- Supabase Replication

Deployment:

- Vercel

---

5. USER ROLES

OWNER

Full Access

Can:

- Create Owner
- Create Doctor
- Create Staff
- Create Customer
- Disable User
- Reset Password
- Access Reports
- Access Settings

---

STAFF

Can:

- Create Customer
- Create Pet
- Manage Queue
- Manage Appointment
- POS
- Inventory

Cannot:

- Create Staff
- Create Doctor
- Create Owner

---

DOCTOR

Can:

- View Queue
- Create Medical Record
- Create Prescription
- Manage Hospitalization
- Discharge Patient

Cannot:

- Create User
- Modify Inventory

---

CUSTOMER

Can:

- View Own Pets
- View Medical Records
- View Appointments
- Create Appointments

Cannot:

- Access Internal Data

---

6. USER CREATION POLICY

Owner:

Create:

- Owner
- Doctor
- Staff
- Customer

Staff:

Create:

- Customer

Doctor:

No User Creation

Customer:

No User Creation

---

7. AUTHENTICATION ARCHITECTURE

Offline authentication is mandatory.

Tables:

local_users

local_sessions

Authentication flow:

Login
↓
Local User Lookup
↓
Password Verification
↓
Local Session Creation
↓
Access Granted

Internet is not required.

---

8. DATABASE PRINCIPLE

Primary Database:

haland_local

Engine:

PGlite

Source Of Truth:

YES

Cloud Database:

Supabase PostgreSQL

Source Of Truth:

NO

Purpose:

Replication Only

---

9. MODULES

Mandatory Modules:

- Authentication
- User Management
- Customers
- Pets
- Appointments
- Queue
- Medical Records
- Prescriptions
- Hospitalization
- Inventory
- POS
- Reports
- Audit Logs
- Sync Engine
- Backup Engine
- Notifications

---

10. WALK-IN WORKFLOW

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
WAITING
↓
Doctor Examination
↓
Medical Record
↓
Decision

OUTPATIENT
or
HOSPITALIZATION

---

11. APPOINTMENT WORKFLOW

Appointment
↓
PENDING
↓
Staff Confirmation
↓
CONFIRMED
↓
Arrival
↓
Convert To Queue
↓
Walk-In Flow

---

12. HOSPITALIZATION WORKFLOW

Medical Record
↓
Select Cage
↓
Hospitalization
↓
Daily Monitoring
↓
Doctor Approval
↓
Discharge
↓
Release Cage
↓
POS
↓
Completed

---

13. INVENTORY RULES

Stock Reduction Only After Payment.

Must Use Database Transaction.

Must Prevent:

- Double Deduction
- Race Condition
- Negative Stock

Medicine Must Support:

- Batch Number
- Expired Date

---

14. CAGE MANAGEMENT

Tables:

cages

Fields:

- id
- code
- size
- status
- notes

Status:

- AVAILABLE
- OCCUPIED
- MAINTENANCE

---

15. AUDIT SYSTEM

Every Important Action Must Be Logged.

audit_logs

Fields:

- id
- user_id
- action
- table_name
- record_id
- old_data
- new_data
- created_at

Actions:

- INSERT
- UPDATE
- DELETE
- LOGIN
- LOGOUT
- PAYMENT
- DISCHARGE
- CREATE_USER
- RESET_PASSWORD

---

16. BACKUP ENGINE

Automatic Backup:

- Daily
- Weekly

Manual Backup:

- Supported

Restore:

- Supported

Backup Target:

- Local File
- Supabase Storage

---

17. SYNC ENGINE

Every Change:

INSERT
UPDATE
DELETE

↓

Stored Locally

↓

Added To Queue

↓

Background Sync

↓

Cloud Replication

---

18. SYNC QUEUE

sync_queue

Fields:

- id
- device_id
- clinic_id
- table_name
- record_id
- operation
- payload
- entity_version
- checksum
- synced
- retry_count
- last_attempt_at

---

19. CONFLICT RESOLUTION

Never:

Last Write Wins

Strategy:

Merge First

If Conflict:

Create Conflict Entry

conflict_queue

Fields:

- id
- table_name
- record_id
- local_version
- server_version
- status
- resolved_by
- resolved_at

---

20. DEVICE REGISTRY

devices

Fields:

- id
- name
- type
- registered_by
- last_sync
- status

Types:

- Desktop
- Laptop
- Tablet
- Mobile

---

21. SECURITY POLICY

Default:

DENY ALL

Requirements:

- RLS Enabled
- Role Protected
- Audit Logged

Every Table Must Have:

- created_at
- updated_at

Operational Tables Must Have:

- deleted_at

---

22. GLOBAL STATUS BAR

Always Visible

States:

- ONLINE
- OFFLINE
- SYNCING
- CONFLICT

Show:

- Last Sync Time
- Pending Sync Count

---

23. PERFORMANCE TARGETS

Dashboard:
< 2 seconds

Customer Search:
< 100 ms

Queue Load:
< 300 ms

Medical Record Save:
< 500 ms

POS Payment:
< 500 ms

Background Sync:
Non-blocking

---

24. REPOSITORY STRUCTURE

haland-petcare/

src/
├── app/
├── components/
├── modules/
├── services/
├── stores/
├── hooks/
├── database/
├── sync/
├── auth/
├── inventory/
├── queue/
├── appointments/
├── customers/
├── pets/
├── medical-records/
├── hospitalization/
├── reports/
├── users/
├── lib/
├── types/

public/

docs/
├── BLUEPRINT.md
├── database-schema.md
├── workflow.md
├── sync-engine.md
├── rls-policies.md
├── api-contract.md

---

25. DEFINITION OF DONE

Feature dianggap selesai hanya jika:

✓ Validation

✓ Loading State

✓ Empty State

✓ Error State

✓ Audit Logged

✓ Offline Supported

✓ Sync Supported

✓ Role Protected

✓ RLS Protected

✓ Responsive

✓ Tested Offline

✓ Tested Sync

✓ No Console Error

✓ No Data Loss

✓ Production Ready

---

26. ARCHITECTURAL DECISIONS

Tidak diperbolehkan:

- Direct Cloud CRUD
- Cloud-Only Authentication
- Hard Delete Operational Data
- Internet Dependency
- Bypass Audit Log
- Bypass Role System
- Bypass Sync Queue

Semua perubahan arsitektur wajib memperbarui dokumen ini terlebih dahulu sebelum implementasi.

Dokumen ini adalah sumber kebenaran mutlak (Single Source of Truth) untuk seluruh pengembangan Haland PetCare.
