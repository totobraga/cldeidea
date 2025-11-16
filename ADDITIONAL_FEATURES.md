# Clinical Management System - Additional Critical Features

## Overview

This document specifies additional critical features that are essential for a complete clinical management system but were not fully detailed in other specification documents.

---

## 1. REFERRAL MANAGEMENT SYSTEM

### 1.1 Patient Referrals

#### 1.1.1 Referral Database Schema

```sql
CREATE TABLE referrals (
    id                      BIGSERIAL PRIMARY KEY,

    -- Patient and providers
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    referring_provider_id   BIGINT NOT NULL REFERENCES users(id), -- Doctor making referral
    referred_to_provider_id BIGINT REFERENCES users(id), -- Internal specialist
    referred_to_external    VARCHAR(200), -- External specialist/facility name

    -- Referral details
    referral_type           VARCHAR(50) NOT NULL, -- 'specialist', 'diagnostic', 'therapy', 'hospitalization'
    specialty               VARCHAR(100), -- 'cardiology', 'dermatology', etc.
    urgency                 VARCHAR(50) DEFAULT 'routine', -- 'routine', 'urgent', 'emergency'

    -- Clinical information
    reason                  TEXT NOT NULL, -- Why referral is needed
    diagnosis_codes         VARCHAR(20)[], -- ICD-10 codes
    clinical_summary        TEXT, -- Patient history, current medications, etc.
    specific_questions      TEXT, -- What referring doctor wants to know

    -- Documents
    encounter_id            BIGINT REFERENCES encounters(id),
    attached_documents      BIGINT[], -- Array of document IDs
    lab_results_ids         BIGINT[], -- Relevant lab results

    -- Insurance authorization
    requires_authorization  BOOLEAN DEFAULT false,
    authorization_number    VARCHAR(100),
    authorization_status    VARCHAR(50), -- 'pending', 'approved', 'denied'
    authorization_date      DATE,

    -- Appointment tracking
    appointment_scheduled   BOOLEAN DEFAULT false,
    appointment_date        TIMESTAMP,
    appointment_location    VARCHAR(200),

    -- Follow-up
    status                  VARCHAR(50) DEFAULT 'pending',
    -- Values: 'pending', 'scheduled', 'completed', 'cancelled', 'no_show'

    report_received         BOOLEAN DEFAULT false,
    report_date             DATE,
    report_summary          TEXT,
    report_document_id      BIGINT REFERENCES documents(id),

    -- Communication
    patient_notified        BOOLEAN DEFAULT false,
    patient_notified_at     TIMESTAMP,

    -- Metadata
    referral_date           DATE DEFAULT CURRENT_DATE,
    valid_until             DATE, -- Referral expiration

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_referrals_patient (patient_id),
    INDEX idx_referrals_referring (referring_provider_id),
    INDEX idx_referrals_referred_to (referred_to_provider_id),
    INDEX idx_referrals_status (status),
    INDEX idx_referrals_date (referral_date)
);

-- Referral follow-up tracking
CREATE TABLE referral_follow_ups (
    id                      BIGSERIAL PRIMARY KEY,
    referral_id             BIGINT NOT NULL REFERENCES referrals(id) ON DELETE CASCADE,

    follow_up_date          DATE NOT NULL,
    follow_up_type          VARCHAR(50), -- 'phone_call', 'appointment', 'email'
    notes                   TEXT,
    outcome                 VARCHAR(50), -- 'scheduled', 'completed', 'patient_declined', 'no_response'

    completed_by            BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_referral_followups_referral (referral_id)
);
```

#### 1.1.2 Referral Templates

```sql
CREATE TABLE referral_templates (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    specialty               VARCHAR(100),

    -- Template content
    reason_template         TEXT,
    clinical_summary_template TEXT,
    questions_template      TEXT,

    -- Default settings
    default_urgency         VARCHAR(50),
    requires_authorization  BOOLEAN DEFAULT false,

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_referral_templates_specialty (specialty)
);

-- Example templates
INSERT INTO referral_templates (name, specialty, reason_template, clinical_summary_template, questions_template, default_urgency)
VALUES
('Cardiology Consultation', 'cardiology',
 'Patient presents with {{symptoms}}. Requesting cardiology evaluation for {{concern}}.',
 'Medical History: {{history}}
Current Medications: {{medications}}
Recent Labs: {{labs}}
Physical Exam Findings: {{exam_findings}}',
 'Please evaluate for {{condition}} and advise on management.',
 'routine'),

('Dermatology Referral', 'dermatology',
 'Patient with {{skin_condition}} requiring dermatology evaluation.',
 'Lesion Description: {{description}}
Duration: {{duration}}
Previous Treatments: {{treatments}}
Associated Symptoms: {{symptoms}}',
 'Please evaluate and recommend treatment plan.',
 'routine'),

('Urgent Neurology Consult', 'neurology',
 'Urgent referral for {{neurological_symptoms}}',
 'Onset: {{onset}}
Duration: {{duration}}
Associated Symptoms: {{symptoms}}
Neurological Exam: {{neuro_exam}}
Imaging: {{imaging_results}}',
 'Please evaluate urgently and advise on immediate management.',
 'urgent');
```

### 1.2 Referral Workflow

