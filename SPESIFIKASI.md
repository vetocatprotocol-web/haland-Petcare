# SPESIFIKASI HALAND PETCARE v1.0
### Sumber Kebenaran Mutlak Pengembangan Full-Stack

---

## 1. IDENTITAS SISTEM

```
Nama Sistem  : Haland PetCare
Versi        : 1.0
Tipe         : Clinic Operating System
Environment  : Single Clinic (satu lokasi)
Arsitektur   : Offline-first + Cloud Sync
```

---

## 2. TECH STACK

```
Frontend     : Next.js 14 (App Router) + TypeScript
Styling      : TailwindCSS
State        : Zustand
Form         : react-hook-form + zod
Icons        : lucide-react
Utilities    : date-fns, clsx, tailwind-merge
Offline      : IndexedDB via idb
Backend      : Supabase (PostgreSQL + Auth + Storage)
Deployment   : Vercel
```

---

## 3. STRUKTUR FOLDER

```
haland-petcare/
├── .github/workflows/ci.yml
├── public/icons/
├── src/
│   ├── app/
│   │   ├── (auth)/login/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── customers/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   ├── queue/page.tsx
│   │   │   ├── appointments/page.tsx
│   │   │   ├── medical-records/
│   │   │   │   ├── new/page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   ├── hospitalization/page.tsx
│   │   │   ├── inventory/page.tsx
│   │   │   ├── pos/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [transaction_id]/page.tsx
│   │   │   ├── reports/page.tsx
│   │   │   └── users/page.tsx
│   │   └── api/sync/route.ts
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Modal.tsx
│   │   │   ├── Table.tsx
│   │   │   ├── Badge.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── Select.tsx
│   │   │   ├── Textarea.tsx
│   │   │   ├── DatePicker.tsx
│   │   │   ├── Skeleton.tsx
│   │   │   ├── EmptyState.tsx
│   │   │   ├── ConfirmDialog.tsx
│   │   │   └── Toast.tsx
│   │   ├── layout/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Navbar.tsx
│   │   │   └── SyncStatus.tsx
│   │   └── modules/
│   │       ├── customers/
│   │       ├── pets/
│   │       ├── queue/
│   │       ├── appointments/
│   │       ├── medical-records/
│   │       ├── hospitalization/
│   │       ├── inventory/
│   │       ├── pos/
│   │       └── reports/
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts
│   │   │   └── server.ts
│   │   ├── db/
│   │   │   ├── indexed-db.ts
│   │   │   └── sync.ts
│   │   └── utils/
│   │       ├── format.ts
│   │       ├── validators.ts
│   │       └── export.ts
│   ├── hooks/
│   │   ├── use-offline.ts
│   │   ├── use-sync.ts
│   │   └── use-queue.ts
│   ├── stores/
│   │   ├── auth-store.ts
│   │   ├── queue-store.ts
│   │   └── sync-store.ts
│   ├── types/index.ts
│   └── middleware.ts
├── supabase/
│   ├── migrations/
│   │   ├── 001_initial_schema.sql
│   │   ├── 002_rls_policies.sql
│   │   └── 003_seed_data.sql
│   └── seed.sql
├── .env.example
├── .env.local
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── README.md
```

---

## 4. DATABASE SCHEMA LENGKAP

### 4.1 Tabel users
```sql
id            uuid PRIMARY KEY references auth.users(id)
full_name     text NOT NULL
phone         text
role          text NOT NULL CHECK (role IN ('owner','doctor','staff','customer'))
is_active     boolean DEFAULT true
created_at    timestamptz DEFAULT now()
updated_at    timestamptz DEFAULT now()
```

### 4.2 Tabel customers
```sql
id            uuid PRIMARY KEY DEFAULT uuid_generate_v4()
user_id       uuid REFERENCES users(id) ON DELETE SET NULL
full_name     text NOT NULL
phone         text NOT NULL
email         text
address       text
created_at    timestamptz DEFAULT now()
updated_at    timestamptz DEFAULT now()
```

