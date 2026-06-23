# BLUEPRINT.md - HALAND PETCARE v2.1

## Definisi Lengkap Sistem Arsitektur

---

## 1. OVERVIEW SISTEM

Haland PetCare adalah Clinic Operating System berbasis Local-First Architecture yang memastikan operasional klinik hewan tetap berjalan 100% tanpa internet. Sistem ini menggunakan PGlite sebagai database lokal (source of truth) dan Supabase PostgreSQL sebagai layer replication/backup.

**Prinsip Utama:**
- Local First, Cloud Second
- Offline Always Works
- No Single Point of Failure
- Complete Audit Trail
- Role-Based Access Control

---

## 2. TECHNOLOGY STACK DETAIL

### Frontend Layer
```
Next.js 15 + React 19
├── TypeScript (strict mode)
├── Tailwind CSS (utility-first)
├── shadcn/ui (accessible components)
└── PWA (offline capability)
```

### Service Worker & Offline
```
Service Worker Registration
├── Cache Strategy (Cache-First untuk assets)
├── Background Sync (untuk sync queue)
├── Offline Detection
└── Push Notifications
```

### Local Storage Layer
```
PGlite (PostgreSQL in Browser)
├── IndexedDB (fallback storage)
├── In-Memory Cache (hot data)
└── Local Storage (settings)
```

### ORM & Validation
```
Drizzle ORM
├── Type-Safe Queries
├── Migration Support
└── Schema Validation

Zod
├── Runtime Type Checking
├── API Input Validation
└── Error Messages
```

### State Management
```
Zustand
├── Global State (sync status, user)
├── Local State (UI)
├── Persistence (localStorage)
└── Subscription (event-based)
```

### Cloud Sync Layer
```
Supabase PostgreSQL
├── Replication Target
├── Backup Storage
├── Remote Access
└── Multi-Device Sync
```

### Deployment
```
Vercel
├── Next.js Optimization
├── Edge Functions
├── Automatic Deployments
└── Environment Management
```

---

## 3. DATABASE SCHEMA COMPLETE

### 3.1 Core Authentication Tables

#### local_users
```sql
CREATE TABLE local_users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  phone VARCHAR(20),
  full_name VARCHAR(255) NOT NULL,
  role VARCHAR(50) NOT NULL CHECK (role IN ('OWNER', 'DOCTOR', 'STAFF', 'CUSTOMER')),
  clinic_id UUID NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'INACTIVE', 'SUSPENDED')),
  last_login_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_email (email),
  INDEX idx_role (role)
);
```

#### local_sessions
```sql
CREATE TABLE local_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES local_users(id),
  device_id VARCHAR(255) NOT NULL,
  token VARCHAR(500) NOT NULL,
  ip_address VARCHAR(45),
  user_agent TEXT,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user_id (user_id),
  INDEX idx_device_id (device_id),
  INDEX idx_expires_at (expires_at)
);
```

### 3.2 Clinic & Device Management

#### clinics
```sql
CREATE TABLE clinics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  address TEXT NOT NULL,
  phone VARCHAR(20) NOT NULL,
  email VARCHAR(255),
  owner_id UUID NOT NULL REFERENCES local_users(id),
  timezone VARCHAR(50) DEFAULT 'Asia/Jakarta',
  logo_url TEXT,
  settings JSONB DEFAULT '{}'::jsonb,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_owner_id (owner_id)
);
```

#### devices
```sql
CREATE TABLE devices (
  id VARCHAR(255) PRIMARY KEY,
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  name VARCHAR(255) NOT NULL,
  type VARCHAR(50) NOT NULL CHECK (type IN ('Desktop', 'Laptop', 'Tablet', 'Mobile')),
  os VARCHAR(100),
  browser VARCHAR(100),
  registered_by UUID NOT NULL REFERENCES local_users(id),
  last_sync TIMESTAMP,
  last_activity TIMESTAMP,
  status VARCHAR(50) DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'INACTIVE', 'OFFLINE')),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_last_sync (last_sync)
);
```

### 3.3 Customer & Pet Management

#### customers
```sql
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  user_id UUID REFERENCES local_users(id),
  name VARCHAR(255) NOT NULL,
  phone VARCHAR(20) NOT NULL,
  email VARCHAR(255),
  address TEXT,
  city VARCHAR(100),
  province VARCHAR(100),
  postal_code VARCHAR(10),
  id_number VARCHAR(20),
  id_type VARCHAR(50),
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_user_id (user_id),
  INDEX idx_phone (phone),
  INDEX idx_name (name)
);
```

#### pets
```sql
CREATE TABLE pets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  name VARCHAR(255) NOT NULL,
  species VARCHAR(100) NOT NULL CHECK (species IN ('Dog', 'Cat', 'Bird', 'Rabbit', 'Hamster', 'Other')),
  breed VARCHAR(100),
  color VARCHAR(100),
  weight DECIMAL(5,2),
  age INT,
  age_unit VARCHAR(20) CHECK (age_unit IN ('days', 'months', 'years')),
  gender VARCHAR(20) CHECK (gender IN ('Male', 'Female', 'Unknown')),
  microchip_id VARCHAR(50),
  birth_date DATE,
  photo_url TEXT,
  medical_history TEXT,
  allergies TEXT,
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_customer_id (customer_id),
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_microchip_id (microchip_id)
);
```

### 3.4 Appointment Management

#### appointments
```sql
CREATE TABLE appointments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  customer_id UUID NOT NULL REFERENCES customers(id),
  pet_id UUID NOT NULL REFERENCES pets(id),
  doctor_id UUID NOT NULL REFERENCES local_users(id),
  appointment_date TIMESTAMP NOT NULL,
  appointment_type VARCHAR(100) NOT NULL CHECK (appointment_type IN ('Consultation', 'Vaccination', 'Surgery', 'Checkup', 'Grooming', 'Other')),
  reason TEXT,
  notes TEXT,
  status VARCHAR(50) DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'CONFIRMED', 'COMPLETED', 'CANCELLED', 'NO_SHOW')),
  created_by UUID NOT NULL REFERENCES local_users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_appointment_date (appointment_date),
  INDEX idx_customer_id (customer_id),
  INDEX idx_doctor_id (doctor_id),
  INDEX idx_status (status)
);
```

### 3.5 Queue Management