```
┌─────────────────────────────────────────────────────────────┐
│ REFERRAL WORKFLOW                                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. CREATION                                                 │
│    ├─ Doctor identifies need for specialist                │
│    ├─ Selects referral template (optional)                 │
│    ├─ Fills clinical information                           │
│    └─ Attaches relevant documents/labs                     │
│                                                             │
│ 2. INSURANCE AUTHORIZATION (if required)                    │
│    ├─ Check if insurance requires prior auth               │
│    ├─ Submit authorization request                         │
│    ├─ Track authorization status                           │
│    └─ Receive authorization number                         │
│                                                             │
│ 3. PATIENT NOTIFICATION                                     │
│    ├─ Notify patient of referral                           │
│    ├─ Provide specialist contact information               │
│    └─ Instructions for scheduling                          │
│                                                             │
│ 4. SCHEDULING                                               │
│    ├─ Patient schedules with specialist, OR                │
│    ├─ Staff schedules appointment for patient              │
│    └─ Update referral with appointment details             │
│                                                             │
│ 5. FOLLOW-UP                                                │
│    ├─ Track appointment completion                         │
│    ├─ Request report from specialist                       │
│    ├─ Review specialist report                             │
│    └─ Document recommendations in patient chart            │
│                                                             │
│ 6. CLOSURE                                                  │
│    ├─ Mark referral as completed                           │
│    ├─ Follow up on specialist recommendations              │
│    └─ Schedule follow-up appointment if needed             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. PRIOR AUTHORIZATION MANAGEMENT

### 2.1 Authorization Requests

```sql
CREATE TABLE authorization_requests (
    id                      BIGSERIAL PRIMARY KEY,

    -- Patient and insurance
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    insurance_id            BIGINT NOT NULL REFERENCES insurances(id),

    -- Request details
    request_type            VARCHAR(50) NOT NULL,
    -- Values: 'procedure', 'medication', 'imaging', 'specialist', 'hospitalization', 'durable_medical_equipment'

    -- What is being authorized
    service_code            VARCHAR(50), -- Nomenclator code or CPT code
    service_description     TEXT NOT NULL,
    medication_name         VARCHAR(200),
    procedure_name          VARCHAR(200),

    -- Clinical justification
    diagnosis_codes         VARCHAR(20)[] NOT NULL, -- ICD-10 codes
    clinical_justification  TEXT NOT NULL,
    medical_necessity       TEXT,

    -- Requesting provider
    requesting_provider_id  BIGINT NOT NULL REFERENCES users(id),
    provider_license        VARCHAR(100),

    -- Urgency
    urgency                 VARCHAR(50) DEFAULT 'routine', -- 'routine', 'urgent', 'emergency'
    requested_date          DATE, -- When service is needed

    -- Submission details
    submission_method       VARCHAR(50), -- 'online', 'phone', 'fax', 'email'
    submission_date         DATE DEFAULT CURRENT_DATE,
    submitted_by            BIGINT REFERENCES users(id),

    -- Insurance response
    status                  VARCHAR(50) DEFAULT 'pending',
    -- Values: 'pending', 'submitted', 'under_review', 'approved', 'denied', 'more_info_needed'

    authorization_number    VARCHAR(100),
    approval_date           DATE,
    denial_reason           TEXT,
    valid_from              DATE,
    valid_until             DATE,

    -- Units/visits authorized
    units_requested         INTEGER,
    units_approved          INTEGER,
    units_used              INTEGER DEFAULT 0,

    -- Appeal process
    appeal_filed            BOOLEAN DEFAULT false,
    appeal_date             DATE,
    appeal_outcome          VARCHAR(50),

    -- Documents
    supporting_documents    BIGINT[], -- Array of document IDs
    response_document_id    BIGINT REFERENCES documents(id),

    -- Related entities
    referral_id             BIGINT REFERENCES referrals(id),
    encounter_id            BIGINT REFERENCES encounters(id),

    -- Tracking
    review_notes            TEXT,
    internal_notes          TEXT,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_auth_requests_patient (patient_id),
    INDEX idx_auth_requests_insurance (insurance_id),
    INDEX idx_auth_requests_status (status),
    INDEX idx_auth_requests_provider (requesting_provider_id),
    INDEX idx_auth_requests_submission (submission_date)
);

-- Authorization tracking history
CREATE TABLE authorization_history (
    id                      BIGSERIAL PRIMARY KEY,
    authorization_id        BIGINT NOT NULL REFERENCES authorization_requests(id) ON DELETE CASCADE,

    status_from             VARCHAR(50),
    status_to               VARCHAR(50) NOT NULL,
    notes                   TEXT,

    changed_by              BIGINT REFERENCES users(id),
    changed_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_auth_history_authorization (authorization_id)
);
```

### 2.2 Authorization Templates by Insurance

```sql
CREATE TABLE insurance_authorization_requirements (
    id                      BIGSERIAL PRIMARY KEY,
    insurance_id            BIGINT NOT NULL REFERENCES insurances(id),

    -- Service requiring auth
    service_type            VARCHAR(50) NOT NULL, -- 'procedure', 'medication', 'imaging', etc.
    service_code            VARCHAR(50), -- Specific code, or NULL for all

    -- Requirements
    requires_authorization  BOOLEAN DEFAULT true,
    submission_method       VARCHAR(50)[], -- Allowed methods: ['online', 'phone', 'fax']

    -- Processing times (in days)
    typical_review_days     INTEGER,
    urgent_review_days      INTEGER,

    -- Required documents
    required_documents      TEXT[], -- ['medical_records', 'lab_results', 'imaging', 'letter_of_necessity']

    -- Contact information
    authorization_phone     VARCHAR(50),
    authorization_fax       VARCHAR(50),
    authorization_url       VARCHAR(500),

    -- Notes
    special_instructions    TEXT,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_insurance_auth_reqs_insurance (insurance_id),
    INDEX idx_insurance_auth_reqs_service (service_type)
);