### 4.3 Tabel pets
```sql
id            uuid PRIMARY KEY DEFAULT uuid_generate_v4()
customer_id   uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE
name          text NOT NULL
species       text NOT NULL
breed         text
gender        text CHECK (gender IN ('male','female','unknown'))
birth_date    date
weight_kg     numeric(5,2)
color         text
notes         text
is_active     boolean DEFAULT true
created_at    timestamptz DEFAULT now()
updated_at    timestamptz DEFAULT now()
```

### 4.4 Tabel cages
```sql
id            uuid PRIMARY KEY DEFAULT uuid_generate_v4()
cage_number   text NOT NULL UNIQUE
cage_type     text NOT NULL
is_occupied   boolean DEFAULT false
notes         text
created_at    timestamptz DEFAULT now()
updated_at    timestamptz DEFAULT now()
```

### 4.5 Tabel appointments
```sql
id                uuid PRIMARY KEY DEFAULT uuid_generate_v4()
pet_id            uuid NOT NULL REFERENCES pets(id) ON DELETE CASCADE
customer_id       uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE
doctor_id         uuid REFERENCES users(id) ON DELETE SET NULL
appointment_date  date NOT NULL
appointment_time  time NOT NULL
visit_type        text NOT NULL CHECK (visit_type IN ('consultation','vaccination','grooming','checkup','other'))
status            text DEFAULT 'pending' CHECK (status IN ('pending','confirmed','cancelled','done'))
notes             text
created_at        timestamptz DEFAULT now()
updated_at        timestamptz DEFAULT now()
```

### 4.6 Tabel queue
```sql
id              uuid PRIMARY KEY DEFAULT uuid_generate_v4()
pet_id          uuid NOT NULL REFERENCES pets(id) ON DELETE CASCADE
customer_id     uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE
doctor_id       uuid REFERENCES users(id) ON DELETE SET NULL
appointment_id  uuid REFERENCES appointments(id) ON DELETE SET NULL
queue_number    int NOT NULL
visit_type      text NOT NULL
status          text DEFAULT 'waiting' CHECK (status IN ('waiting','in_progress','done','cancelled'))
notes           text
queue_date      date DEFAULT current_date
created_at      timestamptz DEFAULT now()
updated_at      timestamptz DEFAULT now()
```

### 4.7 Tabel inventory_items
```sql
id              uuid PRIMARY KEY DEFAULT uuid_generate_v4()
name            text NOT NULL
category        text NOT NULL CHECK (category IN ('medicine','vaccine','supply','food','other'))
unit            text NOT NULL
stock           numeric(10,2) DEFAULT 0
min_stock       numeric(10,2) DEFAULT 5
purchase_price  numeric(12,2) DEFAULT 0
selling_price   numeric(12,2) DEFAULT 0
description     text
is_active       boolean DEFAULT true
created_at      timestamptz DEFAULT now()
updated_at      timestamptz DEFAULT now()
```

### 4.8 Tabel inventory_transactions
```sql
id                  uuid PRIMARY KEY DEFAULT uuid_generate_v4()
inventory_item_id   uuid NOT NULL REFERENCES inventory_items(id) ON DELETE CASCADE
transaction_type    text NOT NULL CHECK (transaction_type IN ('in','out','adjustment'))
quantity            numeric(10,2) NOT NULL
reference_id        uuid
reference_type      text
notes               text
created_by          uuid REFERENCES users(id) ON DELETE SET NULL
created_at          timestamptz DEFAULT now()
```

### 4.9 Tabel medical_records
```sql
id              uuid PRIMARY KEY DEFAULT uuid_generate_v4()
pet_id          uuid NOT NULL REFERENCES pets(id) ON DELETE CASCADE
doctor_id       uuid NOT NULL REFERENCES users(id) ON DELETE SET NULL
queue_id        uuid REFERENCES queue(id) ON DELETE SET NULL
visit_date      date DEFAULT current_date
chief_complaint text
physical_exam   text
diagnosis       text
treatment_plan  text
notes           text
follow_up_date  date
created_at      timestamptz DEFAULT now()
updated_at      timestamptz DEFAULT now()
```