#### queues
```sql
CREATE TABLE queues (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  customer_id UUID NOT NULL REFERENCES customers(id),
  pet_id UUID NOT NULL REFERENCES pets(id),
  doctor_id UUID NOT NULL REFERENCES local_users(id),
  appointment_id UUID REFERENCES appointments(id),
  queue_number INT NOT NULL,
  arrival_time TIMESTAMP NOT NULL,
  estimated_wait_time INT,
  actual_start_time TIMESTAMP,
  status VARCHAR(50) DEFAULT 'WAITING' CHECK (status IN ('WAITING', 'IN_PROGRESS', 'COMPLETED', 'CANCELLED')),
  priority VARCHAR(50) DEFAULT 'NORMAL' CHECK (priority IN ('LOW', 'NORMAL', 'HIGH', 'EMERGENCY')),
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_doctor_id (doctor_id),
  INDEX idx_status (status),
  INDEX idx_queue_number (queue_number),
  INDEX idx_arrival_time (arrival_time)
);
```

### 3.6 Medical Records

#### medical_records
```sql
CREATE TABLE medical_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  pet_id UUID NOT NULL REFERENCES pets(id),
  queue_id UUID REFERENCES queues(id),
  doctor_id UUID NOT NULL REFERENCES local_users(id),
  visit_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  reason_for_visit TEXT NOT NULL,
  chief_complaint TEXT,
  history TEXT,
  
  -- Physical Examination
  temperature DECIMAL(4,1),
  heart_rate INT,
  respiratory_rate INT,
  blood_pressure VARCHAR(20),
  weight DECIMAL(5,2),
  body_condition_score INT CHECK (body_condition_score BETWEEN 1 AND 9),
  
  -- Findings
  physical_exam_findings TEXT,
  preliminary_diagnosis TEXT,
  
  -- Type of Visit
  visit_type VARCHAR(50) CHECK (visit_type IN ('Consultation', 'Follow-up', 'Hospitalization')),
  
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_pet_id (pet_id),
  INDEX idx_doctor_id (doctor_id),
  INDEX idx_visit_date (visit_date),
  INDEX idx_clinic_id (clinic_id)
);
```

### 3.7 Prescriptions & Medications

#### prescriptions
```sql
CREATE TABLE prescriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  medical_record_id UUID NOT NULL REFERENCES medical_records(id),
  doctor_id UUID NOT NULL REFERENCES local_users(id),
  prescription_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_medical_record_id (medical_record_id),
  INDEX idx_doctor_id (doctor_id),
  INDEX idx_clinic_id (clinic_id)
);
```

#### prescription_items
```sql
CREATE TABLE prescription_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  prescription_id UUID NOT NULL REFERENCES prescriptions(id),
  medicine_id UUID NOT NULL REFERENCES medicines(id),
  
  dosage DECIMAL(10,2),
  dosage_unit VARCHAR(50) CHECK (dosage_unit IN ('mg', 'ml', 'tablet', 'capsule', 'injection')),
  frequency VARCHAR(100),
  duration INT,
  duration_unit VARCHAR(20) CHECK (duration_unit IN ('days', 'weeks', 'months')),
  
  instructions TEXT,
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_prescription_id (prescription_id),
  INDEX idx_medicine_id (medicine_id)
);
```

#### medicines
```sql
CREATE TABLE medicines (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  name VARCHAR(255) NOT NULL,
  generic_name VARCHAR(255),
  manufacturer VARCHAR(255),
  category VARCHAR(100),
  unit VARCHAR(50),
  price DECIMAL(10,2),
  stock INT DEFAULT 0,
  min_stock INT DEFAULT 0,
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_name (name)
);
```

### 3.8 Hospitalization Management

#### cages
```sql
CREATE TABLE cages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  code VARCHAR(50) NOT NULL UNIQUE,
  size VARCHAR(50) CHECK (size IN ('Small', 'Medium', 'Large')),
  species VARCHAR(100),
  features TEXT,
  
  status VARCHAR(50) DEFAULT 'AVAILABLE' CHECK (status IN ('AVAILABLE', 'OCCUPIED', 'MAINTENANCE', 'CLEANING')),
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_status (status)
);
```

#### hospitalizations
```sql
CREATE TABLE hospitalizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  medical_record_id UUID NOT NULL REFERENCES medical_records(id),
  pet_id UUID NOT NULL REFERENCES pets(id),
  cage_id UUID NOT NULL REFERENCES cages(id),
  
  admission_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  estimated_discharge_date DATE,
  actual_discharge_date TIMESTAMP,
  
  reason_for_hospitalization TEXT,
  initial_condition TEXT,
  
  status VARCHAR(50) DEFAULT 'ADMITTED' CHECK (status IN ('ADMITTED', 'MONITORING', 'RECOVERING', 'DISCHARGED', 'TRANSFERRED')),
  
  daily_cost DECIMAL(10,2),
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_pet_id (pet_id),
  INDEX idx_admission_date (admission_date),
  INDEX idx_status (status)
);
```

#### hospitalization_monitoring
```sql
CREATE TABLE hospitalization_monitoring (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  hospitalization_id UUID NOT NULL REFERENCES hospitalizations(id),
  doctor_id UUID NOT NULL REFERENCES local_users(id),
  
  monitoring_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  temperature DECIMAL(4,1),
  heart_rate INT,
  respiratory_rate INT,
  blood_pressure VARCHAR(20),
  
  condition_notes TEXT,
  medications_given TEXT,
  food_intake VARCHAR(100),
  urine_output VARCHAR(100),
  stool_output VARCHAR(100),
  
  overall_status VARCHAR(50) CHECK (overall_status IN ('Stable', 'Improving', 'Declining', 'Critical')),
  
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_hospitalization_id (hospitalization_id),
  INDEX idx_monitoring_date (monitoring_date)
);
```

### 3.9 Inventory Management

#### inventory_items
```sql
CREATE TABLE inventory_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  item_type VARCHAR(50) CHECK (item_type IN ('Medicine', 'Consumable', 'Equipment', 'Supply')),
  name VARCHAR(255) NOT NULL,
  code VARCHAR(100) UNIQUE,
  category VARCHAR(100),
  unit VARCHAR(50),
  
  quantity_on_hand INT DEFAULT 0,
  reorder_level INT,
  reorder_quantity INT,
  
  unit_price DECIMAL(10,2),
  supplier_id UUID,
  supplier_name VARCHAR(255),
  
  location VARCHAR(100),
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_item_type (item_type)
);
```

#### inventory_batches

```sql
CREATE TABLE inventory_batches (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  inventory_item_id UUID NOT NULL REFERENCES inventory_items(id),
  batch_number VARCHAR(100) NOT NULL,
  quantity INT NOT NULL,
  manufacturing_date DATE,
  expiry_date DATE NOT NULL,
  unit_cost DECIMAL(10,2),
  received_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_inventory_item_id (inventory_item_id),
  INDEX idx_expiry_date (expiry_date),
  INDEX idx_batch_number (batch_number)
);
```

