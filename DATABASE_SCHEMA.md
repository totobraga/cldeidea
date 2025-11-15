# Clinical Management System - Database Schema

## Database Design Principles
- Normalized to 3NF (Third Normal Form)
- Soft deletes (deleted_at) for audit trail
- Timestamps (created_at, updated_at) on all tables
- UUID or auto-increment IDs
- Foreign key constraints with appropriate ON DELETE actions
- Indexes on frequently queried fields

---

## CORE TABLES

### 1. users
User accounts for staff members

```sql
CREATE TABLE users (
    id                  BIGSERIAL PRIMARY KEY,
    uuid                UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    email               VARCHAR(255) UNIQUE NOT NULL,
    password_hash       VARCHAR(255) NOT NULL,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    role                VARCHAR(50) NOT NULL, -- admin, doctor, nurse, receptionist, billing
    medical_license     VARCHAR(100), -- Matrícula profesional
    specialization      VARCHAR(100),
    phone               VARCHAR(50),
    is_active           BOOLEAN DEFAULT true,
    last_login_at       TIMESTAMP,
    two_factor_enabled  BOOLEAN DEFAULT false,
    two_factor_secret   VARCHAR(255),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_users_email (email),
    INDEX idx_users_role (role),
    INDEX idx_users_active (is_active)
);
```

### 2. patients
Patient demographic and contact information

```sql
CREATE TABLE patients (
    id                      BIGSERIAL PRIMARY KEY,
    uuid                    UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    patient_number          VARCHAR(50) UNIQUE, -- Auto-generated patient ID

    -- Personal Information
    first_name              VARCHAR(100) NOT NULL,
    last_name               VARCHAR(100) NOT NULL,
    dni                     VARCHAR(20), -- Documento Nacional de Identidad
    passport                VARCHAR(50),
    date_of_birth           DATE NOT NULL,
    gender                  VARCHAR(20), -- male, female, other, prefer_not_to_say
    blood_type              VARCHAR(5), -- A+, A-, B+, B-, AB+, AB-, O+, O-
    marital_status          VARCHAR(50),
    occupation              VARCHAR(100),

    -- Contact Information
    email                   VARCHAR(255),
    phone_primary           VARCHAR(50),
    phone_secondary         VARCHAR(50),
    address_street          VARCHAR(255),
    address_city            VARCHAR(100),
    address_state           VARCHAR(100),
    address_postal_code     VARCHAR(20),
    address_country         VARCHAR(100) DEFAULT 'Argentina',

    -- Emergency Contact
    emergency_contact_name  VARCHAR(200),
    emergency_contact_phone VARCHAR(50),
    emergency_contact_relationship VARCHAR(100),

    -- System
    photo_url               VARCHAR(500),
    notes                   TEXT,
    is_active               BOOLEAN DEFAULT true,
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at              TIMESTAMP,

    INDEX idx_patients_dni (dni),
    INDEX idx_patients_name (last_name, first_name),
    INDEX idx_patients_phone (phone_primary),
    INDEX idx_patients_number (patient_number),
    INDEX idx_patients_active (is_active)
);
```

### 3. patient_insurance
Patient health coverage (Obra Social, Prepaga)

```sql
CREATE TABLE patient_insurance (
    id                  BIGSERIAL PRIMARY KEY,
    patient_id          BIGINT NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
    insurance_id        BIGINT NOT NULL REFERENCES insurances(id),
    member_number       VARCHAR(100),
    plan_name           VARCHAR(200),
    coverage_percentage DECIMAL(5,2), -- e.g., 100.00, 80.00
    copay_amount        DECIMAL(10,2),
    is_primary          BOOLEAN DEFAULT false,
    valid_from          DATE,
    valid_until         DATE,
    notes               TEXT,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_patient_insurance_patient (patient_id),
    INDEX idx_patient_insurance_insurance (insurance_id)
);
```

### 4. insurances
Obra Social and Prepaga catalog

```sql
CREATE TABLE insurances (
    id                  BIGSERIAL PRIMARY KEY,
    name                VARCHAR(200) NOT NULL,
    type                VARCHAR(50), -- obra_social, prepaga, pami, particular
    code                VARCHAR(50) UNIQUE,
    contact_phone       VARCHAR(50),
    contact_email       VARCHAR(255),
    website             VARCHAR(500),
    billing_address     TEXT,
    requires_authorization BOOLEAN DEFAULT false,
    nomenclator_type    VARCHAR(100), -- Type of pricing nomenclator
    notes               TEXT,
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_insurances_name (name),
    INDEX idx_insurances_code (code),
    INDEX idx_insurances_type (type)
);
```

### 5. patient_medical_history
Medical background information