### 4.10 Tabel medical_record_items
```sql
id                  uuid PRIMARY KEY DEFAULT uuid_generate_v4()
medical_record_id   uuid NOT NULL REFERENCES medical_records(id) ON DELETE CASCADE
item_type           text NOT NULL CHECK (item_type IN ('medicine','action','service'))
inventory_item_id   uuid REFERENCES inventory_items(id) ON DELETE SET NULL
item_name           text NOT NULL
quantity            numeric(10,2) DEFAULT 1
unit                text
price               numeric(12,2) DEFAULT 0
notes               text
created_at          timestamptz DEFAULT now()
```

### 4.11 Tabel hospitalizations
```sql
id                  uuid PRIMARY KEY DEFAULT uuid_generate_v4()
pet_id              uuid NOT NULL REFERENCES pets(id) ON DELETE CASCADE
doctor_id           uuid NOT NULL REFERENCES users(id) ON DELETE SET NULL
cage_id             uuid NOT NULL REFERENCES cages(id) ON DELETE SET NULL
medical_record_id   uuid REFERENCES medical_records(id) ON DELETE SET NULL
admission_date      date DEFAULT current_date
discharge_date      date
initial_condition   text
status              text DEFAULT 'active' CHECK (status IN ('active','discharged'))
notes               text
created_at          timestamptz DEFAULT now()
updated_at          timestamptz DEFAULT now()
```

### 4.12 Tabel hospitalization_logs
```sql
id                    uuid PRIMARY KEY DEFAULT uuid_generate_v4()
hospitalization_id    uuid NOT NULL REFERENCES hospitalizations(id) ON DELETE CASCADE
doctor_id             uuid REFERENCES users(id) ON DELETE SET NULL
log_date              date DEFAULT current_date
condition_update      text NOT NULL
treatment_given       text
created_at            timestamptz DEFAULT now()
```

### 4.13 Tabel transactions
```sql
id                  uuid PRIMARY KEY DEFAULT uuid_generate_v4()
invoice_number      text NOT NULL UNIQUE
customer_id         uuid REFERENCES customers(id) ON DELETE SET NULL
pet_id              uuid REFERENCES pets(id) ON DELETE SET NULL
medical_record_id   uuid REFERENCES medical_records(id) ON DELETE SET NULL
cashier_id          uuid REFERENCES users(id) ON DELETE SET NULL
subtotal            numeric(12,2) DEFAULT 0
discount            numeric(12,2) DEFAULT 0
total               numeric(12,2) DEFAULT 0
payment_method      text CHECK (payment_method IN ('cash','transfer'))
payment_status      text DEFAULT 'unpaid' CHECK (payment_status IN ('unpaid','paid'))
paid_at             timestamptz
notes               text
created_at          timestamptz DEFAULT now()
updated_at          timestamptz DEFAULT now()
```

### 4.14 Tabel transaction_items
```sql
id                  uuid PRIMARY KEY DEFAULT uuid_generate_v4()
transaction_id      uuid NOT NULL REFERENCES transactions(id) ON DELETE CASCADE
item_type           text NOT NULL CHECK (item_type IN ('medicine','action','service','other'))
inventory_item_id   uuid REFERENCES inventory_items(id) ON DELETE SET NULL
item_name           text NOT NULL
quantity            numeric(10,2) DEFAULT 1
unit_price          numeric(12,2) DEFAULT 0
subtotal            numeric(12,2) DEFAULT 0
created_at          timestamptz DEFAULT now()
```

---

## 5. RELASI ANTAR TABEL

```
auth.users (Supabase)
    └── users (1:1)
            └── customers (1:1, optional)
                    └── pets (1:N)
                            ├── appointments (1:N)
                            ├── queue (1:N)
                            ├── medical_records (1:N)
                            │       └── medical_record_items (1:N)
                            ├── hospitalizations (1:N)
                            │       └── hospitalization_logs (1:N)
                            └── transactions (1:N)
                                    └── transaction_items (1:N)

inventory_items
    ├── inventory_transactions (1:N)
    ├── medical_record_items (1:N)
    └── transaction_items (1:N)

cages
    └── hospitalizations (1:N)
```