#### inventory_transactions
```sql
CREATE TABLE inventory_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  inventory_item_id UUID NOT NULL REFERENCES inventory_items(id),
  batch_id UUID REFERENCES inventory_batches(id),
  
  transaction_type VARCHAR(50) CHECK (transaction_type IN ('IN', 'OUT', 'ADJUSTMENT', 'DAMAGE', 'RETURN')),
  quantity INT NOT NULL,
  
  reference_type VARCHAR(100),
  reference_id UUID,
  
  notes TEXT,
  created_by UUID NOT NULL REFERENCES local_users(id),
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_inventory_item_id (inventory_item_id),
  INDEX idx_transaction_type (transaction_type),
  INDEX idx_created_at (created_at)
);
```

### 3.10 Payment & POS

#### invoices
```sql
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  invoice_number VARCHAR(50) UNIQUE NOT NULL,
  
  customer_id UUID NOT NULL REFERENCES customers(id),
  pet_id UUID REFERENCES pets(id),
  
  reference_type VARCHAR(100),
  reference_id UUID,
  
  issue_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  due_date DATE,
  
  subtotal DECIMAL(12,2),
  tax DECIMAL(12,2),
  discount DECIMAL(12,2),
  total DECIMAL(12,2),
  
  status VARCHAR(50) DEFAULT 'DRAFT' CHECK (status IN ('DRAFT', 'ISSUED', 'PAID', 'PARTIAL', 'OVERDUE', 'CANCELLED')),
  payment_method VARCHAR(100),
  
  notes TEXT,
  
  created_by UUID NOT NULL REFERENCES local_users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_customer_id (customer_id),
  INDEX idx_status (status),
  INDEX idx_issue_date (issue_date)
);
```

#### invoice_items
```sql
CREATE TABLE invoice_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id UUID NOT NULL REFERENCES invoices(id),
  
  item_type VARCHAR(50) CHECK (item_type IN ('Service', 'Medicine', 'Consumable', 'Procedure')),
  item_name VARCHAR(255) NOT NULL,
  item_id UUID,
  
  quantity DECIMAL(10,2),
  unit_price DECIMAL(10,2),
  subtotal DECIMAL(12,2),
  
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_invoice_id (invoice_id)
);
```

#### payments
```sql
CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  invoice_id UUID NOT NULL REFERENCES invoices(id),
  
  payment_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  amount DECIMAL(12,2),
  
  payment_method VARCHAR(100) CHECK (payment_method IN ('Cash', 'Card', 'Transfer', 'Check', 'Other')),
  payment_reference VARCHAR(255),
  
  notes TEXT,
  received_by UUID NOT NULL REFERENCES local_users(id),
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_invoice_id (invoice_id),
  INDEX idx_payment_date (payment_date)
);
```

### 3.11 Audit & Sync

#### audit_logs
```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  user_id UUID REFERENCES local_users(id),
  device_id VARCHAR(255) REFERENCES devices(id),
  
  action VARCHAR(100) NOT NULL CHECK (action IN (
    'INSERT', 'UPDATE', 'DELETE', 
    'LOGIN', 'LOGOUT', 
    'CREATE_USER', 'RESET_PASSWORD', 
    'PAYMENT', 'DISCHARGE', 
    'PRESCRIBE', 'HOSPITALIZE',
    'INVENTORY_ADJUST', 'SYNC_ATTEMPT'
  )),
  
  table_name VARCHAR(100),
  record_id UUID,
  
  old_data JSONB,
  new_data JSONB,
  
  ip_address VARCHAR(45),
  user_agent TEXT,
  
  status VARCHAR(50) DEFAULT 'SUCCESS' CHECK (status IN ('SUCCESS', 'FAILED', 'PARTIAL')),
  error_message TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_user_id (user_id),
  INDEX idx_action (action),
  INDEX idx_created_at (created_at),
  INDEX idx_table_name (table_name)
);
```

#### sync_queue
```sql
CREATE TABLE sync_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  device_id VARCHAR(255) NOT NULL REFERENCES devices(id),
  
  table_name VARCHAR(100) NOT NULL,
  record_id UUID NOT NULL,
  operation VARCHAR(20) CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
  
  payload JSONB NOT NULL,
  entity_version INT DEFAULT 1,
  checksum VARCHAR(64),
  
  synced BOOLEAN DEFAULT FALSE,
  synced_at TIMESTAMP,
  
  retry_count INT DEFAULT 0,
  last_attempt_at TIMESTAMP,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_synced (synced),
  INDEX idx_created_at (created_at),
  INDEX idx_device_id (device_id)
);
```

#### conflict_queue

```sql
CREATE TABLE conflict_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  device_id VARCHAR(255) NOT NULL REFERENCES devices(id),
  
  table_name VARCHAR(100) NOT NULL,
  record_id UUID NOT NULL,
  
  local_version INT,
  local_data JSONB,
  
  server_version INT,
  server_data JSONB,
  
  conflict_detected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  status VARCHAR(50) DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'RESOLVED', 'MANUAL_REVIEW')),
  resolution_strategy VARCHAR(100) CHECK (resolution_strategy IN ('LOCAL_WINS', 'SERVER_WINS', 'MERGE', 'MANUAL')),
  
  resolved_by UUID REFERENCES local_users(id),
  resolved_data JSONB,
  resolved_at TIMESTAMP,
  
  notes TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_status (status)
);
```

#### sync_logs
```sql
CREATE TABLE sync_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id UUID NOT NULL REFERENCES clinics(id),
  device_id VARCHAR(255) NOT NULL REFERENCES devices(id),
  
  sync_start_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  sync_end_time TIMESTAMP,
  duration_ms INT,
  
  status VARCHAR(50) CHECK (status IN ('SUCCESS', 'FAILED', 'PARTIAL')),
  
  items_synced INT DEFAULT 0,
  items_failed INT DEFAULT 0,
  conflicts_detected INT DEFAULT 0,
  
  error_message TEXT,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_clinic_id (clinic_id),
  INDEX idx_device_id (device_id),
  INDEX idx_sync_start_time (sync_start_time)
);
```

---

## 4. AUTHENTICATION FLOW DETAIL

### 4.1 Login Flow (Offline-First)

```
User Input Email & Password
    ↓
[ONLINE?] YES → Send to Cloud (verify backup)
    ↓ NO
Lookup in local_users table
    ↓
Password Hash Match?
    ↓ YES
Generate Session Token
    ↓
Create local_sessions record
    ↓
Set zustand global state
    ↓
Store token in localStorage
    ↓
Register device if new
    ↓
✓ Access Granted
```

### 4.2 Session Management

**Token Storage:**
- Primary: localStorage (encrypted)
- Fallback: in-memory (sessionStorage)
- Device ID: unique identifier per browser

**Session Validation:**
```typescript
// Every API call validates:
1. Token exists in local_sessions
2. Token not expired
3. User status = ACTIVE
4. Role has permission
```