-- Example: OSDE authorization requirements
INSERT INTO insurance_authorization_requirements (
    insurance_id, service_type, requires_authorization, submission_method,
    typical_review_days, authorization_phone, authorization_url, special_instructions
) VALUES
(1, 'imaging', true, ARRAY['online', 'phone'],
 3, '0810-555-6733', 'https://www.osde.com.ar/autorizaciones',
 'Resonancias y tomografías requieren autorización previa. Rx y ecografías no.'),

(1, 'specialist', true, ARRAY['online'],
 1, '0810-555-6733', 'https://www.osde.com.ar/autorizaciones',
 'Derivación a especialista requiere orden del médico de cabecera.'),

(1, 'procedure', true, ARRAY['online', 'phone', 'fax'],
 5, '0810-555-6733', 'https://www.osde.com.ar/autorizaciones',
 'Procedimientos quirúrgicos y de alto costo requieren evaluación médica.');
```

---

## 3. CLINICAL TEMPLATES & PROTOCOLS

### 3.1 Clinical Note Templates

```sql
CREATE TABLE clinical_templates (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Template type
    template_type           VARCHAR(50) NOT NULL,
    -- Values: 'soap_note', 'procedure_note', 'consultation', 'progress_note', 'discharge_summary'

    -- Specialty/condition specific
    specialty               VARCHAR(100),
    condition               VARCHAR(100), -- Specific condition (e.g., 'Diabetes', 'Hypertension')
    icd10_codes             VARCHAR(20)[],

    -- Template sections
    template_structure      JSONB NOT NULL,
    /* Example:
    {
      "sections": [
        {
          "name": "Subjective",
          "fields": [
            {"name": "Chief Complaint", "type": "text", "required": true},
            {"name": "HPI", "type": "textarea", "placeholder": "History of present illness..."},
            {"name": "Review of Systems", "type": "checklist", "options": ["Constitutional", "HEENT", "Cardiovascular", ...]}
          ]
        },
        {
          "name": "Objective",
          "fields": [
            {"name": "Vital Signs", "type": "vitals"},
            {"name": "Physical Exam", "type": "textarea"},
            {"name": "Pertinent Findings", "type": "text"}
          ]
        },
        {
          "name": "Assessment",
          "fields": [
            {"name": "Diagnoses", "type": "icd10_search", "multiple": true},
            {"name": "Clinical Impression", "type": "textarea"}
          ]
        },
        {
          "name": "Plan",
          "fields": [
            {"name": "Treatment Plan", "type": "textarea"},
            {"name": "Medications", "type": "medication_picker", "multiple": true},
            {"name": "Orders", "type": "order_picker"},
            {"name": "Follow-up", "type": "text"}
          ]
        }
      ]
    }
    */

    -- Usage
    is_favorite             BOOLEAN DEFAULT false,
    use_count               INTEGER DEFAULT 0,

    -- Access control
    is_public               BOOLEAN DEFAULT false,
    created_by              BIGINT REFERENCES users(id),
    allowed_roles           VARCHAR(50)[],

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_clinical_templates_type (template_type),
    INDEX idx_clinical_templates_specialty (specialty),
    INDEX idx_clinical_templates_creator (created_by)
);
```

### 3.2 Clinical Pathways/Protocols

```sql
CREATE TABLE clinical_protocols (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Applicability
    condition               VARCHAR(100) NOT NULL, -- 'Diabetes Type 2', 'Hypertension', etc.
    icd10_codes             VARCHAR(20)[],
    age_min                 INTEGER,
    age_max                 INTEGER,

    -- Protocol steps
    protocol_steps          JSONB NOT NULL,
    /* Example for Diabetes Management:
    {
      "initial_assessment": {
        "required_tests": ["HbA1c", "Fasting Glucose", "Lipid Panel", "Microalbumin"],
        "required_vitals": ["weight", "BMI", "blood_pressure"],
        "required_exams": ["foot_exam", "eye_exam"]
      },
      "treatment_thresholds": {
        "hba1c_target": 7.0,
        "bp_target": "130/80",
        "ldl_target": 100
      },
      "medication_algorithm": {
        "first_line": "Metformin",
        "second_line": ["Sulfonylurea", "DPP-4 inhibitor", "GLP-1 agonist"],
        "insulin_criteria": "HbA1c > 9% or symptoms"
      },
      "monitoring_schedule": {
        "hba1c": "every 3 months",
        "lipids": "annually",
        "microalbumin": "annually",
        "eye_exam": "annually",
        "foot_exam": "every visit"
      },
      "patient_education": [
        "Diet and nutrition counseling",
        "Exercise recommendations",
        "Blood glucose monitoring",
        "Foot care",
        "Hypoglycemia recognition"
      ]
    }
    */

    -- Evidence-based guidelines
    guideline_source        VARCHAR(200), -- e.g., "ADA 2025 Guidelines"
    evidence_level          VARCHAR(50),

    -- Usage
    is_active               BOOLEAN DEFAULT true,
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_clinical_protocols_condition (condition)
);