---

## 6. ROLE & AKSES

```
OWNER
├── Bisa akses  : /dashboard, /reports, /users
├── Tidak bisa  : operasional harian
└── Supabase    : SELECT semua tabel, MANAGE users

DOCTOR
├── Bisa akses  : /queue, /medical-records, /hospitalization
├── Tidak bisa  : /pos, /inventory, /reports, /users
└── Supabase    : SELECT queue, CRUD medical_records, hospitalizations

STAFF
├── Bisa akses  : /queue, /customers, /appointments, /pos, /inventory
├── Tidak bisa  : /medical-records (write), /reports, /users
└── Supabase    : CRUD customers, pets, queue, appointments, transactions, inventory

CUSTOMER
├── Bisa akses  : /my-pets, /appointments
├── Tidak bisa  : semua modul internal klinik
└── Supabase    : SELECT own data only, INSERT appointments
```

---

## 7. WORKFLOW LENGKAP

### 7.1 Walk-in Patient Flow

```
Customer datang
    ↓
Staff cari / buat customer
    ↓
Staff pilih / daftarkan hewan
    ↓
Staff buat queue (nomor otomatis, pilih dokter, jenis kunjungan)
    ↓
Status queue: WAITING
    ↓
Staff panggil → status: IN_PROGRESS
    ↓
Doctor buka halaman pemeriksaan
    ↓
Doctor input rekam medis + items (obat/tindakan)
    ↓
Doctor pilih: RAWAT JALAN atau RAWAT INAP
    ↓
[RAWAT JALAN]                    [RAWAT INAP]
    ↓                                ↓
Status queue: DONE           Pilih kandang kosong
    ↓                                ↓
Auto-create transaction      Buat hospitalization record
    ↓                                ↓
Redirect ke /pos/[id]        Cage is_occupied = true
    ↓                                ↓
Staff proses pembayaran      Doctor monitoring harian
    ↓                                ↓
Stok obat berkurang          Doctor putuskan pulang
    ↓                                ↓
Invoice tercetak             Cage is_occupied = false
                                     ↓
                             Auto-create transaction
                                     ↓
                             Redirect ke /pos/[id]
```

### 7.2 Appointment Flow
```
Customer / Staff buat appointment
    ↓
Status: PENDING
    ↓
Staff konfirmasi → status: CONFIRMED
    ↓
Hari H customer datang
    ↓
Staff klik "Jadikan Walk-in"
    ↓
Auto-create queue dari appointment
    ↓
Status appointment: DONE
    ↓
Lanjut ke Walk-in Flow
```

### 7.3 Inventory Flow
```
Staff tambah item inventory
    ↓
Stok masuk → insert inventory_transaction (type: in)
    ↓
Doctor resepkan obat di rekam medis
    ↓
Staff proses kasir
    ↓
Pembayaran dikonfirmasi
    ↓
Stok otomatis berkurang → insert inventory_transaction (type: out)
    ↓
Jika stok < min_stock → tampilkan alert di inventory
```

---

## 8. LOGIKA BISNIS KRITIS

### 8.1 Nomor Antrian
```
- Format: angka integer, reset setiap hari
- Generate: SELECT MAX(queue_number) FROM queue WHERE queue_date = today, lalu +1
- Jika belum ada antrian hari ini → mulai dari 1
```

### 8.2 Nomor Invoice
```
- Format: INV-YYYYMMDD-XXX (contoh: INV-20240615-001)
- XXX adalah counter harian, reset tiap hari
- Generate di server saat transaction dibuat
```

### 8.3 Kalkulasi POS
```
subtotal  = SUM(quantity × unit_price) semua items
discount  = input manual (nominal)
total     = subtotal - discount
kembalian = jumlah_bayar - total (hanya untuk cash)
```