**Automatic Logout:**
- Token expires after 30 days
- Inactivity after 8 hours
- Manual logout clears session

### 4.3 Multi-Device Sync

**Device Registration:**
```sql
INSERT INTO devices
  (id, clinic_id, name, type, registered_by, status)
VALUES
  (uuid, clinic_id, 'Desktop - Dr. Budi', 'Desktop', user_id, 'ACTIVE')
```

**Session Across Devices:**
- Each device gets unique device_id
- Same user_id can have multiple sessions
- Logout deletes specific device session only

---

## 5. ROLE-BASED ACCESS CONTROL (RBAC)

### 5.1 Permission Matrix

| Feature | OWNER | DOCTOR | STAFF | CUSTOMER |
|---------|-------|--------|-------|----------|
| User Management | ✓ | ✗ | ✗ | ✗ |
| Queue Management | ✓ | ✓ | ✓ | ✗ |
| Medical Records | ✓ | ✓ | ✓ | View Own |
| Prescriptions | ✓ | ✓ | ✗ | View Own |
| Hospitalization | ✓ | ✓ | ✓ | View Own |
| Inventory | ✓ | ✗ | ✓ | ✗ |
| POS/Payment | ✓ | ✗ | ✓ | ✗ |
| Reports | ✓ | ✓ | Limited | ✗ |
| Settings | ✓ | ✗ | ✗ | ✗ |

### 5.2 Permission Checking

```typescript
// Middleware check sebelum setiap operasi
const checkPermission = async (
  userId: UUID,
  action: string,
  resource: string
): Promise<boolean> => {
  // 1. Get user role dari local_users
  const user = await getUser(userId);
  
  // 2. Check role permissions
  const allowed = ROLE_PERMISSIONS[user.role][action];
  
  // 3. If resource requires ownership:
  if (requiresOwnership(resource)) {
    const owner = await getResourceOwner(resource);
    return owner === userId;
  }
  
  // 4. Log attempt
  await auditLog({
    user_id: userId,
    action: allowed ? 'PERMITTED' : 'DENIED',
    resource
  });
  
  return allowed;
};
```

---

## 6. OFFLINE-FIRST WORKFLOW DETAIL

### 6.1 Data Availability Offline

| Feature | Offline | Notes |
|---------|---------|-------|
| Login | ✓ | From local_users |
| Customer Search | ✓ | From local cache |
| Pet Management | ✓ | From local cache |
| Queue Management | ✓ | Full functionality |
| Medical Records | ✓ | Create & view |
| Prescriptions | ✓ | Create & modify |
| Inventory | ✓ | View only (write queued) |
| POS | ✓ | Create invoice |
| Payment | ✓ | Record locally |
| Reports | Partial | Cache only |

### 6.2 Sync Queue Mechanism

**When User is Offline:**
```
Create/Update/Delete Operation
    ↓
Write to PGlite immediately
    ↓
Add to sync_queue table
    ↓
Show "PENDING SYNC" badge
    ↓
Continue working normally
```

**When Connection Restored:**
```
Detect ONLINE state
    ↓
Background Sync starts
    ↓
Process sync_queue in order (by created_at)
    ↓
Send batches to /api/sync endpoint
    ↓
Verify server acknowledgment
    ↓
Mark as synced = true
    ↓
If conflict detected → conflict_queue
    ↓
Show sync status to user
```

### 6.3 Offline Mode Indicators

**Global Status Bar:**
```
┌─────────────────────────────────────────┐
│ OFFLINE (Red) | Last Sync: 2 min ago   │
│ Items Pending: 5                        │
└─────────────────────────────────────────┘
```

**Per-Item Indicator:**
```
Medical Record #123  [⟳ SYNCING]
Customer: John Doe   [✓ SYNCED]
Queue #05            [⚠ PENDING]
```

---

## 7. SYNC ENGINE ARCHITECTURE

### 7.1 Sync Strategy

**Optimistic Updates:**
```
User Action
    ↓
Update UI immediately (optimistic)
    ↓
Write to PGlite
    ↓
Add to sync_queue
    ↓
If ONLINE → sync immediately
    ↓
If OFFLINE → continue anyway
    ↓
On sync success → confirm
    ↓
On sync failure → rollback
```

### 7.2 Batch Sync Logic

```typescript
async function backgroundSync() {
  // Get all pending items
  const pending = await db.select()
    .from(sync_queue)
    .where(eq(sync_queue.synced, false))
    .orderBy(asc(sync_queue.created_at))
    .limit(100); // Batch size

  // Group by table for efficient processing
  const grouped = groupBy(pending, 'table_name');

  for (const [table, items] of Object.entries(grouped)) {
    try {
      const response = await fetch('/api/sync', {
        method: 'POST',
        body: JSON.stringify({ table, items })
      });

      if (response.ok) {
        // Mark as synced
        await db.update(sync_queue)
          .set({ synced: true, synced_at: now() })
          .where(
            inArray(sync_queue.id, items.map(i => i.id))
          );
      } else {
        // Retry with backoff
        await updateRetryCount(items);
      }
    } catch (error) {
      await logSyncError(table, items, error);
    }
  }
}
```

### 7.3 Conflict Resolution Strategy

**Never Last-Write-Wins!**

```typescript
async function detectConflict(
  local: Record,
  server: Record
): Promise<Conflict | null> {
  
  // Compare versions
  if (local.version === server.version) {
    return null; // No conflict
  }

  // Both modified since last sync
  if (local.modified_at > local.synced_at &&
      server.modified_at > local.synced_at) {
    
    // Create conflict entry
    return {
      table_name: local.table,
      record_id: local.id,
      local_version: local.version,
      server_version: server.version,
      local_data: local,
      server_data: server,
      status: 'PENDING'
    };
  }

  // Server wins if it's newer
  if (server.modified_at > local.modified_at) {
    return null; // No conflict, just update local
  }
}
```

**Manual Resolution Flow:**
```
Admin Dashboard → Conflicts Tab
    ↓
Display conflicting records
    ↓
Admin chooses resolution:
  - Keep Local Version
  - Use Server Version
  - Manual Merge
    ↓
Update conflict_queue
    ↓
Sync continues
```

---

## 8. INVENTORY TRANSACTION SAFETY

### 8.1 Double Deduction Prevention

**Problem:**
```
Stock: 10
User A: Deduct 5
User B: Deduct 5
Risk: Both deductions succeed (stock = 0), but should fail one
```

**Solution: Database Transaction**