-- Protocol adherence tracking
CREATE TABLE protocol_adherence (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    protocol_id             BIGINT NOT NULL REFERENCES clinical_protocols(id),

    -- Enrollment
    enrolled_date           DATE DEFAULT CURRENT_DATE,
    enrolled_by             BIGINT REFERENCES users(id),

    -- Status
    status                  VARCHAR(50) DEFAULT 'active', -- 'active', 'completed', 'discontinued'

    -- Tracking
    adherence_score         DECIMAL(5,2), -- Percentage of protocol steps completed
    last_assessment_date    DATE,
    next_assessment_due     DATE,

    -- Notes
    notes                   TEXT,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_protocol_adherence_patient (patient_id),
    INDEX idx_protocol_adherence_protocol (protocol_id)
);
```

---

## 4. AUDIT LOGGING SYSTEM

### 4.1 Comprehensive Audit Trail

```sql
-- Already exists in DATABASE_SCHEMA.md, but enhanced here
CREATE TABLE audit_logs (
    id                      BIGSERIAL PRIMARY KEY,

    -- Who
    user_id                 BIGINT REFERENCES users(id),
    user_email              VARCHAR(255),
    user_role               VARCHAR(50),

    -- What
    action                  VARCHAR(100) NOT NULL,
    -- Values: 'create', 'read', 'update', 'delete', 'login', 'logout', 'export', 'print', 'send', etc.

    entity_type             VARCHAR(100),
    -- Values: 'patient', 'appointment', 'encounter', 'prescription', 'invoice', 'document', etc.

    entity_id               BIGINT,
    entity_description      VARCHAR(500), -- Human-readable description

    -- When
    timestamp               TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Where (source)
    ip_address              VARCHAR(45),
    user_agent              TEXT,
    session_id              VARCHAR(255),

    -- Changes (for update actions)
    old_values              JSONB, -- Previous state
    new_values              JSONB, -- New state
    changed_fields          TEXT[], -- Fields that changed

    -- Additional context
    reason                  TEXT, -- Why action was performed
    notes                   TEXT,

    -- Security
    is_suspicious           BOOLEAN DEFAULT false,
    risk_level              VARCHAR(50), -- 'low', 'medium', 'high'

    -- Related entities
    patient_id              BIGINT REFERENCES patients(id), -- For patient-related actions
    appointment_id          BIGINT REFERENCES appointments(id),

    INDEX idx_audit_logs_user (user_id),
    INDEX idx_audit_logs_timestamp (timestamp),
    INDEX idx_audit_logs_entity (entity_type, entity_id),
    INDEX idx_audit_logs_action (action),
    INDEX idx_audit_logs_patient (patient_id),
    INDEX idx_audit_logs_suspicious (is_suspicious)
);

-- Partitioning by month for performance
-- CREATE TABLE audit_logs_2025_01 PARTITION OF audit_logs
-- FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

### 4.2 Critical Actions to Audit

```sql
-- Configuration: What actions require auditing
CREATE TABLE audit_configuration (
    id                      BIGSERIAL PRIMARY KEY,
    entity_type             VARCHAR(100) NOT NULL,
    action                  VARCHAR(100) NOT NULL,

    -- Audit settings
    enabled                 BOOLEAN DEFAULT true,
    capture_old_values      BOOLEAN DEFAULT false,
    capture_new_values      BOOLEAN DEFAULT true,
    require_reason          BOOLEAN DEFAULT false,

    -- Alerting
    trigger_alert           BOOLEAN DEFAULT false,
    alert_roles             VARCHAR(50)[], -- Notify these roles

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(entity_type, action)
);

-- Critical actions to audit
INSERT INTO audit_configuration (entity_type, action, capture_old_values, capture_new_values, require_reason, trigger_alert, alert_roles)
VALUES
-- Patient data access
('patient', 'read', false, false, false, false, NULL),
('patient', 'update', true, true, false, false, NULL),
('patient', 'delete', true, false, true, true, ARRAY['ADMIN']),
('patient', 'merge', true, true, true, true, ARRAY['ADMIN']),
('patient', 'export', false, true, true, true, ARRAY['ADMIN', 'COMPLIANCE']),

-- Clinical records
('encounter', 'create', false, true, false, false, NULL),
('encounter', 'update', true, true, false, false, NULL),
('encounter', 'delete', true, false, true, true, ARRAY['ADMIN']),
('encounter', 'finalize', true, true, false, false, NULL),

-- Prescriptions
('prescription', 'create', false, true, false, false, NULL),
('prescription', 'update', true, true, true, false, NULL),
('prescription', 'cancel', true, false, true, false, NULL),

-- Financial
('invoice', 'create', false, true, false, false, NULL),
('invoice', 'update', true, true, true, false, NULL),
('invoice', 'delete', true, false, true, true, ARRAY['ADMIN', 'BILLING_MANAGER']),
('payment', 'create', false, true, false, false, NULL),
('payment', 'delete', true, false, true, true, ARRAY['ADMIN', 'BILLING_MANAGER']),
('payment', 'refund', true, true, true, true, ARRAY['BILLING_MANAGER']),

-- System access
('user', 'login', false, true, false, false, NULL),
('user', 'logout', false, false, false, false, NULL),
('user', 'failed_login', false, true, false, true, ARRAY['ADMIN']), -- Security alert
('user', 'password_reset', true, true, true, true, ARRAY['ADMIN']),
('user', 'role_change', true, true, true, true, ARRAY['ADMIN']),

-- Document access
('document', 'read', false, false, false, false, NULL),
('document', 'download', false, true, false, false, NULL),
('document', 'delete', true, false, true, true, ARRAY['ADMIN']),
('document', 'print', false, true, false, false, NULL),

-- Reports and exports
('report', 'export', false, true, false, false, NULL),
('data', 'bulk_export', false, true, true, true, ARRAY['ADMIN', 'COMPLIANCE']);
```