### 8.4 Pengurangan Stok
```
- Stok hanya berkurang SETELAH pembayaran dikonfirmasi (payment_status = paid)
- Setiap item obat di transaction_items yang punya inventory_item_id:
  → INSERT inventory_transactions (type: out, qty: item.quantity)
  → UPDATE inventory_items SET stock = stock - qty
- Jika stok tidak cukup → tampilkan warning tapi TIDAK blokir transaksi
```

### 8.5 Rawat Inap
```
- Kandang hanya bisa dipilih jika is_occupied = false
- Saat pasien masuk: UPDATE cages SET is_occupied = true
- Saat pasien pulang: UPDATE cages SET is_occupied = false
- discharge_date diset ke tanggal dokter klik "Pulangkan"
```

---

## 9. RLS POLICIES

### Prinsip RLS
```
- Semua tabel wajib aktifkan RLS
- Default: tidak ada yang bisa akses tanpa policy
- Helper function get_my_role() digunakan di semua policy
- Owner bisa SELECT semua tabel
- Customer hanya bisa akses data miliknya sendiri
```

### Policy per Role
```
TABEL users
  - SELECT: own data (id = auth.uid()) OR role = owner
  - INSERT/UPDATE/DELETE: role = owner only

TABEL customers
  - ALL: role IN (owner, doctor, staff)
  - SELECT: user_id = auth.uid() (untuk customer login)

TABEL pets
  - ALL: role IN (owner, doctor, staff)
  - SELECT: customer_id → customers.user_id = auth.uid()

TABEL appointments
  - ALL: role IN (owner, doctor, staff)
  - SELECT + INSERT: customer via customers.user_id

TABEL queue
  - ALL: role IN (owner, doctor, staff)

TABEL medical_records
  - ALL: role IN (owner, doctor)
  - SELECT: role = staff
  - SELECT: customer via pets → customers.user_id

TABEL medical_record_items
  - ALL: role IN (owner, doctor)
  - SELECT: role = staff

TABEL inventory_items
  - ALL: role IN (owner, staff)
  - SELECT: role = doctor

TABEL inventory_transactions
  - ALL: role IN (owner, staff)
  - SELECT: role = doctor

TABEL cages
  - ALL: role IN (owner, doctor, staff)

TABEL hospitalizations
  - ALL: role IN (owner, doctor)
  - SELECT: role = staff

TABEL hospitalization_logs
  - ALL: role IN (owner, doctor)
  - SELECT: role = staff

TABEL transactions
  - ALL: role IN (owner, staff)
  - SELECT: role = doctor

TABEL transaction_items
  - ALL: role IN (owner, staff)
  - SELECT: role = doctor
```

---

## 10. TYPESCRIPT TYPES LENGKAP