```sql
CREATE TABLE patient_medical_history (
    id                  BIGSERIAL PRIMARY KEY,
    patient_id          BIGINT NOT NULL REFERENCES patients(id) ON DELETE CASCADE,

    -- Allergies
    allergies           JSONB, -- [{type: 'medication', name: 'penicillin', severity: 'high', notes: ''}]

    -- Chronic Conditions
    chronic_conditions  JSONB, -- [{condition: 'diabetes', diagnosed_date: '2020-01-01', notes: ''}]

    -- Previous Surgeries
    surgeries           JSONB, -- [{procedure: 'appendectomy', date: '2015-06-01', hospital: '', notes: ''}]

    -- Family History
    family_history      JSONB, -- [{relation: 'father', condition: 'hypertension', age_diagnosed: 45}]

    -- Immunizations
    immunizations       JSONB, -- [{vaccine: 'COVID-19', date: '2021-06-01', lot_number: '', notes: ''}]

    -- Risk Factors
    smoking_status      VARCHAR(50), -- never, former, current
    alcohol_consumption VARCHAR(50), -- never, occasional, moderate, heavy
    drug_use            TEXT,

    -- Other
    notes               TEXT,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_patient_history_patient (patient_id)
);
```

---

## APPOINTMENT TABLES

### 6. appointments
Scheduled patient appointments

```sql
CREATE TABLE appointments (
    id                  BIGSERIAL PRIMARY KEY,
    uuid                UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    patient_id          BIGINT NOT NULL REFERENCES patients(id),
    provider_id         BIGINT NOT NULL REFERENCES users(id),
    appointment_type_id BIGINT REFERENCES appointment_types(id),

    -- Scheduling
    scheduled_at        TIMESTAMP NOT NULL,
    duration_minutes    INTEGER NOT NULL DEFAULT 30,

    -- Status
    status              VARCHAR(50) NOT NULL DEFAULT 'scheduled',
                        -- scheduled, confirmed, arrived, in_progress, completed, cancelled, no_show, rescheduled

    -- Details
    chief_complaint     TEXT,
    notes               TEXT,
    cancellation_reason TEXT,

    -- Check-in/out
    checked_in_at       TIMESTAMP,
    checked_out_at      TIMESTAMP,

    -- Reminders
    reminder_sent_at    TIMESTAMP,
    confirmation_sent_at TIMESTAMP,

    -- Recurrence
    is_recurring        BOOLEAN DEFAULT false,
    recurrence_pattern  VARCHAR(50), -- weekly, monthly
    recurrence_end_date DATE,
    parent_appointment_id BIGINT REFERENCES appointments(id),

    created_by          BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_appointments_patient (patient_id),
    INDEX idx_appointments_provider (provider_id),
    INDEX idx_appointments_datetime (scheduled_at),
    INDEX idx_appointments_status (status),
    INDEX idx_appointments_date (DATE(scheduled_at))
);
```

### 7. appointment_types
Types of appointments with duration and pricing