### 4.3 Audit Log Viewing and Reporting

```sql
CREATE VIEW audit_summary AS
SELECT
    DATE(timestamp) as date,
    user_id,
    user_email,
    entity_type,
    action,
    COUNT(*) as action_count,
    COUNT(DISTINCT patient_id) as unique_patients_accessed
FROM audit_logs
GROUP BY DATE(timestamp), user_id, user_email, entity_type, action;

-- Suspicious activity detection
CREATE VIEW suspicious_activity AS
SELECT
    user_id,
    user_email,
    DATE(timestamp) as date,
    COUNT(*) as total_actions,
    COUNT(*) FILTER (WHERE action = 'read' AND entity_type = 'patient') as patient_views,
    COUNT(*) FILTER (WHERE action = 'export') as exports,
    COUNT(DISTINCT patient_id) as unique_patients,
    MAX(timestamp) as last_activity
FROM audit_logs
WHERE timestamp >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY user_id, user_email, DATE(timestamp)
HAVING
    COUNT(*) > 1000 OR -- Excessive activity
    COUNT(*) FILTER (WHERE action = 'export') > 10 OR -- Many exports
    COUNT(DISTINCT patient_id) > 100; -- Accessing many patients
```

---

## 5. ELECTRONIC CONSENT MANAGEMENT

### 5.1 Consent Forms

```sql
CREATE TABLE consent_forms (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Form type
    form_type               VARCHAR(50) NOT NULL,
    -- Values: 'general_treatment', 'procedure_specific', 'anesthesia', 'research',
    --         'photography', 'data_sharing', 'telehealth', 'hipaa_equivalent'

    -- Content
    title                   VARCHAR(500) NOT NULL,
    content                 TEXT NOT NULL, -- HTML content
    legal_text              TEXT,

    -- Requirements
    required_for            VARCHAR(50)[], -- ['all_patients', 'procedures', 'specific_diagnoses']
    procedure_codes         VARCHAR(50)[],
    diagnosis_codes         VARCHAR(20)[],

    -- Validity
    expires_after_days      INTEGER, -- NULL = never expires
    requires_renewal        BOOLEAN DEFAULT false,

    -- Versions
    version                 VARCHAR(20) NOT NULL,
    effective_date          DATE NOT NULL,
    supersedes_version      VARCHAR(20),

    -- Language
    language                VARCHAR(10) DEFAULT 'es',

    -- Status
    is_active               BOOLEAN DEFAULT true,
    created_by              BIGINT REFERENCES users(id),
    approved_by             BIGINT REFERENCES users(id),
    approved_at             TIMESTAMP,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_consent_forms_type (form_type),
    INDEX idx_consent_forms_active (is_active),
    INDEX idx_consent_forms_version (version)
);

-- Patient consent records
CREATE TABLE patient_consents (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    consent_form_id         BIGINT NOT NULL REFERENCES consent_forms(id),

    -- Consent details
    consent_version         VARCHAR(20) NOT NULL,
    consented_at            TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    consented_by            VARCHAR(50) DEFAULT 'patient',
    -- Values: 'patient', 'legal_guardian', 'power_of_attorney'

    -- Guardian information (if applicable)
    guardian_name           VARCHAR(200),
    guardian_relationship   VARCHAR(100),
    guardian_dni            VARCHAR(50),

    -- Signature
    signature_type          VARCHAR(50), -- 'electronic', 'digital', 'wet_signature_scanned'
    signature_data          TEXT, -- Base64 encoded signature image or digital cert
    signature_ip_address    VARCHAR(45),
    signature_timestamp     TIMESTAMP,

    -- Witness (if required)
    witness_name            VARCHAR(200),
    witness_signature       TEXT,
    witnessed_by_user_id    BIGINT REFERENCES users(id),

    -- Consent status
    status                  VARCHAR(50) DEFAULT 'active',
    -- Values: 'active', 'expired', 'revoked', 'superseded'

    expires_at              TIMESTAMP,
    revoked_at              TIMESTAMP,
    revoked_reason          TEXT,

    -- Related entities
    encounter_id            BIGINT REFERENCES encounters(id),
    procedure_id            BIGINT,

    -- Document storage
    signed_document_id      BIGINT REFERENCES documents(id), -- PDF of signed consent

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_patient_consents_patient (patient_id),
    INDEX idx_patient_consents_form (consent_form_id),
    INDEX idx_patient_consents_status (status),
    INDEX idx_patient_consents_expires (expires_at)
);
```

### 5.2 Standard Consent Forms (Argentina)