```typescript
// ===========================
// ENUMS
// ===========================
type UserRole = 'owner' | 'doctor' | 'staff' | 'customer'
type Gender = 'male' | 'female' | 'unknown'
type VisitType = 'consultation' | 'vaccination' | 'grooming' | 'checkup' | 'other'
type AppointmentStatus = 'pending' | 'confirmed' | 'cancelled' | 'done'
type QueueStatus = 'waiting' | 'in_progress' | 'done' | 'cancelled'
type ItemType = 'medicine' | 'action' | 'service' | 'other'
type InventoryCategory = 'medicine' | 'vaccine' | 'supply' | 'food' | 'other'
type InventoryTransactionType = 'in' | 'out' | 'adjustment'
type HospitalizationStatus = 'active' | 'discharged'
type PaymentMethod = 'cash' | 'transfer'
type PaymentStatus = 'unpaid' | 'paid'

// ===========================
// CORE INTERFACES
// ===========================
interface User {
  id: string
  full_name: string
  phone?: string
  role: UserRole
  is_active: boolean
  created_at: string
  updated_at: string
}

interface Customer {
  id: string
  user_id?: string
  full_name: string
  phone: string
  email?: string
  address?: string
  created_at: string
  updated_at: string
  pets?: Pet[]
}

interface Pet {
  id: string
  customer_id: string
  name: string
  species: string
  breed?: string
  gender: Gender
  birth_date?: string
  weight_kg?: number
  color?: string
  notes?: string
  is_active: boolean
  created_at: string
  updated_at: string
  customer?: Customer
}

interface Cage {
  id: string
  cage_number: string
  cage_type: string
  is_occupied: boolean
  notes?: string
  created_at: string
  updated_at: string
}

interface Appointment {
  id: string
  pet_id: string
  customer_id: string
  doctor_id?: string
  appointment_date: string
  appointment_time: string
  visit_type: VisitType
  status: AppointmentStatus
  notes?: string
  created_at: string
  updated_at: string
  pet?: Pet
  customer?: Customer
  doctor?: User
}

interface Queue {
  id: string
  pet_id: string
  customer_id: string
  doctor_id?: string
  appointment_id?: string
  queue_number: number
  visit_type: string
  status: QueueStatus
  notes?: string
  queue_date: string
  created_at: string
  updated_at: string
  pet?: Pet
  customer?: Customer
  doctor?: User
}

interface MedicalRecord {
  id: string
  pet_id: string
  doctor_id: string
  queue_id?: string
  visit_date: string
  chief_complaint?: string
  physical_exam?: string
  diagnosis?: string
  treatment_plan?: string
  notes?: string
  follow_up_date?: string
  created_at: string
  updated_at: string
  pet?: Pet
  doctor?: User
  items?: MedicalRecordItem[]
}

interface MedicalRecordItem {
  id: string
  medical_record_id: string
  item_type: 'medicine' | 'action' | 'service'
  inventory_item_id?: string
  item_name: string
  quantity: number
  unit?: string
  price: number
  notes?: string
  created_at: string
  inventory_item?: InventoryItem
}

interface InventoryItem {
  id: string
  name: string
  category: InventoryCategory
  unit: string
  stock: number
  min_stock: number
  purchase_price: number
  selling_price: number
  description?: string
  is_active: boolean
  created_at: string
  updated_at: string
}

interface InventoryTransaction {
  id: string
  inventory_item_id: string
  transaction_type: InventoryTransactionType
  quantity: number
  reference_id?: string
  reference_type?: string
  notes?: string
  created_by?: string
  created_at: string
  inventory_item?: InventoryItem
  created_by_user?: User
}

interface Hospitalization {
  id: string
  pet_id: string
  doctor_id: string
  cage_id: string
  medical_record_id?: string
  admission_date: string
  discharge_date?: string
  initial_condition?: string
  status: HospitalizationStatus
  notes?: string
  created_at: string
  updated_at: string
  pet?: Pet
  doctor?: User
  cage?: Cage
  logs?: HospitalizationLog[]
}

interface HospitalizationLog {
  id: string
  hospitalization_id: string
  doctor_id?: string
  log_date: string
  condition_update: string
  treatment_given?: string
  created_at: string
  doctor?: User
}

interface Transaction {
  id: string
  invoice_number: string
  customer_id?: string
  pet_id?: string
  medical_record_id?: string
  cashier_id?: string
  subtotal: number
  discount: number
  total: number
  payment_method?: PaymentMethod
  payment_status: PaymentStatus
  paid_at?: string
  notes?: string
  created_at: string
  updated_at: string
  customer?: Customer
  pet?: Pet
  cashier?: User
  items?: TransactionItem[]
}

interface TransactionItem {
  id: string
  transaction_id: string
  item_type: ItemType
  inventory_item_id?: string
  item_name: string
  quantity: number
  unit_price: number
  subtotal: number
  created_at: string
  inventory_item?: InventoryItem
}

// ===========================
// FORM TYPES (Zod input shapes)
// ===========================
interface CustomerFormInput {
  full_name: string
  phone: string
  email?: string
  address?: string
}

interface PetFormInput {
  name: string
  species: string
  breed?: string
  gender: Gender
  birth_date?: string
  weight_kg?: number
  color?: string
  notes?: string
}

interface QueueFormInput {
  customer_id: string
  pet_id: string
  doctor_id?: string
  visit_type: VisitType
  notes?: string
}

interface MedicalRecordFormInput {
  chief_complaint?: string
  physical_exam?: string
  diagnosis?: string
  treatment_plan?: string
  notes?: string
  follow_up_date?: string
  items: MedicalRecordItemInput[]
  disposition: 'outpatient' | 'hospitalize'
  cage_id?: string
  initial_condition?: string
}

interface MedicalRecordItemInput {
  item_type: 'medicine' | 'action' | 'service'
  inventory_item_id?: string
  item_name: string
  quantity: number
  unit?: string
  price: number
  notes?: string
}

interface TransactionPaymentInput {
  payment_method: PaymentMethod
  amount_paid?: number
  discount?: number
  notes?: string
}
```