```typescript
async function deductInventory(
  itemId: UUID,
  quantity: number,
  reference: { type: string; id: UUID }
): Promise<void> {
  
  return await db.transaction(async (trx) => {
    // 1. Lock row (SELECT FOR UPDATE equivalent in PGlite)
    const item = await trx.select()
      .from(inventory_items)
      .where(eq(inventory_items.id, itemId));

    if (item.quantity_on_hand < quantity) {
      throw new Error('INSUFFICIENT_STOCK');
    }

    // 2. Deduct
    await trx.update(inventory_items)
      .set({
        quantity_on_hand: item.quantity_on_hand - quantity
      })
      .where(eq(inventory_items.id, itemId));

    // 3. Record transaction
    await trx.insert(inventory_transactions)
      .values({
        inventory_item_id: itemId,
        transaction_type: 'OUT',
        quantity,
        reference_type: reference.type,
        reference_id: reference.id
      });

    // 4. Audit log
    await trx.insert(audit_logs)
      .values({
        action: 'INVENTORY_ADJUST',
        table_name: 'inventory_items',
        record_id: itemId,
        new_data: { quantity_deducted: quantity }
      });
  });
}
```

### 8.2 Inventory Batches (FIFO)

```typescript
async function getExpiringSoonBatch(
  itemId: UUID
): Promise<InventoryBatch | null> {
  
  return await db.select()
    .from(inventory_batches)
    .where(
      and(
        eq(inventory_batches.inventory_item_id, itemId),
        gt(inventory_batches.quantity, 0),
        isNotNull(inventory_batches.expiry_date)
      )
    )
    .orderBy(asc(inventory_batches.expiry_date))
    .limit(1)
    .then(rows => rows[0] ?? null);
}
```

---

## 9. POS & PAYMENT WORKFLOW

### 9.1 Invoice Creation

```
Customer Arrives
    ↓
Create Queue Entry
    ↓
Doctor Examination & Medical Record
    ↓
Decision:
  ├─ OUTPATIENT → Create Invoice
  └─ HOSPITALIZATION → Create Hospitalization Record

Invoice Creation:
    ├─ Auto-generate invoice_number
    ├─ Add services from medical record
    ├─ Add medicines from prescription
    ├─ Add inventory deductions
    ├─ Calculate subtotal, tax, discount
    ├─ Create invoice_items records
    └─ Status = DRAFT
```

### 9.2 Payment Processing

```typescript
async function processPayment(
  invoiceId: UUID,
  amount: number,
  method: PaymentMethod
): Promise<void> {
  
  return await db.transaction(async (trx) => {
    // 1. Lock invoice
    const invoice = await trx.select()
      .from(invoices)
      .where(eq(invoices.id, invoiceId));

    // 2. Validate amount
    if (amount > invoice.total) {
      throw new Error('OVERPAYMENT');
    }

    // 3. Create payment record
    const payment = await trx.insert(payments)
      .values({
        invoice_id: invoiceId,
        amount,
        payment_method: method
      })
      .returning();

    // 4. Update invoice status
    const newTotal = invoice.total - amount;
    const newStatus = newTotal === 0 ? 'PAID' : 'PARTIAL';
    
    await trx.update(invoices)
      .set({ status: newStatus })
      .where(eq(invoices.id, invoiceId));

    // 5. Update inventory (deduct medicines used)
    for (const item of invoice.items) {
      if (item.item_type === 'Medicine') {
        await deductInventory(
          item.item_id,
          item.quantity,
          { type: 'INVOICE', id: invoiceId }
        );
      }
    }

    // 6. Audit log
    await trx.insert(audit_logs)
      .values({
        action: 'PAYMENT',
        table_name: 'payments',
        record_id: payment.id,
        new_data: { amount, method }
      });
  });
}
```

### 9.3 Invoice Printing

```typescript
// Generate receipt in multiple formats
async function generateReceipt(invoiceId: UUID) {
  const invoice = await getInvoiceWithItems(invoiceId);
  
  return {
    // Thermal printer format (80mm)
    thermal: formatThermalReceipt(invoice),
    
    // A4 PDF
    pdf: await generatePDF(invoice),
    
    // Email format
    email: formatEmailReceipt(invoice),
    
    // QR code for follow-up
    qrCode: generateQRCode(invoiceId)
  };
}
```

---

## 10. HOSPITALIZATION WORKFLOW COMPLETE

### 10.1 Admission Process

```
Medical Record Created
    ↓
Doctor: "Admit to Hospital"
    ↓
Find Available Cage
    ├─ Filter by species & size
    ├─ Status = AVAILABLE
    └─ Display options
    ↓
Select Cage
    ↓
Create Hospitalization Record
    ├─ admission_date = now()
    ├─ cage status = OCCUPIED
    ├─ cage_id linked
    └─ status = ADMITTED
    ↓
Create initial monitoring record
    ├─ temperature
    ├─ heart_rate
    ├─ condition_notes
    └─ timestamp
    ↓
✓ Hospitalization Started
```

### 10.2 Daily Monitoring

```
Doctor Views Hospitalization
    ↓
History of monitoring records shown
    ↓
Create New Monitoring Entry
    ├─ Vital signs
    ├─ Condition assessment
    ├─ Medications given
    ├─ Food/urine/stool output
    └─ Overall status
    ↓
Save monitoring_date = now()
    ↓
Update hospitalization status
    ├─ IMPROVING
    ├─ STABLE
    ├─ DECLINING
    └─ CRITICAL
    ↓
✓ Monitoring Recorded
```

### 10.3 Discharge Process

```typescript
async function dischargePatient(
  hospitalizationId: UUID,
  dischargeNotes: string
): Promise<void> {
  
  return await db.transaction(async (trx) => {
    // 1. Get hospitalization
    const hosp = await getHospitalization(hospitalizationId);

    // 2. Calculate final cost
    const daysStayed = daysBetween(
      hosp.admission_date,
      now()
    );
    const totalCost = daysStayed * hosp.daily_cost;

    // 3. Update hospitalization
    await trx.update(hospitalizations)
      .set({
        actual_discharge_date: now(),
        status: 'DISCHARGED'
      })
      .where(eq(hospitalizations.id, hospitalizationId));

    // 4. Release cage
    await trx.update(cages)
      .set({ status: 'AVAILABLE' })
      .where(eq(cages.id, hosp.cage_id));

    // 5. Create invoice
    const invoice = await trx.insert(invoices)
      .values({
        customer_id: hosp.customer_id,
        reference_type: 'HOSPITALIZATION',
        reference_id: hospitalizationId,
        total: totalCost,
        status: 'ISSUED'
      })
      .returning();

    // 6. Add invoice items
    await trx.insert(invoice_items)
      .values({
        invoice_id: invoice.id,
        item_type: 'Service',
        item_name: `Hospitalization (${daysStayed} days)`,
        quantity: daysStayed,
        unit_price: hosp.daily_cost,
        subtotal: totalCost
      });

    // 7. Audit
    await trx.insert(audit_logs)
      .values({
        action: 'DISCHARGE',
        table_name: 'hospitalizations',
        record_id: hospitalizationId,
        new_data: { status: 'DISCHARGED', totalCost }
      });
  });
}
```