```sql
-- General treatment consent
INSERT INTO consent_forms (
    name, form_type, title, content, version, effective_date, is_active
) VALUES
('Consentimiento General de Tratamiento', 'general_treatment',
 'CONSENTIMIENTO INFORMADO PARA ATENCIÓN MÉDICA',
 '<div class="consent-form">
    <h2>CONSENTIMIENTO INFORMADO PARA ATENCIÓN MÉDICA</h2>

    <p>Yo, <strong>{{patient_name}}</strong>, DNI <strong>{{patient_dni}}</strong>, por medio del presente documento:</p>

    <h3>DECLARO:</h3>
    <ul>
        <li>Que he sido informado/a sobre mi estado de salud y el tratamiento propuesto</li>
        <li>Que he tenido la oportunidad de hacer preguntas y todas fueron respondidas satisfactoriamente</li>
        <li>Que comprendo los beneficios, riesgos y alternativas del tratamiento propuesto</li>
        <li>Que autorizo al equipo médico de {{clinic_name}} a realizar los procedimientos necesarios para mi atención</li>
    </ul>

    <h3>AUTORIZO:</h3>
    <ul>
        <li>La realización de exámenes médicos, análisis de laboratorio, estudios de diagnóstico y tratamientos que se consideren necesarios</li>
        <li>El acceso a mi historia clínica por parte del personal médico autorizado</li>
        <li>La administración de medicamentos según prescripción médica</li>
    </ul>

    <h3>DERECHO A REVOCAR:</h3>
    <p>Comprendo que puedo revocar este consentimiento en cualquier momento, notificando por escrito a mi médico tratante.</p>

    <h3>CONFIDENCIALIDAD:</h3>
    <p>Entiendo que mi información médica será tratada de forma confidencial de acuerdo con la Ley 25.326 de Protección de Datos Personales.</p>

    <div class="signature-section">
        <p>Fecha: {{consent_date}}</p>
        <p>Firma del Paciente: ____________________</p>
        <p>Aclaración: {{patient_name}}</p>
        <p>DNI: {{patient_dni}}</p>
    </div>
</div>',
 '1.0', '2025-01-01', true
);

-- Data privacy consent (Ley 25.326)
INSERT INTO consent_forms (
    name, form_type, title, content, version, effective_date, is_active, required_for
) VALUES
('Consentimiento Tratamiento de Datos Personales', 'data_sharing',
 'CONSENTIMIENTO PARA TRATAMIENTO DE DATOS PERSONALES - LEY 25.326',
 '<div class="consent-form">
    <h2>CONSENTIMIENTO PARA TRATAMIENTO DE DATOS PERSONALES</h2>

    <p>De conformidad con la Ley 25.326 de Protección de Datos Personales, yo, <strong>{{patient_name}}</strong>, DNI <strong>{{patient_dni}}</strong>:</p>

    <h3>AUTORIZO a {{clinic_name}} a:</h3>
    <ul>
        <li>Recolectar y almacenar mis datos personales y de salud</li>
        <li>Utilizar mis datos para fines de atención médica, facturación y gestión administrativa</li>
        <li>Compartir información médica relevante con profesionales de la salud involucrados en mi atención</li>
        <li>Enviar recordatorios de turnos y comunicaciones relacionadas con mi atención médica</li>
        <li>Compartir datos con obras sociales/prepagas para gestión de autorizaciones y facturación</li>
    </ul>

    <h3>DERECHOS (Art. 14 Ley 25.326):</h3>
    <p>Tengo derecho a:</p>
    <ul>
        <li>Acceder gratuitamente a mis datos personales</li>
        <li>Solicitar la actualización, rectificación o supresión de datos inexactos</li>
        <li>Revocar este consentimiento en cualquier momento</li>
    </ul>

    <h3>SEGURIDAD:</h3>
    <p>{{clinic_name}} se compromete a adoptar las medidas técnicas y organizativas necesarias para garantizar la seguridad y confidencialidad de mis datos personales.</p>

    <div class="signature-section">
        <p>Fecha: {{consent_date}}</p>
        <p>Firma del Paciente: ____________________</p>
    </div>
</div>',
 '1.0', '2025-01-01', true, ARRAY['all_patients']
);
```

---

## 6. PATIENT SATISFACTION SURVEYS

### 6.1 Survey Management