---

## 11. OFFLINE SYNC SPECIFICATION

### Strategi
```
- IndexedDB sebagai primary store saat offline
- Supabase sebagai source of truth saat online
- Conflict resolution: LAST WRITE WINS berdasarkan updated_at
- Sync direction: bidirectional (IndexedDB ↔ Supabase)
```

### Object Stores di IndexedDB
```
DB Name    : haland-petcare-db
Version    : 1

Stores:
- queue           (keyPath: id)
- customers       (keyPath: id)
- pets            (keyPath: id)
- inventory_items (keyPath: id)

Setiap record punya field tambahan:
- _synced    : boolean (sudah tersinkron ke Supabase atau belum)
- _deleted   : boolean (soft delete untuk sync)
- updated_at : string  (timestamp untuk conflict resolution)
```

### Sync Rules
```
1. Saat ONLINE:
   - Semua write → langsung ke Supabase + update IndexedDB
   - _synced = true

2. Saat OFFLINE:
   - Semua write → IndexedDB only
   - _synced = false

3. Saat kembali ONLINE:
   - Trigger syncToSupabase()
   - Ambil semua record dengan _synced = false
   - Push ke Supabase satu per satu
   - Jika sukses → update _synced = true di IndexedDB

4. Pull dari Supabase (saat awal load atau manual sync):
   - Ambil semua record yang updated_at > last_sync_timestamp
   - Simpan ke IndexedDB
   - Update last_sync_timestamp
```

### Modul yang Support Offline
```
✅ Queue (tambah pasien walk-in)
✅ Customers (tambah customer baru)
✅ Pets (tambah hewan baru)
✅ Inventory (lihat stok)
❌ Medical Records (butuh data dokter realtime)
❌ POS (butuh konfirmasi stok realtime)
❌ Reports (butuh data agregat terbaru)
```

---

## 12. KOMPONEN UI STANDAR

### Button
```
Variants : primary | secondary | danger | ghost
Sizes    : sm | md | lg
States   : default | loading | disabled
```

### Badge
```
Variants:
- waiting    → bg-yellow-100 text-yellow-800
- in_progress → bg-blue-100 text-blue-800
- done       → bg-green-100 text-green-800
- cancelled  → bg-red-100 text-red-800
- pending    → bg-gray-100 text-gray-800
- confirmed  → bg-green-100 text-green-800
- active     → bg-blue-100 text-blue-800
- discharged → bg-gray-100 text-gray-800
- paid       → bg-green-100 text-green-800
- unpaid     → bg-red-100 text-red-800
```

### Table
```
Props:
- columns: { key, label, render? }[]
- data: any[]
- loading: boolean
- emptyMessage: string
- onRowClick?: (row) => void
```

### Modal
```
Props:
- isOpen: boolean
- onClose: () => void
- title: string
- size: sm | md | lg | xl
- children: ReactNode
```

---

## 13. NAVIGASI & ROUTE GUARD