```sql
CREATE TABLE appointment_types (
    id                  BIGSERIAL PRIMARY KEY,
    name                VARCHAR(100) NOT NULL, -- Consulta nueva, Control, Procedimiento
    description         TEXT,
    duration_minutes    INTEGER NOT NULL DEFAULT 30,
    color               VARCHAR(20), -- For calendar display
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 8. provider_schedules
Doctor working hours and availability

```sql
CREATE TABLE provider_schedules (
    id                  BIGSERIAL PRIMARY KEY,
    provider_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    day_of_week         INTEGER NOT NULL, -- 0=Sunday, 1=Monday, ..., 6=Saturday
    start_time          TIME NOT NULL,
    end_time            TIME NOT NULL,
    location            VARCHAR(200),
    is_active           BOOLEAN DEFAULT true,
    effective_from      DATE,
    effective_until     DATE,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_provider_schedules_provider (provider_id),
    INDEX idx_provider_schedules_day (day_of_week)
);
```

### 9. provider_time_off
Doctor vacations and blocked time

```sql
CREATE TABLE provider_time_off (
    id                  BIGSERIAL PRIMARY KEY,
    provider_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    start_datetime      TIMESTAMP NOT NULL,
    end_datetime        TIMESTAMP NOT NULL,
    reason              VARCHAR(200),
    notes               TEXT,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_provider_timeoff_provider (provider_id),
    INDEX idx_provider_timeoff_dates (start_datetime, end_datetime)
);
```

---

## CLINICAL TABLES

### 10. encounters
Clinical consultations

```sql
CREATE TABLE encounters (
    id                      BIGSERIAL PRIMARY KEY,
    uuid                    UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    provider_id             BIGINT NOT NULL REFERENCES users(id),
    appointment_id          BIGINT REFERENCES appointments(id),

    -- Encounter Info
    encounter_date          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    encounter_type          VARCHAR(50), -- outpatient, emergency, follow_up

    -- SOAP Notes
    subjective              TEXT, -- Motivo de consulta, enfermedad actual
    objective               TEXT, -- Examen físico
    assessment              TEXT, -- Impresión diagnóstica
    plan                    TEXT, -- Plan de tratamiento

    -- Additional
    chief_complaint         TEXT,
    history_present_illness TEXT,
    review_of_systems       TEXT,
    physical_examination    TEXT,
    clinical_impression     TEXT,

    -- Status
    status                  VARCHAR(50) DEFAULT 'draft', -- draft, finalized, locked
    finalized_at            TIMESTAMP,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at              TIMESTAMP,

    INDEX idx_encounters_patient (patient_id),
    INDEX idx_encounters_provider (provider_id),
    INDEX idx_encounters_date (encounter_date),
    INDEX idx_encounters_appointment (appointment_id)
);
```

### 11. vital_signs
Patient vital signs measurements

```sql
CREATE TABLE vital_signs (
    id                  BIGSERIAL PRIMARY KEY,
    encounter_id        BIGINT REFERENCES encounters(id) ON DELETE CASCADE,
    patient_id          BIGINT NOT NULL REFERENCES patients(id),
    measured_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- Vitals
    systolic_bp         INTEGER, -- mmHg
    diastolic_bp        INTEGER, -- mmHg
    heart_rate          INTEGER, -- bpm
    respiratory_rate    INTEGER, -- breaths/min
    temperature         DECIMAL(4,2), -- Celsius
    oxygen_saturation   INTEGER, -- SpO2 percentage
    weight              DECIMAL(6,2), -- kg
    height              DECIMAL(5,2), -- cm
    bmi                 DECIMAL(5,2), -- calculated

    -- Additional
    pain_scale          INTEGER, -- 0-10
    notes               TEXT,

    measured_by         BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_vital_signs_patient (patient_id),
    INDEX idx_vital_signs_encounter (encounter_id),
    INDEX idx_vital_signs_date (measured_at)
);
```

### 12. diagnoses
Patient diagnoses

```sql
CREATE TABLE diagnoses (
    id                  BIGSERIAL PRIMARY KEY,
    patient_id          BIGINT NOT NULL REFERENCES patients(id),
    encounter_id        BIGINT REFERENCES encounters(id),
    provider_id         BIGINT NOT NULL REFERENCES users(id),

    -- Diagnosis Info
    icd10_code          VARCHAR(20),
    diagnosis_name      VARCHAR(500) NOT NULL,
    diagnosis_type      VARCHAR(50), -- primary, secondary

    -- Status
    status              VARCHAR(50) DEFAULT 'active', -- active, resolved, chronic
    diagnosed_date      DATE NOT NULL,
    resolved_date       DATE,

    -- Additional
    severity            VARCHAR(50), -- mild, moderate, severe
    notes               TEXT,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_diagnoses_patient (patient_id),
    INDEX idx_diagnoses_encounter (encounter_id),
    INDEX idx_diagnoses_icd10 (icd10_code),
    INDEX idx_diagnoses_status (status)
);
```

---

## PRESCRIPTION TABLES

### 13. prescriptions
Medication prescriptions

```sql
CREATE TABLE prescriptions (
    id                  BIGSERIAL PRIMARY KEY,
    uuid                UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    patient_id          BIGINT NOT NULL REFERENCES patients(id),
    encounter_id        BIGINT REFERENCES encounters(id),
    provider_id         BIGINT NOT NULL REFERENCES users(id),

    -- Prescription Info
    prescription_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    prescription_number VARCHAR(50) UNIQUE,

    -- Status
    status              VARCHAR(50) DEFAULT 'active', -- active, dispensed, cancelled, expired

    -- Additional
    notes               TEXT,
    is_controlled       BOOLEAN DEFAULT false, -- Psychotropic/controlled substance

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_prescriptions_patient (patient_id),
    INDEX idx_prescriptions_provider (provider_id),
    INDEX idx_prescriptions_date (prescription_date)
);
```

### 14. prescription_items
Individual medications on a prescription

```sql
CREATE TABLE prescription_items (
    id                  BIGSERIAL PRIMARY KEY,
    prescription_id     BIGINT NOT NULL REFERENCES prescriptions(id) ON DELETE CASCADE,
    medication_id       BIGINT REFERENCES medications(id),

    -- Medication Details (if custom, not from catalog)
    medication_name     VARCHAR(500) NOT NULL,
    generic_name        VARCHAR(500),

    -- Dosage
    dosage              VARCHAR(200) NOT NULL, -- e.g., "500mg"
    form                VARCHAR(100), -- tablet, capsule, liquid, injection
    route               VARCHAR(100), -- oral, topical, intramuscular
    frequency           VARCHAR(200) NOT NULL, -- e.g., "cada 8 horas", "3 veces al día"
    duration            VARCHAR(200), -- e.g., "7 días", "continuo"

    -- Quantity
    quantity            INTEGER,
    refills             INTEGER DEFAULT 0,

    -- Instructions
    instructions        TEXT, -- e.g., "Tomar con alimentos"

    -- Status
    is_chronic          BOOLEAN DEFAULT false,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_prescription_items_prescription (prescription_id),
    INDEX idx_prescription_items_medication (medication_id)
);
```

### 15. medications
Medication catalog

```sql
CREATE TABLE medications (
    id                  BIGSERIAL PRIMARY KEY,
    name                VARCHAR(500) NOT NULL,
    generic_name        VARCHAR(500),
    brand_names         JSONB, -- Array of brand names
    drug_class          VARCHAR(200),
    form                VARCHAR(100), -- tablet, capsule, liquid
    strength            VARCHAR(100), -- e.g., "500mg", "10mg/ml"

    -- Safety
    interactions        JSONB, -- Array of drug interaction warnings
    contraindications   TEXT,
    side_effects        TEXT,

    -- Classification
    is_controlled       BOOLEAN DEFAULT false,
    requires_prescription BOOLEAN DEFAULT true,

    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_medications_name (name),
    INDEX idx_medications_generic (generic_name)
);
```

---

## LABORATORY TABLES

### 16. lab_orders
Laboratory test orders

```sql
CREATE TABLE lab_orders (
    id                  BIGSERIAL PRIMARY KEY,
    uuid                UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    patient_id          BIGINT NOT NULL REFERENCES patients(id),
    encounter_id        BIGINT REFERENCES encounters(id),
    provider_id         BIGINT NOT NULL REFERENCES users(id),

    -- Order Info
    order_date          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    order_number        VARCHAR(50) UNIQUE,

    -- Status
    status              VARCHAR(50) DEFAULT 'ordered',
                        -- ordered, collected, processing, completed, cancelled

    -- Urgency
    is_urgent           BOOLEAN DEFAULT false,

    -- Instructions
    fasting_required    BOOLEAN DEFAULT false,
    special_instructions TEXT,

    -- Results
    results_date        TIMESTAMP,
    results_notes       TEXT,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_lab_orders_patient (patient_id),
    INDEX idx_lab_orders_provider (provider_id),
    INDEX idx_lab_orders_status (status),
    INDEX idx_lab_orders_date (order_date)
);
```

### 17. lab_order_items
Individual tests on a lab order

```sql
CREATE TABLE lab_order_items (
    id                  BIGSERIAL PRIMARY KEY,
    lab_order_id        BIGINT NOT NULL REFERENCES lab_orders(id) ON DELETE CASCADE,
    lab_test_id         BIGINT REFERENCES lab_tests(id),
    test_name           VARCHAR(200) NOT NULL,
    test_code           VARCHAR(50),
    status              VARCHAR(50) DEFAULT 'ordered',
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_lab_order_items_order (lab_order_id),
    INDEX idx_lab_order_items_test (lab_test_id)
);
```

### 18. lab_tests
Catalog of available lab tests

```sql
CREATE TABLE lab_tests (
    id                  BIGSERIAL PRIMARY KEY,
    name                VARCHAR(200) NOT NULL,
    code                VARCHAR(50) UNIQUE,
    category            VARCHAR(100), -- hematology, chemistry, microbiology
    sample_type         VARCHAR(100), -- blood, urine, tissue
    turnaround_time     VARCHAR(100), -- Expected time for results
    preparation         TEXT, -- Fasting, special prep
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_lab_tests_name (name),
    INDEX idx_lab_tests_code (code),
    INDEX idx_lab_tests_category (category)
);
```

### 19. lab_results
Individual test results

```sql
CREATE TABLE lab_results (
    id                  BIGSERIAL PRIMARY KEY,
    lab_order_id        BIGINT NOT NULL REFERENCES lab_orders(id),
    lab_order_item_id   BIGINT REFERENCES lab_order_items(id),
    lab_test_id         BIGINT REFERENCES lab_tests(id),

    -- Result Info
    test_name           VARCHAR(200) NOT NULL,
    result_value        VARCHAR(500),
    result_unit         VARCHAR(50),
    reference_range     VARCHAR(200),

    -- Flags
    is_abnormal         BOOLEAN DEFAULT false,
    abnormal_flag       VARCHAR(50), -- high, low, critical

    -- Additional
    result_date         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    notes               TEXT,

    -- Entry
    entered_by          BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_lab_results_order (lab_order_id),
    INDEX idx_lab_results_test (lab_test_id),
    INDEX idx_lab_results_date (result_date)
);
```

### 20. imaging_orders
Diagnostic imaging orders (X-ray, CT, MRI, etc.)

```sql
CREATE TABLE imaging_orders (
    id                  BIGSERIAL PRIMARY KEY,
    uuid                UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    patient_id          BIGINT NOT NULL REFERENCES patients(id),
    encounter_id        BIGINT REFERENCES encounters(id),
    provider_id         BIGINT NOT NULL REFERENCES users(id),

    -- Order Info
    order_date          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    order_number        VARCHAR(50) UNIQUE,

    -- Study Type
    study_type          VARCHAR(100) NOT NULL, -- x_ray, ct, mri, ultrasound, mammography
    body_part           VARCHAR(200),
    laterality          VARCHAR(50), -- left, right, bilateral

    -- Status
    status              VARCHAR(50) DEFAULT 'ordered', -- ordered, scheduled, completed, cancelled

    -- Clinical
    clinical_indication TEXT,
    urgent              BOOLEAN DEFAULT false,

    -- Results
    study_date          TIMESTAMP,
    report              TEXT,
    radiologist_id      BIGINT REFERENCES users(id),

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_imaging_orders_patient (patient_id),
    INDEX idx_imaging_orders_provider (provider_id),
    INDEX idx_imaging_orders_status (status)
);
```

---

## BILLING TABLES

### 21. invoices
Patient invoices (AFIP compliant)

```sql
CREATE TABLE invoices (
    id                  BIGSERIAL PRIMARY KEY,
    uuid                UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    patient_id          BIGINT NOT NULL REFERENCES patients(id),
    encounter_id        BIGINT REFERENCES encounters(id),

    -- Invoice Info
    invoice_number      VARCHAR(50) UNIQUE NOT NULL,
    invoice_type        VARCHAR(10) NOT NULL, -- A, B, C
    invoice_date        DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date            DATE,

    -- AFIP
    cae                 VARCHAR(50), -- Código de Autorización Electrónico
    cae_expiration      DATE,
    punto_venta         INTEGER, -- Point of sale

    -- Amounts
    subtotal            DECIMAL(12,2) NOT NULL DEFAULT 0,
    tax_amount          DECIMAL(12,2) NOT NULL DEFAULT 0, -- IVA
    discount_amount     DECIMAL(12,2) NOT NULL DEFAULT 0,
    total_amount        DECIMAL(12,2) NOT NULL DEFAULT 0,

    -- Payment
    amount_paid         DECIMAL(12,2) NOT NULL DEFAULT 0,
    balance_due         DECIMAL(12,2) NOT NULL DEFAULT 0,

    -- Insurance
    insurance_id        BIGINT REFERENCES insurances(id),
    insurance_claim_amount DECIMAL(12,2),
    patient_responsibility DECIMAL(12,2),

    -- Status
    status              VARCHAR(50) DEFAULT 'draft', -- draft, sent, paid, partially_paid, cancelled

    -- Additional
    notes               TEXT,

    created_by          BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_invoices_patient (patient_id),
    INDEX idx_invoices_number (invoice_number),
    INDEX idx_invoices_date (invoice_date),
    INDEX idx_invoices_status (status),
    INDEX idx_invoices_cae (cae)
);
```

### 22. invoice_items
Line items on an invoice

```sql
CREATE TABLE invoice_items (
    id                  BIGSERIAL PRIMARY KEY,
    invoice_id          BIGINT NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    service_id          BIGINT REFERENCES services(id),

    -- Item Info
    description         VARCHAR(500) NOT NULL,
    service_code        VARCHAR(50),
    quantity            INTEGER NOT NULL DEFAULT 1,
    unit_price          DECIMAL(12,2) NOT NULL,
    discount_percent    DECIMAL(5,2) DEFAULT 0,
    tax_rate            DECIMAL(5,2) DEFAULT 0, -- IVA rate

    -- Calculated
    subtotal            DECIMAL(12,2) NOT NULL,
    tax_amount          DECIMAL(12,2) NOT NULL,
    total               DECIMAL(12,2) NOT NULL,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_invoice_items_invoice (invoice_id),
    INDEX idx_invoice_items_service (service_id)
);
```

### 23. services
Catalog of billable services

```sql
CREATE TABLE services (
    id                  BIGSERIAL PRIMARY KEY,
    name                VARCHAR(200) NOT NULL,
    code                VARCHAR(50) UNIQUE, -- Nomenclator code
    description         TEXT,
    category            VARCHAR(100),

    -- Default pricing
    default_price       DECIMAL(12,2),
    cost                DECIMAL(12,2),

    -- Tax
    taxable             BOOLEAN DEFAULT true,
    tax_rate            DECIMAL(5,2) DEFAULT 21.00, -- IVA 21%

    -- Billing
    duration_minutes    INTEGER,

    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_services_name (name),
    INDEX idx_services_code (code),
    INDEX idx_services_category (category)
);
```

### 24. service_pricing
Insurance-specific pricing

```sql
CREATE TABLE service_pricing (
    id                  BIGSERIAL PRIMARY KEY,
    service_id          BIGINT NOT NULL REFERENCES services(id) ON DELETE CASCADE,
    insurance_id        BIGINT REFERENCES insurances(id),
    price               DECIMAL(12,2) NOT NULL,
    effective_from      DATE,
    effective_until     DATE,
    notes               TEXT,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_service_pricing_service (service_id),
    INDEX idx_service_pricing_insurance (insurance_id)
);
```

### 25. payments
Payment transactions

```sql
CREATE TABLE payments (
    id                  BIGSERIAL PRIMARY KEY,
    uuid                UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),
    invoice_id          BIGINT NOT NULL REFERENCES invoices(id),
    patient_id          BIGINT NOT NULL REFERENCES patients(id),

    -- Payment Info
    payment_date        DATE NOT NULL DEFAULT CURRENT_DATE,
    payment_number      VARCHAR(50) UNIQUE,
    amount              DECIMAL(12,2) NOT NULL,

    -- Payment Method
    payment_method      VARCHAR(50) NOT NULL, -- cash, card, bank_transfer, mercado_pago

    -- Card/Transfer Details
    card_type           VARCHAR(50), -- visa, mastercard, amex
    card_last_4         VARCHAR(4),
    transaction_id      VARCHAR(200), -- For electronic payments

    -- Status
    status              VARCHAR(50) DEFAULT 'completed', -- pending, completed, failed, refunded

    -- Additional
    notes               TEXT,

    received_by         BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_payments_invoice (invoice_id),
    INDEX idx_payments_patient (patient_id),
    INDEX idx_payments_date (payment_date),
    INDEX idx_payments_method (payment_method)
);
```

### 26. insurance_claims
Claims submitted to insurances

```sql
CREATE TABLE insurance_claims (
    id                  BIGSERIAL PRIMARY KEY,
    invoice_id          BIGINT NOT NULL REFERENCES invoices(id),
    insurance_id        BIGINT NOT NULL REFERENCES insurances(id),
    patient_id          BIGINT NOT NULL REFERENCES patients(id),

    -- Claim Info
    claim_number        VARCHAR(100) UNIQUE,
    claim_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    service_date        DATE NOT NULL,

    -- Amounts
    billed_amount       DECIMAL(12,2) NOT NULL,
    approved_amount     DECIMAL(12,2),
    paid_amount         DECIMAL(12,2),

    -- Status
    status              VARCHAR(50) DEFAULT 'submitted',
                        -- submitted, in_review, approved, partially_approved, denied, paid

    -- Prior Authorization
    authorization_number VARCHAR(100),

    -- Rejection
    denial_reason       TEXT,
    denial_code         VARCHAR(50),

    -- Payment
    payment_date        DATE,
    payment_reference   VARCHAR(200),

    -- Additional
    notes               TEXT,

    submitted_by        BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_claims_invoice (invoice_id),
    INDEX idx_claims_insurance (insurance_id),
    INDEX idx_claims_patient (patient_id),
    INDEX idx_claims_status (status),
    INDEX idx_claims_date (claim_date)
);
```

---

## INVENTORY TABLES

### 27. inventory_items
Inventory catalog

```sql
CREATE TABLE inventory_items (
    id                  BIGSERIAL PRIMARY KEY,
    name                VARCHAR(200) NOT NULL,
    description         TEXT,
    sku                 VARCHAR(100) UNIQUE,
    category            VARCHAR(100), -- medication, consumable, equipment, office

    -- Stock
    unit_of_measure     VARCHAR(50), -- piece, box, bottle, vial
    current_stock       INTEGER NOT NULL DEFAULT 0,
    minimum_stock       INTEGER DEFAULT 0,
    reorder_point       INTEGER,

    -- Pricing
    unit_cost           DECIMAL(12,2),
    unit_price          DECIMAL(12,2),

    -- Storage
    storage_location    VARCHAR(200),

    -- Tracking
    track_lot_numbers   BOOLEAN DEFAULT false,
    track_expiration    BOOLEAN DEFAULT false,

    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_inventory_name (name),
    INDEX idx_inventory_sku (sku),
    INDEX idx_inventory_category (category)
);
```

### 28. inventory_batches
Batch/lot tracking for inventory

```sql
CREATE TABLE inventory_batches (
    id                  BIGSERIAL PRIMARY KEY,
    inventory_item_id   BIGINT NOT NULL REFERENCES inventory_items(id),
    batch_number        VARCHAR(100) NOT NULL,
    lot_number          VARCHAR(100),
    expiration_date     DATE,
    quantity            INTEGER NOT NULL,
    quantity_remaining  INTEGER NOT NULL,
    unit_cost           DECIMAL(12,2),
    received_date       DATE NOT NULL,
    supplier_id         BIGINT REFERENCES suppliers(id),
    notes               TEXT,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_batches_item (inventory_item_id),
    INDEX idx_batches_expiration (expiration_date),
    INDEX idx_batches_lot (lot_number)
);
```

### 29. inventory_transactions
Stock movements

```sql
CREATE TABLE inventory_transactions (
    id                  BIGSERIAL PRIMARY KEY,
    inventory_item_id   BIGINT NOT NULL REFERENCES inventory_items(id),
    batch_id            BIGINT REFERENCES inventory_batches(id),

    -- Transaction
    transaction_type    VARCHAR(50) NOT NULL, -- in, out, adjustment, transfer, waste
    quantity            INTEGER NOT NULL,
    unit_cost           DECIMAL(12,2),

    -- Reference
    reference_type      VARCHAR(50), -- purchase_order, patient_use, adjustment
    reference_id        BIGINT,

    -- Details
    transaction_date    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    notes               TEXT,

    performed_by        BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_transactions_item (inventory_item_id),
    INDEX idx_transactions_type (transaction_type),
    INDEX idx_transactions_date (transaction_date)
);
```

### 30. suppliers
Supplier/vendor information

```sql
CREATE TABLE suppliers (
    id                  BIGSERIAL PRIMARY KEY,
    name                VARCHAR(200) NOT NULL,
    contact_person      VARCHAR(200),
    phone               VARCHAR(50),
    email               VARCHAR(255),
    address             TEXT,
    website             VARCHAR(500),
    tax_id              VARCHAR(50), -- CUIT
    notes               TEXT,
    is_active           BOOLEAN DEFAULT true,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_suppliers_name (name)
);
```

### 31. purchase_orders
Purchase orders for inventory

```sql
CREATE TABLE purchase_orders (
    id                  BIGSERIAL PRIMARY KEY,
    po_number           VARCHAR(50) UNIQUE NOT NULL,
    supplier_id         BIGINT NOT NULL REFERENCES suppliers(id),
    order_date          DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_delivery   DATE,
    status              VARCHAR(50) DEFAULT 'draft', -- draft, sent, received, cancelled
    subtotal            DECIMAL(12,2) NOT NULL DEFAULT 0,
    tax_amount          DECIMAL(12,2) NOT NULL DEFAULT 0,
    total_amount        DECIMAL(12,2) NOT NULL DEFAULT 0,
    notes               TEXT,
    created_by          BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_po_number (po_number),
    INDEX idx_po_supplier (supplier_id),
    INDEX idx_po_status (status)
);
```

### 32. purchase_order_items
Items on a purchase order

```sql
CREATE TABLE purchase_order_items (
    id                  BIGSERIAL PRIMARY KEY,
    purchase_order_id   BIGINT NOT NULL REFERENCES purchase_orders(id) ON DELETE CASCADE,
    inventory_item_id   BIGINT NOT NULL REFERENCES inventory_items(id),
    quantity_ordered    INTEGER NOT NULL,
    quantity_received   INTEGER DEFAULT 0,
    unit_cost           DECIMAL(12,2) NOT NULL,
    total_cost          DECIMAL(12,2) NOT NULL,
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_po_items_po (purchase_order_id),
    INDEX idx_po_items_item (inventory_item_id)
);
```

---

## COMMUNICATION TABLES

### 33. messages
Internal messaging between staff

```sql
CREATE TABLE messages (
    id                  BIGSERIAL PRIMARY KEY,
    from_user_id        BIGINT NOT NULL REFERENCES users(id),
    to_user_id          BIGINT NOT NULL REFERENCES users(id),
    subject             VARCHAR(500),
    body                TEXT NOT NULL,
    is_read             BOOLEAN DEFAULT false,
    read_at             TIMESTAMP,
    parent_message_id   BIGINT REFERENCES messages(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_messages_from (from_user_id),
    INDEX idx_messages_to (to_user_id),
    INDEX idx_messages_read (is_read)
);
```

### 34. notifications
System notifications for users

```sql
CREATE TABLE notifications (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             BIGINT NOT NULL REFERENCES users(id),
    type                VARCHAR(50) NOT NULL, -- appointment_reminder, lab_result, task
    title               VARCHAR(200) NOT NULL,
    message             TEXT,

    -- Reference
    reference_type      VARCHAR(50), -- appointment, lab_order, message
    reference_id        BIGINT,

    -- Status
    is_read             BOOLEAN DEFAULT false,
    read_at             TIMESTAMP,

    -- Delivery
    sent_via            VARCHAR(50), -- in_app, email, sms

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_notifications_user (user_id),
    INDEX idx_notifications_read (is_read),
    INDEX idx_notifications_type (type)
);
```

### 35. communication_logs
Log of outbound communications (SMS, email)

```sql
CREATE TABLE communication_logs (
    id                  BIGSERIAL PRIMARY KEY,
    patient_id          BIGINT REFERENCES patients(id),
    user_id             BIGINT REFERENCES users(id),

    -- Communication
    type                VARCHAR(50) NOT NULL, -- sms, email, whatsapp
    recipient           VARCHAR(255) NOT NULL, -- Phone or email
    subject             VARCHAR(500),
    message             TEXT NOT NULL,

    -- Reference
    reference_type      VARCHAR(50), -- appointment_reminder, lab_result
    reference_id        BIGINT,

    -- Status
    status              VARCHAR(50) DEFAULT 'pending', -- pending, sent, delivered, failed
    sent_at             TIMESTAMP,
    delivered_at        TIMESTAMP,
    error_message       TEXT,

    -- Provider
    provider            VARCHAR(50), -- twilio, sendgrid, whatsapp_business
    provider_message_id VARCHAR(200),

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_comm_logs_patient (patient_id),
    INDEX idx_comm_logs_type (type),
    INDEX idx_comm_logs_status (status),
    INDEX idx_comm_logs_sent (sent_at)
);
```

---

## DOCUMENT MANAGEMENT

### 36. documents
Uploaded documents and files

```sql
CREATE TABLE documents (
    id                  BIGSERIAL PRIMARY KEY,
    uuid                UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),

    -- Association
    patient_id          BIGINT REFERENCES patients(id),
    encounter_id        BIGINT REFERENCES encounters(id),

    -- Document Info
    title               VARCHAR(200) NOT NULL,
    description         TEXT,
    category            VARCHAR(100), -- consent, lab_result, imaging, prescription, id_copy

    -- File
    file_name           VARCHAR(500) NOT NULL,
    file_path           VARCHAR(1000) NOT NULL,
    file_size           BIGINT, -- bytes
    mime_type           VARCHAR(100),
    file_hash           VARCHAR(64), -- SHA-256 for verification

    -- Metadata
    document_date       DATE,

    uploaded_by         BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP,

    INDEX idx_documents_patient (patient_id),
    INDEX idx_documents_encounter (encounter_id),
    INDEX idx_documents_category (category)
);
```

---

## SYSTEM TABLES

### 37. audit_logs
Audit trail of system actions

```sql
CREATE TABLE audit_logs (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             BIGINT REFERENCES users(id),

    -- Action
    action              VARCHAR(100) NOT NULL, -- login, create, update, delete, view
    entity_type         VARCHAR(100) NOT NULL, -- patient, appointment, invoice
    entity_id           BIGINT,

    -- Details
    description         TEXT,
    ip_address          VARCHAR(50),
    user_agent          TEXT,

    -- Changes (for update/delete)
    old_values          JSONB,
    new_values          JSONB,

    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_audit_user (user_id),
    INDEX idx_audit_action (action),
    INDEX idx_audit_entity (entity_type, entity_id),
    INDEX idx_audit_date (created_at)
);
```

### 38. system_settings
Application configuration

```sql
CREATE TABLE system_settings (
    id                  BIGSERIAL PRIMARY KEY,
    setting_key         VARCHAR(100) UNIQUE NOT NULL,
    setting_value       TEXT,
    setting_type        VARCHAR(50), -- string, number, boolean, json
    description         TEXT,
    is_public           BOOLEAN DEFAULT false, -- Can be accessed by frontend
    updated_by          BIGINT REFERENCES users(id),
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_settings_key (setting_key)
);
```

---

## VIEWS (Useful Database Views)

### Patient Summary View
```sql
CREATE VIEW patient_summary AS
SELECT
    p.id,
    p.patient_number,
    p.first_name,
    p.last_name,
    p.dni,
    p.date_of_birth,
    EXTRACT(YEAR FROM AGE(p.date_of_birth)) as age,
    p.gender,
    p.phone_primary,
    p.email,
    COUNT(DISTINCT a.id) as total_appointments,
    COUNT(DISTINCT e.id) as total_encounters,
    MAX(e.encounter_date) as last_visit,
    pi.name as primary_insurance