```sql
CREATE TABLE surveys (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Survey type
    survey_type             VARCHAR(50) NOT NULL,
    -- Values: 'post_appointment', 'post_procedure', 'general_satisfaction', 'nps', 'service_specific'

    -- Trigger rules
    trigger_type            VARCHAR(50) DEFAULT 'manual',
    -- Values: 'manual', 'after_appointment', 'after_encounter', 'scheduled'

    trigger_delay_hours     INTEGER, -- Send X hours after appointment

    -- Questions
    questions               JSONB NOT NULL,
    /* Example:
    [
      {
        "id": 1,
        "type": "rating",
        "question": "¿Cómo calificaría la atención recibida?",
        "scale": 5,
        "required": true
      },
      {
        "id": 2,
        "type": "rating",
        "question": "¿Qué tan probable es que recomiende nuestra clínica?",
        "scale": 10,
        "required": true,
        "is_nps": true
      },
      {
        "id": 3,
        "type": "text",
        "question": "¿Qué podríamos mejorar?",
        "required": false
      },
      {
        "id": 4,
        "type": "multiple_choice",
        "question": "¿Cómo conoció nuestra clínica?",
        "options": ["Recomendación", "Internet", "Obra Social", "Otro"],
        "required": false
      }
    ]
    */

    -- Delivery
    delivery_channels       VARCHAR(50)[] DEFAULT ARRAY['email'],
    -- Values: 'email', 'sms', 'whatsapp', 'portal'

    -- Status
    is_active               BOOLEAN DEFAULT true,
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_surveys_type (survey_type),
    INDEX idx_surveys_active (is_active)
);

-- Survey responses
CREATE TABLE survey_responses (
    id                      BIGSERIAL PRIMARY KEY,
    survey_id               BIGINT NOT NULL REFERENCES surveys(id),

    -- Respondent
    patient_id              BIGINT REFERENCES patients(id),

    -- Context
    appointment_id          BIGINT REFERENCES appointments(id),
    encounter_id            BIGINT REFERENCES encounters(id),
    provider_id             BIGINT REFERENCES users(id),

    -- Response data
    answers                 JSONB NOT NULL,
    /* Example:
    {
      "1": {"rating": 5},
      "2": {"rating": 9},
      "3": {"text": "Todo muy bien, solo el tiempo de espera fue largo"},
      "4": {"choice": "Recomendación"}
    }
    */

    -- Calculated scores
    overall_rating          DECIMAL(3,2),
    nps_score               INTEGER, -- -100 to 100

    -- Timing
    sent_at                 TIMESTAMP,
    started_at              TIMESTAMP,
    completed_at            TIMESTAMP,
    time_to_complete_seconds INTEGER,

    -- Status
    status                  VARCHAR(50) DEFAULT 'pending',
    -- Values: 'pending', 'started', 'completed', 'expired'

    -- Follow-up
    requires_follow_up      BOOLEAN DEFAULT false,
    follow_up_reason        TEXT,
    follow_up_completed     BOOLEAN DEFAULT false,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_survey_responses_survey (survey_id),
    INDEX idx_survey_responses_patient (patient_id),
    INDEX idx_survey_responses_provider (provider_id),
    INDEX idx_survey_responses_status (status),
    INDEX idx_survey_responses_completed (completed_at)
);
```

### 6.2 Default Survey Templates

```sql
-- Post-appointment satisfaction survey
INSERT INTO surveys (name, survey_type, trigger_type, trigger_delay_hours, questions, delivery_channels, is_active)
VALUES
('Encuesta Post-Consulta', 'post_appointment', 'after_appointment', 2,
'[
  {
    "id": 1,
    "type": "rating",
    "question": "¿Cómo calificaría la atención del profesional?",
    "scale": 5,
    "labels": ["Muy mala", "Mala", "Regular", "Buena", "Excelente"],
    "required": true
  },
  {
    "id": 2,
    "type": "rating",
    "question": "¿Cómo calificaría el tiempo de espera?",
    "scale": 5,
    "labels": ["Muy largo", "Largo", "Aceptable", "Corto", "Muy corto"],
    "required": true
  },
  {
    "id": 3,
    "type": "rating",
    "question": "¿Cómo calificaría la atención del personal administrativo?",
    "scale": 5,
    "labels": ["Muy mala", "Mala", "Regular", "Buena", "Excelente"],
    "required": true
  },
  {
    "id": 4,
    "type": "rating",
    "question": "¿Qué tan probable es que recomiende nuestra clínica a familiares o amigos?",
    "scale": 10,
    "labels": ["0 = Nada probable", "10 = Muy probable"],
    "required": true,
    "is_nps": true
  },
  {
    "id": 5,
    "type": "text",
    "question": "¿Qué podríamos mejorar?",
    "placeholder": "Sus comentarios nos ayudan a mejorar...",
    "required": false
  },
  {
    "id": 6,
    "type": "text",
    "question": "¿Algo que hicimos especialmente bien?",
    "placeholder": "Nos encantaría saber qué le gustó...",
    "required": false
  }
]'::jsonb,
ARRAY['email', 'whatsapp'],
true);

-- NPS Survey
INSERT INTO surveys (name, survey_type, trigger_type, questions, delivery_channels, is_active)
VALUES
('Net Promoter Score', 'nps', 'manual',
'[
  {
    "id": 1,
    "type": "rating",
    "question": "En una escala de 0 a 10, ¿qué tan probable es que recomiende {{clinic_name}} a un amigo o colega?",
    "scale": 10,
    "labels": ["0 = Nada probable", "10 = Extremadamente probable"],
    "required": true,
    "is_nps": true
  },
  {
    "id": 2,
    "type": "text",
    "question": "¿Cuál es la razón principal de su puntuación?",
    "required": true
  }
]'::jsonb,
ARRAY['email', 'sms'],
true);
```

### 6.3 Analytics Views