---

## 11. QUEUE MANAGEMENT SYSTEM

### 11.1 Queue Workflow

```
Customer Arrives (Walk-in)
    ↓
Search/Create Customer
    ↓
Select Pet
    ↓
Auto-generate Queue Number
    ├─ Format: ClinicID-YYYY-MM-DD-XXXX
    ├─ Example: KLN-2026-06-23-0001
    └─ Reset daily at midnight
    ↓
Create Queue Record
    ├─ queue_number
    ├─ arrival_time = now()
    ├─ doctor_id selected
    ├─ status = WAITING
    └─ priority = NORMAL
    ↓
Display on Queue Monitor
    ├─ All WAITING
    ├─ Sorted by queue_number
    └─ Estimated wait time
    ↓
When doctor ready:
    ├─ Click "Next"
    ├─ Queue status = IN_PROGRESS
    ├─ actual_start_time = now()
    └─ Medical record opened
    ↓
After examination:
    ├─ Queue status = COMPLETED
    ├─ Medical record saved
    └─ Decision: OUTPATIENT/HOSPITALIZATION
```

### 11.2 Priority Queue Handling

```typescript
async function getNextInQueue(doctorId: UUID): Promise<Queue> {
  
  // First: EMERGENCY priority
  let next = await db.select()
    .from(queues)
    .where(
      and(
        eq(queues.doctor_id, doctorId),
        eq(queues.status, 'WAITING'),
        eq(queues.priority, 'EMERGENCY')
      )
    )
    .orderBy(asc(queues.arrival_time))
    .limit(1);

  if (next.length > 0) return next[0];

  // Second: HIGH priority
  next = await db.select()
    .from(queues)
    .where(
      and(
        eq(queues.doctor_id, doctorId),
        eq(queues.status, 'WAITING'),
        eq(queues.priority, 'HIGH')
      )
    )
    .orderBy(asc(queues.arrival_time))
    .limit(1);

  if (next.length > 0) return next[0];

  // Default: FIFO
  next = await db.select()
    .from(queues)
    .where(
      and(
        eq(queues.doctor_id, doctorId),
        eq(queues.status, 'WAITING')
      )
    )
    .orderBy(asc(queues.queue_number))
    .limit(1);

  return next[0];
}
```

### 11.3 Queue Monitor Display

```typescript
// Real-time queue status untuk display di clinic
async function getQueueStatus(clinicId: UUID) {
  const current = await db.select()
    .from(queues)
    .where(
      and(
        eq(queues.clinic_id, clinicId),
        eq(queues.status, 'IN_PROGRESS')
      )
    );

  const waiting = await db.select()
    .from(queues)
    .where(
      and(
        eq(queues.clinic_id, clinicId),
        eq(queues.status, 'WAITING')
      )
    )
    .orderBy(asc(queues.queue_number));

  return {
    currentServing: current.map(q => ({
      queueNumber: q.queue_number,
      petName: q.pet_name,
      doctorName: q.doctor_name,
      counter: q.doctor_id
    })),
    
    waiting: waiting.map((q, idx) => ({
      position: idx + 1,
      queueNumber: q.queue_number,
      petName: q.pet_name,
      estimatedWait: calculateWaitTime(waiting, idx)
    }))
  };
}
```

---

## 12. AUDIT LOGGING SYSTEM

### 12.1 Audit Log Entry

```typescript
async function auditLog(
  userId: UUID,
  action: string,
  tableName: string,
  recordId: UUID,
  oldData?: Record<string, any>,
  newData?: Record<string, any>
): Promise<void> {
  
  await db.insert(audit_logs).values({
    clinic_id: getCurrentClinic(),
    user_id: userId,
    device_id: getDeviceId(),
    action,
    table_name: tableName,
    record_id: recordId,
    old_data: oldData || null,
    new_data: newData || null,
    ip_address: getClientIp(),
    user_agent: navigator.userAgent,
    status: 'SUCCESS'
  });
}
```

### 12.2 Automatic Audit Triggers

```typescript
// Wrap setiap database operation
async function createRecord<T>(
  table: string,
  data: T,
  userId: UUID
): Promise<T> {
  
  const result = await db.insert(table).values(data).returning();
  
  await auditLog(
    userId,
    'INSERT',
    table,
    result.id,
    null,
    result
  );
  
  return result;
}

async function updateRecord<T>(
  table: string,
  recordId: UUID,
  changes: Partial<T>,
  userId: UUID
): Promise<T> {
  
  const oldData = await db.select()
    .from(table)
    .where(eq(table.id, recordId));
  
  const newData = await db.update(table)
    .set(changes)
    .where(eq(table.id, recordId))
    .returning();
  
  await auditLog(
    userId,
    'UPDATE',
    table,
    recordId,
    oldData[0],
    newData[0]
  );
  
  return newData[0];
}
```

### 12.3 Audit Report

```typescript
async function getAuditReport(
  clinicId: UUID,
  filters: {
    userId?: UUID;
    action?: string;
    tableName?: string;
    dateFrom?: Date;
    dateTo?: Date;
  }
): Promise<AuditLog[]> {
  
  let query = db.select()
    .from(audit_logs)
    .where(eq(audit_logs.clinic_id, clinicId));

  if (filters.userId) {
    query = query.where(eq(audit_logs.user_id, filters.userId));
  }

  if (filters.action) {
    query = query.where(eq(audit_logs.action, filters.action));
  }

  if (filters.tableName) {
    query = query.where(eq(audit_logs.table_name, filters.tableName));
  }

  if (filters.dateFrom) {
    query = query.where(
      gte(audit_logs.created_at, filters.dateFrom)
    );
  }

  if (filters.dateTo) {
    query = query.where(
      lte(audit_logs.created_at, filters.dateTo)
    );
  }

  return await query
    .orderBy(desc(audit_logs.created_at))
    .limit(10000);
}
```

---

## 13. BACKUP & RECOVERY SYSTEM

### 13.1 Automatic Backup Schedule