FROM patients p
LEFT JOIN appointments a ON p.id = a.patient_id
LEFT JOIN encounters e ON p.id = e.patient_id
LEFT JOIN patient_insurance pains ON p.id = pains.patient_id AND pains.is_primary = true
LEFT JOIN insurances pi ON pains.insurance_id = pi.id
WHERE p.deleted_at IS NULL
GROUP BY p.id, pi.name;
```

### Provider Schedule View
```sql
CREATE VIEW provider_schedules_view AS
SELECT
    ps.id,
    u.id as provider_id,
    u.first_name || ' ' || u.last_name as provider_name,
    ps.day_of_week,
    CASE ps.day_of_week
        WHEN 0 THEN 'Domingo'
        WHEN 1 THEN 'Lunes'
        WHEN 2 THEN 'Martes'
        WHEN 3 THEN 'Miércoles'
        WHEN 4 THEN 'Jueves'
        WHEN 5 THEN 'Viernes'
        WHEN 6 THEN 'Sábado'
    END as day_name,
    ps.start_time,
    ps.end_time,
    ps.location
FROM provider_schedules ps
JOIN users u ON ps.provider_id = u.id
WHERE ps.is_active = true;
```

---

## INDEXES SUMMARY

Key indexes for performance:
- Patient search: `idx_patients_name`, `idx_patients_dni`, `idx_patients_phone`
- Appointment calendar: `idx_appointments_datetime`, `idx_appointments_provider`
- Billing: `idx_invoices_patient`, `idx_invoices_date`, `idx_invoices_cae`
- Clinical: `idx_encounters_patient`, `idx_encounters_date`
- Audit: `idx_audit_date`, `idx_audit_user`

---

*Database Schema Version: 1.0*
*Last Updated: 2025-11-15*
*RDBMS: PostgreSQL 14+*