```sql
-- Provider satisfaction scores
CREATE VIEW provider_satisfaction_scores AS
SELECT
    sr.provider_id,
    u.first_name || ' ' || u.last_name as provider_name,
    COUNT(*) as total_responses,
    ROUND(AVG(sr.overall_rating), 2) as avg_rating,
    ROUND(AVG(sr.nps_score), 0) as avg_nps,
    COUNT(*) FILTER (WHERE sr.requires_follow_up) as issues_count
FROM survey_responses sr
JOIN users u ON u.id = sr.provider_id
WHERE sr.status = 'completed'
  AND sr.completed_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY sr.provider_id, u.first_name, u.last_name;

-- NPS calculation
CREATE VIEW nps_calculation AS
SELECT
    DATE_TRUNC('month', completed_at) as month,
    COUNT(*) as total_responses,
    COUNT(*) FILTER (WHERE nps_score >= 9) as promoters,
    COUNT(*) FILTER (WHERE nps_score >= 7 AND nps_score <= 8) as passives,
    COUNT(*) FILTER (WHERE nps_score <= 6) as detractors,
    ROUND(
        (COUNT(*) FILTER (WHERE nps_score >= 9)::DECIMAL - COUNT(*) FILTER (WHERE nps_score <= 6)::DECIMAL)
        / COUNT(*)::DECIMAL * 100,
        1
    ) as nps_score
FROM survey_responses
WHERE status = 'completed' AND nps_score IS NOT NULL
GROUP BY DATE_TRUNC('month', completed_at)
ORDER BY month DESC;
```

---

## 7. BACKUP AND DISASTER RECOVERY

### 7.1 Backup Configuration

```sql
CREATE TABLE backup_configurations (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,

    -- Backup type
    backup_type             VARCHAR(50) NOT NULL,
    -- Values: 'full', 'incremental', 'differential'

    -- Schedule
    schedule_type           VARCHAR(50) NOT NULL,
    -- Values: 'daily', 'weekly', 'monthly', 'on_demand'

    schedule_time           TIME,
    schedule_day            INTEGER, -- Day of week (1-7) or day of month (1-31)

    -- What to backup
    include_database        BOOLEAN DEFAULT true,
    include_documents       BOOLEAN DEFAULT true,
    include_images          BOOLEAN DEFAULT true,
    include_system_config   BOOLEAN DEFAULT true,

    -- Retention policy
    retention_days          INTEGER NOT NULL,
    max_backup_count        INTEGER, -- Keep max N backups

    -- Storage location
    storage_type            VARCHAR(50) NOT NULL,
    -- Values: 'local', 's3', 'azure', 'google_cloud'

    storage_path            VARCHAR(500),
    storage_credentials     JSONB, -- Encrypted credentials

    -- Encryption
    encrypt_backup          BOOLEAN DEFAULT true,
    encryption_key_id       VARCHAR(200),

    -- Status
    is_active               BOOLEAN DEFAULT true,
    last_backup_at          TIMESTAMP,
    last_backup_status      VARCHAR(50),
    last_backup_size_mb     DECIMAL(12,2),

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_backup_configs_active (is_active)
);

-- Backup execution log
CREATE TABLE backup_executions (
    id                      BIGSERIAL PRIMARY KEY,
    configuration_id        BIGINT REFERENCES backup_configurations(id),

    -- Execution details
    backup_type             VARCHAR(50) NOT NULL,
    started_at              TIMESTAMP NOT NULL,
    completed_at            TIMESTAMP,
    duration_seconds        INTEGER,

    -- Status
    status                  VARCHAR(50) DEFAULT 'running',
    -- Values: 'running', 'completed', 'failed', 'partial'

    -- Results
    backup_file_path        VARCHAR(500),
    backup_size_mb          DECIMAL(12,2),
    compressed_size_mb      DECIMAL(12,2),

    -- Statistics
    tables_backed_up        INTEGER,
    documents_backed_up     INTEGER,
    total_records           BIGINT,

    -- Error details
    error_message           TEXT,
    warning_messages        TEXT[],

    -- Verification
    checksum                VARCHAR(255),
    verified                BOOLEAN DEFAULT false,
    verified_at             TIMESTAMP,

    -- Metadata
    triggered_by            VARCHAR(50), -- 'schedule', 'manual', 'system'
    triggered_by_user_id    BIGINT REFERENCES users(id),

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_backup_executions_config (configuration_id),
    INDEX idx_backup_executions_started (started_at),
    INDEX idx_backup_executions_status (status)
);
```

### 7.2 Default Backup Strategy

```sql
-- Daily full backup
INSERT INTO backup_configurations (
    name, backup_type, schedule_type, schedule_time,
    include_database, include_documents, include_images,
    retention_days, storage_type, storage_path, encrypt_backup, is_active
) VALUES
('Daily Full Backup', 'full', 'daily', '02:00:00',
 true, true, true,
 30, 's3', 's3://clinic-backups/daily/',
 true, true);

-- Weekly full backup (long-term retention)
INSERT INTO backup_configurations (
    name, backup_type, schedule_type, schedule_day, schedule_time,
    include_database, include_documents, include_images,
    retention_days, storage_type, storage_path, encrypt_backup, is_active
) VALUES
('Weekly Full Backup', 'full', 'weekly', 7, '01:00:00', -- Sunday at 1 AM
 true, true, true,
 365, 's3', 's3://clinic-backups/weekly/',
 true, true);

-- Monthly full backup (archive)
INSERT INTO backup_configurations (
    name, backup_type, schedule_type, schedule_day, schedule_time,
    include_database, include_documents, include_images,
    retention_days, storage_type, storage_path, encrypt_backup, is_active
) VALUES
('Monthly Archive Backup', 'full', 'monthly', 1, '00:00:00', -- 1st of month at midnight
 true, true, true,
 2555, -- 7 years (legal requirement in Argentina)
 's3', 's3://clinic-backups/archive/',
 true, true);
```

---

*Additional Features Specification Version: 1.0*
*Last Updated: 2025-11-16*
*Comprehensive additional features for complete clinical management system*
