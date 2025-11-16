# Clinical Management System - Gap Analysis & Missing Features

## Overview

This document identifies features commonly found in modern clinical management systems (2025) that are currently missing or under-specified in our system, with special focus on Argentina-specific requirements.

---

## 1. CRITICAL MISSING FEATURES

### 1.1 Telemedicine / Virtual Consultations

**Status:** ⚠️ Mentioned in roadmap Phase 5 but NO detailed specification

**Why Critical:**
- Post-COVID essential feature
- Obra Social coverage for telehealth increasing in Argentina
- Patient expectation in 2025
- Reduces no-shows
- Expands service reach

**Required Specifications:**

```sql
-- Telemedicine schema
CREATE TABLE telemedicine_sessions (
    id                      BIGSERIAL PRIMARY KEY,
    appointment_id          BIGINT REFERENCES appointments(id),
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    provider_id             BIGINT NOT NULL REFERENCES users(id),

    -- Session details
    session_type            VARCHAR(50) NOT NULL, -- 'video', 'audio', 'chat'
    session_url             VARCHAR(500), -- Meeting room URL
    session_id              VARCHAR(200), -- External platform session ID

    -- Platform integration
    platform                VARCHAR(50), -- 'zoom', 'meet', 'whereby', 'custom'
    platform_meeting_id     VARCHAR(200),

    -- Scheduling
    scheduled_start         TIMESTAMP NOT NULL,
    scheduled_end           TIMESTAMP NOT NULL,
    actual_start            TIMESTAMP,
    actual_end              TIMESTAMP,
    duration_minutes        INTEGER,

    -- Technical requirements
    requires_waiting_room   BOOLEAN DEFAULT true,
    recording_enabled       BOOLEAN DEFAULT false,
    recording_url           VARCHAR(500),

    -- Status
    status                  VARCHAR(50) DEFAULT 'scheduled',
    -- 'scheduled', 'waiting_room', 'in_progress', 'completed', 'cancelled', 'no_show', 'technical_issue'

    -- Connection quality
    patient_connection_quality VARCHAR(50), -- 'excellent', 'good', 'poor', 'failed'
    provider_connection_quality VARCHAR(50),

    -- Billing
    is_billable             BOOLEAN DEFAULT true,
    obra_social_authorized  BOOLEAN DEFAULT false,
    authorization_number    VARCHAR(100),

    -- Documentation
    encounter_id            BIGINT REFERENCES encounters(id),
    clinical_notes          TEXT,
    prescriptions_issued    BIGINT[],

    -- Technical logs
    connection_log          JSONB, -- Technical details for troubleshooting

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_telemedicine_patient (patient_id),
    INDEX idx_telemedicine_provider (provider_id),
    INDEX idx_telemedicine_scheduled (scheduled_start),
    INDEX idx_telemedicine_status (status)
);

-- Telemedicine consent (required in Argentina)
CREATE TABLE telemedicine_consents (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),

    -- Consent details
    consent_text            TEXT NOT NULL,
    consented_at            TIMESTAMP NOT NULL,
    consent_ip_address      VARCHAR(45),

    -- Digital signature
    signature_data          TEXT,

    -- Expiration
    valid_until             DATE,
    is_active               BOOLEAN DEFAULT true,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_telemedicine_consent_patient (patient_id)
);
```

**Features Required:**
- ✅ Video consultation platform integration (Zoom, Google Meet, Whereby)
- ✅ HIPAA-equivalent compliant video (encryption, no recording without consent)
- ✅ Waiting room feature
- ✅ Screen sharing for viewing lab results/images
- ✅ Session recording (with consent) for compliance
- ✅ Obra Social authorization for telemedicine
- ✅ Mobile app support (iOS/Android)
- ✅ Connection quality monitoring
- ✅ Automatic documentation in EHR
- ✅ E-prescriptions during video visit
- ✅ Digital consent for telemedicine
- ✅ Payment collection for teleconsultations
- ✅ Integration with appointment system

**Argentina-Specific Requirements:**
- Telemedicine consent form (Ley de Telemedicina)
- Obra Social coverage verification for virtual visits
- Remote prescription validity (30 days for chronic conditions)
- Patient identification verification
- Geographic restrictions (some provinces have regulations)