```typescript
// Run at 02:00 AM daily
async function scheduleDailyBackup(clinicId: UUID) {
  const backup = await db.transaction(async (trx) => {
    // 1. Export all tables to JSON
    const data = {
      version: BACKUP_VERSION,
      timestamp: now(),
      clinic_id: clinicId,
      tables: {}
    };

    const tables = [
      'local_users', 'customers', 'pets',
      'appointments', 'queues', 'medical_records',
      'prescriptions', 'hospitalizations',
      'invoices', 'payments', 'inventory_items'
    ];

    for (const table of tables) {
      const records = await trx.select().from(table);
      data.tables[table] = records;
    }

    // 2. Compress
    const compressed = await gzip(JSON.stringify(data));

    // 3. Store locally
    const backupId = generateUUID();
    const filename = `backup-${clinicId}-${formatDate(now())}.json.gz`;
    
    await saveToIndexedDB('backups', {
      id: backupId,
      filename,
      size: compressed.byteLength,
      created_at: now(),
      type: 'LOCAL'
    }, compressed);

    // 4. If online, upload to Supabase
    if (isOnline()) {
      try {
        await supabaseStorage.upload(
          `backups/${clinicId}/${filename}`,
          compressed
        );

        await db.update(backup_metadata)
          .set({ synced_at: now() })
          .where(eq(backup_metadata.id, backupId));
      } catch (error) {
        // Backup exists locally, sync will retry later
        logError('Backup upload failed', error);
      }
    }

    return backupId;
  });

  return backup;
}
```

### 13.2 Restore from Backup

```typescript
async function restoreBackup(backupId: UUID): Promise<void> {
  return await db.transaction(async (trx) => {
    // 1. Get backup data
    const backup = await getBackupFromIndexedDB(backupId);
    const decompressed = await ungzip(backup);
    const data = JSON.parse(decompressed);

    // 2. Verify integrity
    if (data.version !== BACKUP_VERSION) {
      throw new Error('INCOMPATIBLE_BACKUP_VERSION');
    }

    // 3. Clear current data (optional warning)
    const confirmed = confirm(
      'This will replace all current data. Continue?'
    );
    if (!confirmed) return;

    // 4. Restore tables
    for (const [tableName, records] of Object.entries(data.tables)) {
      // Clear existing
      await trx.delete(table(tableName));

      // Insert restored
      for (const record of records) {
        await trx.insert(table(tableName)).values(record);
      }
    }

    // 5. Log restoration
    await auditLog(
      getCurrentUserId(),
      'RESTORE_BACKUP',
      'system',
      backupId
    );

    showNotification('Backup restored successfully');
  });
}
```

---

## 14. PERFORMANCE OPTIMIZATION

### 14.1 Indexing Strategy

```sql
-- Primary lookups
CREATE INDEX idx_clinic_id ON [table] (clinic_id);
CREATE INDEX idx_user_id ON [table] (user_id);
CREATE INDEX idx_customer_id ON [table] (customer_id);
CREATE INDEX idx_pet_id ON [table] (pet_id);

-- Search & filtering
CREATE INDEX idx_email ON local_users (email);
CREATE INDEX idx_phone ON customers (phone);
CREATE INDEX idx_name ON customers (name);

-- Time-based queries
CREATE INDEX idx_created_at ON [table] (created_at);
CREATE INDEX idx_appointment_date ON appointments (appointment_date);
CREATE INDEX idx_admission_date ON hospitalizations (admission_date);

-- Status filtering
CREATE INDEX idx_status ON [table] (status);

-- Composite indexes
CREATE INDEX idx_clinic_status ON queues (clinic_id, status);
CREATE INDEX idx_pet_date ON medical_records (pet_id, visit_date);
```

### 14.2 Query Optimization

```typescript
// BAD: N+1 query
const customers = await db.select().from(customers);
for (const customer of customers) {
  const pets = await db.select()
    .from(pets)
    .where(eq(pets.customer_id, customer.id));
  customer.pets = pets;
}

// GOOD: Join
const customersWithPets = await db.select({
  customer: customers,
  pets: pets
})
.from(customers)
.leftJoin(pets, eq(customers.id, pets.customer_id))
.where(eq(customers.clinic_id, clinicId));
```

### 14.3 Caching Strategy

```typescript
// In-memory cache untuk hot data
const cache = new Map<string, any>();

async function getCachedCustomer(customerId: UUID) {
  const cacheKey = `customer:${customerId}`;
  
  if (cache.has(cacheKey)) {
    return cache.get(cacheKey);
  }

  const customer = await db.select()
    .from(customers)
    .where(eq(customers.id, customerId));

  cache.set(cacheKey, customer, {
    ttl: 5 * 60 * 1000 // 5 minutes
  });

  return customer;
}

// Invalidate on update
async function updateCustomer(customerId: UUID, data: any) {
  const result = await db.update(customers)
    .set(data)
    .where(eq(customers.id, customerId));

  cache.delete(`customer:${customerId}`);
  
  return result;
}
```

---

## 15. ERROR HANDLING & VALIDATION

### 15.1 Input Validation dengan Zod

```typescript
import { z } from 'zod';

const CreateCustomerSchema = z.object({
  name: z.string().min(1).max(255),
  phone: z.string().regex(/^(\+62|62|0)[0-9]{9,12}$/),
  email: z.string().email().optional(),
  address: z.string().max(500),
  city: z.string().max(100),
  province: z.string().max(100),
  postal_code: z.string().regex(/^[0-9]{5}$/)
});

// Usage
async function createCustomer(data: unknown) {
  try {
    const validated = CreateCustomerSchema.parse(data);
    return await db.insert(customers).values(validated);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return {
        error: 'VALIDATION_FAILED',
        details: error.errors
      };
    }
    throw error;
  }
}
```

### 15.2 Error Codes

```typescript
enum ErrorCode {
  // Auth
  INVALID_CREDENTIALS = 'INVALID_CREDENTIALS',
  SESSION_EXPIRED = 'SESSION_EXPIRED',
  PERMISSION_DENIED = 'PERMISSION_DENIED',

  // Validation
  VALIDATION_FAILED = 'VALIDATION_FAILED',
  DUPLICATE_ENTRY = 'DUPLICATE_ENTRY',

  // Business Logic
  INSUFFICIENT_STOCK = 'INSUFFICIENT_STOCK',
  APPOINTMENT_CONFLICT = 'APPOINTMENT_CONFLICT',
  NO_AVAILABLE_CAGE = 'NO_AVAILABLE_CAGE',

  // Sync
  SYNC_FAILED = 'SYNC_FAILED',
  CONFLICT_DETECTED = 'CONFLICT_DETECTED',

  // System
  DATABASE_ERROR = 'DATABASE_ERROR',
  SERVER_ERROR = 'SERVER_ERROR'
}
```

---

## 16. SECURITY IMPLEMENTATION

### 16.1 Password Security

```typescript
import bcrypt from 'bcrypt';

async function hashPassword(password: string): Promise<string> {
  return await bcrypt.hash(password, 12);
}

async function verifyPassword(
  password: string,
  hash: string
): Promise<boolean> {
  return await bcrypt.compare(password, hash);
}

async function createUser(
  email: string,
  password: string,
  clinicId: UUID
): Promise<void> {
  const hashedPassword = await hashPassword(password);

  await db.insert(local_users).values({
    email,
    password_hash: hashedPassword,
    clinic_id: clinicId,
    role: 'CUSTOMER',
    status: 'ACTIVE'
  });
}
```