```
Route               Akses
/login              Public (redirect jika sudah login)
/dashboard          owner only
/reports            owner only
/users              owner only
/queue              doctor, staff
/medical-records/*  doctor (write), staff (read)
/hospitalization    doctor (write), staff (read)
/customers/*        staff, owner
/appointments       staff, doctor, customer
/inventory          staff, owner
/pos/*              staff, owner
/my-pets            customer only
```

### Redirect Setelah Login
```
owner    → /dashboard
doctor   → /queue
staff    → /queue
customer → /my-pets
```

---

## 14. FORMAT & KONVENSI

### Format Tanggal
```
Display  : DD MMMM YYYY (contoh: 15 Juni 2024)
Input    : YYYY-MM-DD
Jam      : HH:mm (24 jam)
```

### Format Currency
```
Locale   : id-ID
Currency : IDR
Contoh   : Rp 150.000
Fungsi   : formatCurrency(amount: number): string
```

### Format Nomor Antrian
```
Tampilan : #001, #002, dst
Reset    : tiap hari jam 00:00
```

### Konvensi Kode
```
Komponen  : PascalCase (CustomerTable.tsx)
Fungsi    : camelCase (formatCurrency)
Constants : UPPER_SNAKE_CASE (MAX_QUEUE_PER_DAY)
CSS class : TailwindCSS utility only, tidak ada custom CSS
File      : kebab-case (medical-record-form.tsx)
```

---

## 15. SEED DATA WAJIB

```sql
-- Akun owner
INSERT INTO auth.users ... → id: [owner-uuid]
INSERT INTO public.users (id, full_name, role) 
VALUES ('[owner-uuid]', 'Admin Owner', 'owner')

-- Akun doctor
INSERT INTO public.users (id, full_name, role)
VALUES ('[doctor-uuid]', 'drh. Budi Santoso', 'doctor')

-- Akun staff
INSERT INTO public.users (id, full_name, role)
VALUES ('[staff-uuid]', 'Siti Rahayu', 'staff')

-- Kandang awal
INSERT INTO public.cages (cage_number, cage_type) VALUES
('K-01', 'Kucing Kecil'),
('K-02', 'Kucing Kecil'),
('K-03', 'Kucing Besar'),
('A-01', 'Anjing Kecil'),
('A-02', 'Anjing Sedang'),
('A-03', 'Anjing Besar')

-- Inventory awal
INSERT INTO public.inventory_items (name, category, unit, stock, min_stock, selling_price) VALUES
('Amoxicillin 500mg', 'medicine', 'tablet', 100, 20, 5000),
('Paracetamol 500mg', 'medicine', 'tablet', 100, 20, 3000),
('Vitamin B Complex', 'medicine', 'tablet', 50, 10, 8000),
('Vaksin Rabies', 'vaccine', 'vial', 20, 5, 150000),
('Vaksin Distemper', 'vaccine', 'vial', 20, 5, 120000),
('Infus NaCl 500ml', 'supply', 'botol', 30, 10, 25000),
('Syringe 3ml', 'supply', 'pcs', 100, 20, 2000),
('Sarung Tangan Latex', 'supply', 'pcs', 200, 50, 3000)
```

---

## 16. DEFINISI SELESAI (DEFINITION OF DONE)

Setiap fitur dianggap selesai jika memenuhi semua kriteria ini:

```
✅ Semua field yang diperlukan sudah ada dan tervalidasi
✅ Loading state tampil saat fetch data
✅ Empty state tampil jika data kosong
✅ Error state tampil jika request gagal
✅ Confirm dialog muncul sebelum aksi destruktif
✅ Data refresh otomatis setelah create/update/delete
✅ RLS Supabase tidak memblokir akses yang seharusnya diizinkan
✅ Halaman bisa diakses oleh role yang tepat dan diblokir untuk role lain
✅ Responsive di layar 1280px (desktop klinik)
✅ Tidak ada console error saat digunakan normal
```

---

Ini adalah **sumber kebenaran mutlak** — setiap keputusan implementasi harus mengacu ke dokumen ini. Jika ada konflik antara instruksi Cline dan spesifikasi ini, **spesifikasi ini yang menang**.