---

### 1.2 Online Patient Self-Booking

**Status:** ⚠️ Patient portal exists but self-booking not detailed

**Why Critical:**
- 70% of patients prefer online booking (2025 data)
- Reduces receptionist workload
- 24/7 availability
- Reduces phone traffic
- Improves patient satisfaction

**Required Specifications:**

```sql
-- Online booking configuration
CREATE TABLE online_booking_rules (
    id                      BIGSERIAL PRIMARY KEY,
    provider_id             BIGINT REFERENCES users(id),
    appointment_type_id     BIGINT REFERENCES appointment_types(id),

    -- Availability for online booking
    allow_online_booking    BOOLEAN DEFAULT true,

    -- Restrictions
    min_advance_hours       INTEGER DEFAULT 2, -- Book at least 2 hours ahead
    max_advance_days        INTEGER DEFAULT 30, -- Book up to 30 days ahead

    -- Time slots
    allowed_days            INTEGER[], -- [1,2,3,4,5] for Mon-Fri
    allowed_time_start      TIME,
    allowed_time_end        TIME,

    -- Insurance restrictions
    allowed_insurances      BIGINT[], -- NULL = all insurances
    block_particular        BOOLEAN DEFAULT false,

    -- Limits
    max_bookings_per_patient_per_month INTEGER,

    -- Confirmation
    requires_admin_approval BOOLEAN DEFAULT false, -- Some appointments need approval
    auto_confirm            BOOLEAN DEFAULT true,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_online_booking_provider (provider_id)
);

-- Online booking attempts log
CREATE TABLE online_booking_attempts (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT REFERENCES patients(id),

    -- Attempt details
    provider_id             BIGINT REFERENCES users(id),
    appointment_type_id     BIGINT REFERENCES appointment_types(id),
    requested_date          DATE,
    requested_time          TIME,

    -- Result
    status                  VARCHAR(50), -- 'success', 'failed', 'slots_full', 'restricted'
    failure_reason          TEXT,

    -- Created appointment
    appointment_id          BIGINT REFERENCES appointments(id),

    attempted_at            TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_booking_attempts_patient (patient_id),
    INDEX idx_booking_attempts_date (attempted_at)
);
```

**Features Required:**
- ✅ Real-time availability display
- ✅ Provider selection (with photos and specialties)
- ✅ Appointment type selection
- ✅ Obra Social validation
- ✅ Conflict detection (patient already has appointment)
- ✅ Confirmation email/SMS
- ✅ Cancellation/rescheduling by patient
- ✅ Google Calendar integration
- ✅ Reminder about required documents
- ✅ Pre-visit forms
- ✅ Payment collection (if required)

---

### 1.3 Medication Reconciliation

**Status:** ❌ NOT SPECIFIED

**Why Critical:**
- Patient safety (prevent drug interactions)
- Required for quality metrics
- Important for chronic disease management
- Standard of care in 2025

**Required Specifications:**

```sql
CREATE TABLE medication_reconciliation (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    encounter_id            BIGINT REFERENCES encounters(id),

    -- Reconciliation type
    reconciliation_type     VARCHAR(50) NOT NULL,
    -- 'admission', 'discharge', 'transfer', 'routine_visit'

    -- Medications reviewed
    medications_reviewed    JSONB NOT NULL,
    /* Example:
    {
      "current_medications": [
        {
          "medication_id": 123,
          "medication_name": "Enalapril 10mg",
          "dosage": "10mg",
          "frequency": "Once daily",
          "status": "continued",
          "source": "previous_prescription"
        }
      ],
      "new_medications": [...],
      "discontinued_medications": [...],
      "changed_medications": [...]
    }
    */

    -- Discrepancies found
    discrepancies           JSONB,
    discrepancies_resolved  BOOLEAN DEFAULT false,
    resolution_notes        TEXT,

    -- Who performed reconciliation
    performed_by            BIGINT REFERENCES users(id),
    reviewed_by             BIGINT REFERENCES users(id), -- Physician review

    -- Status
    status                  VARCHAR(50) DEFAULT 'pending',
    -- 'pending', 'in_progress', 'completed', 'reviewed'

    completed_at            TIMESTAMP,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_med_recon_patient (patient_id),
    INDEX idx_med_recon_encounter (encounter_id),
    INDEX idx_med_recon_status (status)
);
```