### 16.2 Token Management

```typescript
const TOKEN_EXPIRY = 30 * 24 * 60 * 60 * 1000; // 30 days
const SESSION_INACTIVITY = 8 * 60 * 60 * 1000; // 8 hours

async function createSession(
  userId: UUID,
  deviceId: string
): Promise<string> {
  const token = crypto.randomUUID();
  const expiresAt = new Date(Date.now() + TOKEN_EXPIRY);

  await db.insert(local_sessions).values({
    user_id: userId,
    device_id: deviceId,
    token,
    expires_at: expiresAt
  });

  localStorage.setItem('token', token);
  return token;
}

async function validateSession(token: string): Promise<boolean> {
  const session = await db.select()
    .from(local_sessions)
    .where(eq(local_sessions.token, token));

  if (session.length === 0) return false;

  const now = new Date();
  if (now > session[0].expires_at) {
    // Token expired
    await db.delete(local_sessions)
      .where(eq(local_sessions.token, token));
    return false;
  }

  // Check inactivity
  const inactiveMs = now.getTime() - session[0].updated_at.getTime();
  if (inactiveMs > SESSION_INACTIVITY) {
    await db.delete(local_sessions)
      .where(eq(local_sessions.token, token));
    return false;
  }

  // Update last activity
  await db.update(local_sessions)
    .set({ updated_at: now })
    .where(eq(local_sessions.token, token));

  return true;
}
```

### 16.3 Data Encryption

```typescript
import crypto from 'crypto';

const ENCRYPTION_KEY = process.env.ENCRYPTION_KEY;

function encryptSensitiveData(data: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(
    'aes-256-cbc',
    Buffer.from(ENCRYPTION_KEY, 'hex'),
    iv
  );

  let encrypted = cipher.update(data);
  encrypted = Buffer.concat([encrypted, cipher.final()]);

  return `${iv.toString('hex')}:${encrypted.toString('hex')}`;
}

function decryptSensitiveData(encrypted: string): string {
  const [ivHex, encryptedHex] = encrypted.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const decipher = crypto.createDecipheriv(
    'aes-256-cbc',
    Buffer.from(ENCRYPTION_KEY, 'hex'),
    iv
  );

  let decrypted = decipher.update(Buffer.from(encryptedHex, 'hex'));
  decrypted = Buffer.concat([decrypted, decipher.final()]);

  return decrypted.toString();
}
```

---

## 17. TESTING REQUIREMENTS

### 17.1 Unit Tests

```typescript
// ✓ Each util function
describe('Password Hashing', () => {
  it('should hash password correctly', async () => {
    const password = 'TestPassword123!';
    const hash = await hashPassword(password);
    const isValid = await verifyPassword(password, hash);
    expect(isValid).toBe(true);
  });

  it('should reject wrong password', async () => {
    const password = 'TestPassword123!';
    const hash = await hashPassword(password);
    const isValid = await verifyPassword('WrongPassword', hash);
    expect(isValid).toBe(false);
  });
});
```

### 17.2 Integration Tests

```typescript
// ✓ Database transactions
describe('Inventory Deduction', () => {
  it('should prevent negative stock', async () => {
    await createItem({ quantity: 5 });
    
    expect(async () => {
      await deductInventory(itemId, 10);
    }).rejects.toThrow('INSUFFICIENT_STOCK');
  });

  it('should handle concurrent deductions', async () => {
    await createItem({ quantity: 10 });
    
    const results = await Promise.allSettled([
      deductInventory(itemId, 6),
      deductInventory(itemId, 6)
    ]);

    expect(results[0].status).toBe('fulfilled');
    expect(results[1].status).toBe('rejected');
  });
});
```

### 17.3 Offline Mode Tests

```typescript
// ✓ Operations without internet
describe('Offline Mode', () => {
  beforeEach(() => {
    disableNetwork();
  });

  it('should save medical record offline', async () => {
    const record = await createMedicalRecord(data);
    expect(record.id).toBeDefined();
  });

  it('should add to sync queue', async () => {
    await createMedicalRecord(data);
    const pending = await getPendingSyncItems();
    expect(pending.length).toBeGreaterThan(0);
  });

  it('should sync when online', async () => {
    await createMedicalRecord(data);
    enableNetwork();
    await waitForSync();
    
    const synced = await getPendingSyncItems();
    expect(synced.length).toBe(0);
  });
});
```

---

## 18. DEPLOYMENT CHECKLIST

### 18.1 Pre-Production

- [ ] All unit tests passing
- [ ] All integration tests passing
- [ ] Offline mode tested thoroughly
- [ ] Sync conflict scenarios tested
- [ ] Inventory double-deduction tested
- [ ] Audit logs functional
- [ ] Password hashing tested
- [ ] Session management tested
- [ ] Role-based access tested
- [ ] Database transactions tested
- [ ] Backup/restore functional
- [ ] Performance targets met

### 18.2 Production Deployment

- [ ] Environment variables configured
- [ ] Database migrations applied
- [ ] Encryption keys secured
- [ ] Backup automated
- [ ] Monitoring enabled
- [ ] Error tracking configured
- [ ] Rate limiting enabled
- [ ] CORS configured
- [ ] SSL/TLS enabled
- [ ] Database replicated to Supabase

### 18.3 Post-Deployment

- [ ] Monitor sync queue
- [ ] Monitor error logs
- [ ] Check backup status
- [ ] Verify audit logs
- [ ] Performance monitoring
- [ ] User feedback collection

---

## 19. MAINTENANCE & MONITORING

### 19.1 Daily Checks

```
- Backup status ✓
- Sync queue empty ✓
- No critical errors ✓
- Database size ✓
- User sessions active ✓
```

### 19.2 Weekly Tasks

```
- Clean up old sessions
- Archive old audit logs
- Review sync conflicts
- Check inventory accuracy
- Verify invoice records
```

### 19.3 Monthly Tasks

```
- Database optimization
- Full backup verification
- Disaster recovery test
- Security audit
- User role review
```

---

## 20. DOCUMENTATION STRUCTURE

```
docs/
├── BLUEPRINT.md (THIS FILE)
├── DATABASE-SCHEMA.md
├── WORKFLOW.md
├── SYNC-ENGINE.md
├── RLS-POLICIES.md
├── API-CONTRACT.md
├── DEPLOYMENT.md
└── TROUBLESHOOTING.md
```

---

**Dokumen ini adalah SUMBER KEBENARAN MUTLAK (Single Source of Truth) untuk seluruh pengembangan Haland PetCare v2.1. Setiap perubahan arsitektur, database, atau workflow WAJIB diperbarui di sini terlebih dahulu sebelum implementasi.**

**Last Updated: 2026-06-23**