**Features Required:**
- ✅ Compare current medications with previous prescriptions
- ✅ Identify discrepancies (patient not taking prescribed meds)
- ✅ Import from patient-reported medications
- ✅ Drug interaction checking across all medications
- ✅ Allergy checking
- ✅ Duplicate therapy detection
- ✅ Generate reconciliation report
- ✅ Physician review workflow

---

### 1.4 Immunization Registry & Tracking

**Status:** ⚠️ Mentioned in patient records but not detailed

**Why Critical:**
- Required for pediatric care
- Government reporting requirements (Argentina)
- COVID-19 vaccination tracking
- Obra Social/PAMI requirements
- Public health monitoring

**Required Specifications:**

```sql
CREATE TABLE immunizations (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),

    -- Vaccine details
    vaccine_code            VARCHAR(50) NOT NULL, -- CVX code or local
    vaccine_name            VARCHAR(200) NOT NULL,
    vaccine_type            VARCHAR(100), -- 'covid', 'flu', 'hepatitis_b', etc.

    -- Administration
    administered_date       DATE NOT NULL,
    administered_by         BIGINT REFERENCES users(id),
    administration_site     VARCHAR(100), -- 'left_arm', 'right_arm', 'thigh'
    route                   VARCHAR(50), -- 'intramuscular', 'subcutaneous', 'oral'
    dose_number             INTEGER, -- 1st dose, 2nd dose, booster
    dose_amount             VARCHAR(50),

    -- Manufacturer info
    manufacturer            VARCHAR(200),
    lot_number              VARCHAR(100),
    expiration_date         DATE,

    -- Location
    location_id             BIGINT REFERENCES locations(id),

    -- Funding source (important for Argentina)
    funding_source          VARCHAR(100), -- 'public', 'private', 'obra_social'
    obra_social_id          BIGINT REFERENCES insurances(id),

    -- Adverse reactions
    adverse_reaction        BOOLEAN DEFAULT false,
    adverse_reaction_details TEXT,
    reaction_severity       VARCHAR(50), -- 'mild', 'moderate', 'severe'

    -- Government reporting (Argentina)
    reported_to_nomivac     BOOLEAN DEFAULT false, -- NOMIVAC = Nominal Vaccination Registry
    nomivac_report_date     DATE,
    nomivac_transaction_id  VARCHAR(200),

    -- Status
    status                  VARCHAR(50) DEFAULT 'completed',
    -- 'scheduled', 'completed', 'missed', 'contraindicated'

    -- Documentation
    encounter_id            BIGINT REFERENCES encounters(id),
    consent_document_id     BIGINT REFERENCES documents(id),

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_immunizations_patient (patient_id),
    INDEX idx_immunizations_vaccine (vaccine_code),
    INDEX idx_immunizations_date (administered_date)
);

-- Immunization schedule (Argentina calendar)
CREATE TABLE immunization_schedule (
    id                      BIGSERIAL PRIMARY KEY,
    vaccine_code            VARCHAR(50) NOT NULL,
    vaccine_name            VARCHAR(200) NOT NULL,

    -- Age requirements
    age_min_months          INTEGER, -- Minimum age in months
    age_max_months          INTEGER, -- Maximum age (NULL if no max)

    -- Dose schedule
    dose_number             INTEGER,
    recommended_interval_days INTEGER, -- Days after previous dose

    -- Priority
    is_mandatory            BOOLEAN DEFAULT false, -- Obligatory in Argentina
    is_optional             BOOLEAN DEFAULT false,

    -- Special populations
    for_pediatric           BOOLEAN DEFAULT false,
    for_adult               BOOLEAN DEFAULT false,
    for_elderly             BOOLEAN DEFAULT false,
    for_pregnant            BOOLEAN DEFAULT false,

    -- Government program
    government_funded       BOOLEAN DEFAULT false,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_schedule_vaccine (vaccine_code)
);

-- Due/overdue immunizations view
CREATE VIEW immunizations_due AS
SELECT
    p.id as patient_id,
    p.first_name,
    p.last_name,
    p.date_of_birth,
    EXTRACT(YEAR FROM AGE(p.date_of_birth)) as age_years,
    EXTRACT(MONTH FROM AGE(p.date_of_birth)) as age_months_total,
    s.vaccine_code,
    s.vaccine_name,
    s.dose_number,
    s.is_mandatory,
    CASE
        WHEN i.id IS NULL THEN 'never_received'
        ELSE 'booster_due'
    END as status
FROM patients p
CROSS JOIN immunization_schedule s
LEFT JOIN immunizations i
    ON i.patient_id = p.id
    AND i.vaccine_code = s.vaccine_code
    AND i.dose_number = s.dose_number
WHERE
    (i.id IS NULL OR i.status != 'completed')
    AND EXTRACT(MONTH FROM AGE(p.date_of_birth)) >= s.age_min_months
    AND (s.age_max_months IS NULL OR EXTRACT(MONTH FROM AGE(p.date_of_birth)) <= s.age_max_months);
```

**Features Required:**
- ✅ Argentine vaccination calendar (Calendario Nacional de Vacunación)
- ✅ COVID-19 vaccination tracking
- ✅ Automatic due/overdue alerts
- ✅ Vaccine inventory management
- ✅ Adverse event reporting
- ✅ Integration with NOMIVAC (Argentina's vaccination registry)
- ✅ Vaccination certificate printing
- ✅ SMS reminders for due vaccines
- ✅ Pediatric growth chart integration
- ✅ Obra Social/PAMI coverage verification

**Argentina-Specific:**
- NOMIVAC integration for government reporting
- Mandatory vaccines: BCG, Hepatitis B, DTP, Polio, Rotavirus, etc.
- COVID-19 vaccination tracking (required for many activities)
- Yellow Fever (for travel to certain provinces)

---

### 1.5 Chronic Disease Management Programs

**Status:** ⚠️ Partially covered in clinical protocols but not complete programs

**Why Critical:**
- High prevalence in Argentina (diabetes, hypertension, obesity)
- Obra Social/PAMI coverage programs
- Improve patient outcomes
- Reduce complications and hospitalizations
- Quality metrics tracking

**Required Specifications:**

```sql
CREATE TABLE chronic_disease_programs (
    id                      BIGSERIAL PRIMARY KEY,
    program_name            VARCHAR(200) NOT NULL,
    disease_code            VARCHAR(20), -- ICD-10 code

    -- Program details
    description             TEXT,
    program_goals           JSONB,
    target_metrics          JSONB,
    /* Example:
    {
      "hba1c_target": "< 7.0%",
      "blood_pressure_target": "< 130/80",
      "ldl_target": "< 100 mg/dL",
      "weight_loss_target": "5-10% reduction"
    }
    */

    -- Visit schedule
    recommended_visit_frequency VARCHAR(50), -- 'monthly', 'quarterly', 'annually'

    -- Required tests
    required_tests          JSONB,
    /* Example:
    {
      "HbA1c": {"frequency": "every 3 months"},
      "Lipid Panel": {"frequency": "annually"},
      "Microalbumin": {"frequency": "annually"},
      "Eye Exam": {"frequency": "annually"}
    }
    */

    -- Patient education
    education_modules       JSONB,

    -- Insurance coverage
    covered_by_obras_sociales BOOLEAN DEFAULT false,
    covered_by_pami         BOOLEAN DEFAULT false,

    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_chronic_programs_disease (disease_code)
);

-- Patient enrollment in programs
CREATE TABLE chronic_disease_enrollments (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    program_id              BIGINT NOT NULL REFERENCES chronic_disease_programs(id),

    -- Enrollment details
    enrolled_date           DATE NOT NULL,
    enrolled_by             BIGINT REFERENCES users(id),

    -- Status
    status                  VARCHAR(50) DEFAULT 'active',
    -- 'active', 'inactive', 'completed', 'discontinued'

    discontinuation_date    DATE,
    discontinuation_reason  TEXT,

    -- Metrics tracking
    baseline_metrics        JSONB, -- Metrics at enrollment
    current_metrics         JSONB, -- Current metrics
    last_metrics_update     DATE,

    -- Compliance
    visit_compliance_rate   DECIMAL(5,2), -- % of recommended visits attended
    test_compliance_rate    DECIMAL(5,2), -- % of required tests completed

    -- Goals
    goals_met               BOOLEAN DEFAULT false,
    goals_met_date          DATE,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(patient_id, program_id),
    INDEX idx_chronic_enrollment_patient (patient_id),
    INDEX idx_chronic_enrollment_program (program_id),
    INDEX idx_chronic_enrollment_status (status)
);

-- Program milestones tracking
CREATE TABLE program_milestones (
    id                      BIGSERIAL PRIMARY KEY,
    enrollment_id           BIGINT NOT NULL REFERENCES chronic_disease_enrollments(id),

    -- Milestone details
    milestone_type          VARCHAR(50) NOT NULL,
    -- 'initial_assessment', 'quarterly_review', 'annual_review', 'goal_achieved'

    milestone_date          DATE NOT NULL,

    -- Results
    metrics_recorded        JSONB,
    goals_status            JSONB,
    notes                   TEXT,

    -- Provider
    recorded_by             BIGINT REFERENCES users(id),

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_program_milestones_enrollment (enrollment_id)
);
```

**Features Required:**
- ✅ Pre-configured programs (Diabetes, Hypertension, COPD, Asthma)
- ✅ Automated enrollment based on diagnosis
- ✅ Visit reminders based on program schedule
- ✅ Test reminder system
- ✅ Metrics tracking dashboard
- ✅ Patient education materials
- ✅ Goal setting and tracking
- ✅ Care team coordination
- ✅ Progress reports for patient and obra social
- ✅ Population health analytics

**Argentina Programs:**
- PRODIABA (Diabetes Program)
- PRONACORD (Cardiovascular Risk Program)
- REMEDIAR (Essential Medications Program)
- SUMAR (Health Insurance for vulnerable populations)

---

### 1.6 Medical Device Integration

**Status:** ❌ NOT SPECIFIED

**Why Critical:**
- Common devices in clinics (glucometers, BP monitors, ECG, pulse oximeters)
- Reduces manual data entry
- Improves accuracy
- Standard in modern clinics

**Required Specifications:**

```sql
CREATE TABLE medical_devices (
    id                      BIGSERIAL PRIMARY KEY,
    device_name             VARCHAR(200) NOT NULL,
    device_type             VARCHAR(100) NOT NULL,
    -- 'glucometer', 'blood_pressure_monitor', 'ecg', 'pulse_oximeter', 'spirometer', 'thermometer', 'weight_scale'

    -- Device details
    manufacturer            VARCHAR(200),
    model                   VARCHAR(200),
    serial_number           VARCHAR(200),

    -- Integration
    integration_protocol    VARCHAR(50), -- 'HL7', 'bluetooth', 'usb', 'api'
    integration_config      JSONB,

    -- Location
    location_id             BIGINT REFERENCES locations(id),

    -- Calibration
    last_calibration_date   DATE,
    next_calibration_due    DATE,
    calibration_required    BOOLEAN DEFAULT false,

    -- Status
    status                  VARCHAR(50) DEFAULT 'active',
    -- 'active', 'inactive', 'maintenance', 'decommissioned'

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_devices_type (device_type),
    INDEX idx_devices_location (location_id)
);

-- Device readings
CREATE TABLE device_readings (
    id                      BIGSERIAL PRIMARY KEY,
    device_id               BIGINT NOT NULL REFERENCES medical_devices(id),
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    encounter_id            BIGINT REFERENCES encounters(id),

    -- Reading details
    reading_type            VARCHAR(50) NOT NULL,
    reading_value           JSONB NOT NULL,
    /* Example for BP monitor:
    {
      "systolic": 120,
      "diastolic": 80,
      "pulse": 72
    }
    */

    reading_timestamp       TIMESTAMP NOT NULL,

    -- Interpretation
    is_abnormal             BOOLEAN DEFAULT false,
    abnormal_flag           VARCHAR(50), -- 'high', 'low', 'critical'

    -- Who took the reading
    recorded_by             BIGINT REFERENCES users(id),

    -- Auto-import to vital signs
    imported_to_vitals      BOOLEAN DEFAULT false,
    vital_sign_id           BIGINT,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_device_readings_device (device_id),
    INDEX idx_device_readings_patient (patient_id),
    INDEX idx_device_readings_timestamp (reading_timestamp)
);
```

**Features Required:**
- ✅ Bluetooth device pairing
- ✅ USB device connection
- ✅ Automatic data import to vital signs
- ✅ Device calibration tracking
- ✅ Multi-device support per location
- ✅ Real-time alerts for critical values
- ✅ Device maintenance scheduling

**Common Devices in Argentina:**
- Blood glucose meters (very common for diabetes management)
- Blood pressure monitors
- Pulse oximeters (post-COVID essential)
- Digital thermometers
- ECG machines (small clinics)
- Spirometers (for respiratory conditions)

---

## 2. IMPORTANT MISSING FEATURES

### 2.1 Quality Measures Tracking

**Status:** ❌ NOT SPECIFIED

**Why Important:**
- Required for accreditation
- Obra Social contracts requirements
- Continuous quality improvement
- Benchmarking

```sql
CREATE TABLE quality_measures (
    id                      BIGSERIAL PRIMARY KEY,
    measure_name            VARCHAR(200) NOT NULL,
    measure_code            VARCHAR(50), -- HEDIS, CMS, or local code

    -- Measure details
    category                VARCHAR(100), -- 'prevention', 'chronic_care', 'patient_safety'
    description             TEXT,

    -- Calculation
    numerator_definition    TEXT, -- Who counts in numerator
    denominator_definition  TEXT, -- Total eligible population
    exclusions              TEXT,

    -- Target
    target_percentage       DECIMAL(5,2),

    -- Frequency
    reporting_frequency     VARCHAR(50), -- 'monthly', 'quarterly', 'annually'

    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Quality measure results
CREATE TABLE quality_measure_results (
    id                      BIGSERIAL PRIMARY KEY,
    measure_id              BIGINT NOT NULL REFERENCES quality_measures(id),

    -- Period
    period_start            DATE NOT NULL,
    period_end              DATE NOT NULL,

    -- Results
    numerator               INTEGER,
    denominator             INTEGER,
    percentage              DECIMAL(5,2),

    -- Comparison
    meets_target            BOOLEAN,
    previous_period_percentage DECIMAL(5,2),

    calculated_at           TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_quality_results_measure (measure_id),
    INDEX idx_quality_results_period (period_start, period_end)
);
```

**Examples for Argentina:**
- Diabetes: % patients with HbA1c < 7%
- Hypertension: % patients with BP < 140/90
- Prevention: % patients up to date with vaccines
- Prescribing: % patients on evidence-based medications

---

### 2.2 Population Health Management

**Status:** ❌ NOT SPECIFIED

```sql
CREATE TABLE population_health_panels (
    id                      BIGSERIAL PRIMARY KEY,
    panel_name              VARCHAR(200) NOT NULL,

    -- Panel criteria
    inclusion_criteria      JSONB,
    /* Example:
    {
      "diagnoses": ["E11", "E10"], // Diabetes
      "age_min": 18,
      "insurance_ids": [1, 2, 3]
    }
    */

    -- Assigned care team
    primary_provider_id     BIGINT REFERENCES users(id),
    care_coordinator_id     BIGINT REFERENCES users(id),

    -- Metrics
    total_patients          INTEGER,
    high_risk_count         INTEGER,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Risk stratification
CREATE TABLE patient_risk_scores (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),

    -- Risk calculation
    risk_score              DECIMAL(5,2),
    risk_category           VARCHAR(50), -- 'low', 'medium', 'high', 'very_high'

    -- Factors contributing to risk
    risk_factors            JSONB,
    /* Example:
    {
      "uncontrolled_diabetes": true,
      "no_show_rate": 0.4,
      "missed_medications": true,
      "er_visits_last_year": 3
    }
    */

    -- Interventions recommended
    recommended_interventions JSONB,

    calculated_at           TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_risk_scores_patient (patient_id),
    INDEX idx_risk_scores_category (risk_category)
);
```

---

### 2.3 Care Coordination Tools

**Status:** ⚠️ Partially in referrals but incomplete

```sql
CREATE TABLE care_team_assignments (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),

    -- Care team members
    primary_physician_id    BIGINT REFERENCES users(id),
    care_coordinator_id     BIGINT REFERENCES users(id),
    specialist_ids          BIGINT[], -- Array of specialist IDs

    -- Team communication
    communication_method    VARCHAR(50), -- 'secure_messaging', 'huddle', 'phone'
    last_team_meeting       DATE,
    next_team_meeting       DATE,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_care_team_patient (patient_id),
    INDEX idx_care_team_coordinator (care_coordinator_id)
);

CREATE TABLE care_coordination_tasks (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),

    -- Task details
    task_type               VARCHAR(50),
    -- 'referral_followup', 'test_followup', 'medication_reconciliation', 'insurance_authorization'

    task_description        TEXT,
    priority                VARCHAR(50) DEFAULT 'normal',

    -- Assignment
    assigned_to             BIGINT REFERENCES users(id),
    assigned_by             BIGINT REFERENCES users(id),

    -- Due date
    due_date                DATE,

    -- Status
    status                  VARCHAR(50) DEFAULT 'pending',
    completed_at            TIMESTAMP,
    completion_notes        TEXT,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_coordination_tasks_patient (patient_id),
    INDEX idx_coordination_tasks_assigned (assigned_to),
    INDEX idx_coordination_tasks_status (status)
);
```

---

### 2.4 Clinical Decision Support (CDS) Rules Engine

**Status:** ⚠️ AI integration exists but not rule-based CDS

```sql
CREATE TABLE clinical_decision_rules (
    id                      BIGSERIAL PRIMARY KEY,
    rule_name               VARCHAR(200) NOT NULL,
    rule_category           VARCHAR(100), -- 'preventive_care', 'drug_interaction', 'diagnosis_support'

    -- Trigger conditions
    trigger_conditions      JSONB NOT NULL,
    /* Example:
    {
      "age_min": 50,
      "gender": "M",
      "diagnosis": "E11", // Diabetes
      "lab_result": {
        "test": "HbA1c",
        "operator": ">",
        "value": 9.0
      }
    }
    */

    -- Alert/Recommendation
    alert_type              VARCHAR(50), -- 'info', 'warning', 'critical'
    alert_message           TEXT,
    recommendation          TEXT,

    -- Actions
    suggested_orders        JSONB, -- Tests, medications to order
    educational_materials   BIGINT[], -- Document IDs

    -- Evidence
    evidence_source         VARCHAR(200), -- Guideline reference
    evidence_level          VARCHAR(50), -- 'A', 'B', 'C'

    -- Activation
    is_active               BOOLEAN DEFAULT true,
    priority                INTEGER DEFAULT 0,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_cds_rules_category (rule_category)
);

-- CDS alerts triggered
CREATE TABLE clinical_decision_alerts (
    id                      BIGSERIAL PRIMARY KEY,
    rule_id                 BIGINT NOT NULL REFERENCES clinical_decision_rules(id),
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    encounter_id            BIGINT REFERENCES encounters(id),

    -- Alert details
    alert_message           TEXT,
    alert_severity          VARCHAR(50),

    -- Provider response
    shown_to_provider       BOOLEAN DEFAULT false,
    provider_response       VARCHAR(50), -- 'acknowledged', 'accepted', 'overridden'
    override_reason         TEXT,

    -- Timing
    triggered_at            TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    responded_at            TIMESTAMP,

    INDEX idx_cds_alerts_patient (patient_id),
    INDEX idx_cds_alerts_rule (rule_id),
    INDEX idx_cds_alerts_triggered (triggered_at)
);
```

**Examples of CDS Rules:**
- Age 50+ male without colonoscopy → suggest preventive screening
- Diabetes patient with HbA1c > 9% → alert to intensify therapy
- Drug-drug interaction → warning before prescribing
- Abnormal lab value → alert provider
- Overdue vaccines → reminder in chart

---

## 3. ARGENTINA-SPECIFIC MISSING FEATURES

### 3.1 Integration with Government Health Systems

**Status:** ❌ NOT SPECIFIED

**Required Integrations:**

1. **NOMIVAC** (Registro Federal de Vacunación Nominal)
   - Submit vaccination records
   - Query patient vaccination history
   - Required for COVID-19 vaccination reporting

2. **SiSa** (Sistema de Información Sanitaria)
   - Submit epidemiological data
   - Disease notification (dengue, measles, etc.)
   - Statistical reporting

3. **SISA** (Sistema Integrado de Información Sanitaria Argentina)
   - Healthcare provider registry
   - Professional credential verification

4. **REFES** (Registro Federal de Establecimientos de Salud)
   - Facility registration and updates

### 3.2 Obra Social Specific Features

**Missing:**
- **Obra Social portal integration** - Check coverage online
- **Pre-authorization status checking** - Real-time status
- **Electronic claims submission** - Direct submission to obras sociales
- **Coverage rules engine** - Different rules per obra social
- **PAMI specific workflows** - Chronic disease programs, medication coverage

### 3.3 Argentine Prescription Requirements

**Missing detailed specs:**
- **Receta archivada** (Filed prescription) for controlled substances
- **Receta por duplicado** (Duplicate prescription) for psychotropics
- **Receta electrónica validation** with professional signature
- **Prescription number tracking** per physician
- **Controlled substance registry reporting**

---

## 4. NICE-TO-HAVE FEATURES (Lower Priority)

### 4.1 Voice Recognition / Speech-to-Text

- Dictation for clinical notes
- Faster documentation
- Reduces typing time

### 4.2 Mobile App for Providers

- View schedule on mobile
- Quick patient lookup
- Approve/deny requests
- View lab results

### 4.3 Medical Equipment Maintenance Tracking

- Track equipment maintenance schedules
- Calibration reminders
- Service history

### 4.4 Staff Performance Analytics

- Productivity metrics
- Patient satisfaction by staff
- Revenue generated per provider

### 4.5 Marketing Campaign Management

- Email campaigns
- Patient retention programs
- Birthday greetings
- Health awareness campaigns

---

## 5. PRIORITY RECOMMENDATIONS

### MUST IMPLEMENT (Phase 2):
1. ✅ **Telemedicine** - Essential in 2025, patient expectation
2. ✅ **Online Self-Booking** - Reduces workload, improves satisfaction
3. ✅ **Immunization Registry** - Required for pediatrics, government reporting

### SHOULD IMPLEMENT (Phase 3):
4. ✅ **Medication Reconciliation** - Patient safety, quality of care
5. ✅ **Chronic Disease Programs** - High ROI, obra social requirements
6. ✅ **Medical Device Integration** - Reduces errors, saves time

### COULD IMPLEMENT (Phase 4):
7. ✅ **Quality Measures** - Continuous improvement
8. ✅ **Population Health** - Advanced capability
9. ✅ **CDS Rules Engine** - Clinical decision support
10. ✅ **Care Coordination** - For complex patients

### ARGENTINA-SPECIFIC (Phase 2-3):
11. ✅ **NOMIVAC Integration** - Government requirement
12. ✅ **Obra Social Portal Integration** - Workflow efficiency
13. ✅ **Controlled Substance Prescription** - Legal compliance

---

## 6. IMPLEMENTATION ROADMAP ADJUSTMENTS

### Recommended Changes to PROJECT_ROADMAP.md:

**Phase 2 - Enhanced Clinical (Add):**
- Telemedicine/virtual consultations
- Online patient self-booking
- Immunization registry with NOMIVAC integration

**Phase 3 - Patient Experience (Add):**
- Medication reconciliation
- Chronic disease management programs
- Medical device integration

**Phase 4 - Advanced Features (Add):**
- Clinical decision support rules engine
- Quality measures tracking
- Population health management
- Care coordination tools

**Phase 5 - Integrations (Add Argentina-Specific):**
- NOMIVAC (vaccination registry)
- SiSa/SISA integration
- Obra Social portal integrations
- Controlled substance reporting

---

*Gap Analysis Version: 1.0*
*Last Updated: 2025-11-16*
*Comprehensive feature gap analysis for Argentina clinical management system*
