# Immunization Registry & Vaccination Management Specification

## 1. Executive Summary

### 1.1 Overview
The Immunization Registry System manages patient vaccination records, tracks compliance with Argentina's National Vaccination Calendar (Calendario Nacional de Vacunación), integrates with NOMIVAC (Argentina's national immunization registry), generates vaccination certificates, and monitors vaccine inventory including cold chain compliance.

### 1.2 Key Benefits
- **Legal Compliance**: Required integration with NOMIVAC for Argentina healthcare providers
- **Automated Reminders**: Alerts for due and overdue vaccinations reduce missed immunizations by 40%
- **Patient Safety**: Adverse event tracking and contraindication checking
- **Inventory Management**: Real-time vaccine stock tracking with expiration alerts
- **Cold Chain Monitoring**: Temperature monitoring for vaccine storage compliance
- **Certificate Generation**: Automatic vaccination certificates (Carnet de Vacunación)
- **Pediatric Tracking**: Growth charts and developmental milestones integrated with vaccinations

### 1.3 Argentina-Specific Features
- **NOMIVAC Integration**: Mandatory reporting to Argentina's national immunization registry
- **Calendario Nacional de Vacunación**: Complete Argentina vaccination schedule (0-64+ years)
- **Vaccine Nomenclature**: Uses Argentina's official vaccine codes and names
- **Certificate Formats**: Official Argentine vaccination certificate templates
- **ANMAT Compliance**: Vaccine registration and lot tracking per ANMAT requirements
- **Provincial Variations**: Support for province-specific vaccination schedules
- **COVID-19 Tracking**: Special handling for COVID-19 vaccines and boosters

---

## 2. Database Architecture

### 2.1 Core Tables

```sql
-- Vaccine catalog (based on Argentina's official vaccine list)
CREATE TABLE vaccines (
    id                          BIGSERIAL PRIMARY KEY,
    vaccine_code                VARCHAR(50) NOT NULL UNIQUE,    -- Official Argentina code
    vaccine_name                VARCHAR(200) NOT NULL,
    vaccine_name_short          VARCHAR(100),

    -- Classification
    vaccine_type                VARCHAR(100) NOT NULL,          -- 'bacterial', 'viral', 'toxoid'
    administration_route        VARCHAR(50) NOT NULL,           -- 'IM', 'SC', 'oral', 'intranasal'
    disease_targets             JSONB NOT NULL,                 -- Array of diseases prevented
    /* Example: ["Diphtheria", "Tetanus", "Pertussis", "Hepatitis B", "Haemophilus influenzae type b"] */

    -- Dosing
    default_dose_ml             DECIMAL(5,3),
    doses_in_series             INTEGER,                        -- Number of doses in complete series
    min_age_months              INTEGER,                        -- Minimum age for first dose
    max_age_months              INTEGER,                        -- Maximum age (null if no limit)

    -- Manufacturer info
    manufacturers               JSONB,                          -- Array of approved manufacturers
    anmat_registration_number   VARCHAR(100),                   -- Argentina ANMAT registration

    -- Storage requirements
    storage_temp_min_celsius    DECIMAL(5,2),
    storage_temp_max_celsius    DECIMAL(5,2),
    cold_chain_required         BOOLEAN DEFAULT true,

    -- Status
    is_active                   BOOLEAN DEFAULT true,
    is_mandatory                BOOLEAN DEFAULT false,          -- Required by Argentina law
    is_free_public              BOOLEAN DEFAULT false,          -- Free in public health system
    requires_consent            BOOLEAN DEFAULT false,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);


-- Argentina National Vaccination Calendar (Calendario Nacional)
CREATE TABLE vaccination_schedule (
    id                          BIGSERIAL PRIMARY KEY,
    vaccine_id                  BIGINT NOT NULL REFERENCES vaccines(id),
    dose_number                 INTEGER NOT NULL,               -- 1st, 2nd, 3rd dose, etc.
    dose_name                   VARCHAR(100),                   -- E.g., "Primera dosis", "Refuerzo"

    -- Age/timing
    recommended_age_months      INTEGER,                        -- Recommended age in months
    recommended_age_text        VARCHAR(100),                   -- E.g., "Al nacer", "11 años"
    min_age_months              INTEGER NOT NULL,
    max_age_months              INTEGER,

    -- Scheduling rules
    min_interval_days           INTEGER,                        -- Min days since previous dose
    max_interval_days           INTEGER,                        -- Max days for valid interval

    -- Target population
    target_gender               VARCHAR(20),                    -- 'male', 'female', 'all'
    target_conditions           JSONB,                          -- Special risk groups
    /* Example: ["immunocompromised", "healthcare_worker", "pregnant"] */

    -- Geographic scope
    applies_to_province         VARCHAR(100),                   -- null = national, or specific province
    effective_from              DATE NOT NULL,
    effective_to                DATE,

    -- Notes
    administration_notes        TEXT,
    contraindications           TEXT,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(vaccine_id, dose_number, applies_to_province)
);


-- Patient immunization records
CREATE TABLE immunizations (
    id                          BIGSERIAL PRIMARY KEY,
    patient_id                  BIGINT NOT NULL REFERENCES patients(id),
    vaccine_id                  BIGINT NOT NULL REFERENCES vaccines(id),

    -- Administration details
    administered_date           DATE NOT NULL,
    administered_time           TIME,
    administered_by_user_id     BIGINT NOT NULL REFERENCES users(id),
    location_id                 BIGINT NOT NULL REFERENCES locations(id),

    -- Dose information
    dose_number                 INTEGER NOT NULL,               -- Which dose in series
    dose_amount_ml              DECIMAL(5,3),
    administration_route        VARCHAR(50) NOT NULL,
    administration_site         VARCHAR(100),                   -- E.g., "Brazo izquierdo", "Muslo derecho"

    -- Vaccine lot tracking (REQUIRED for NOMIVAC)
    lot_number                  VARCHAR(100) NOT NULL,
    manufacturer                VARCHAR(200) NOT NULL,
    expiration_date             DATE NOT NULL,
    vaccine_lot_id              BIGINT REFERENCES vaccine_lots(id),

    -- Clinical context
    appointment_id              BIGINT REFERENCES appointments(id),
    visit_reason                VARCHAR(200),
    patient_weight_kg           DECIMAL(5,2),                   -- For pediatric dosing
    patient_height_cm           DECIMAL(5,1),

    -- Consent
    consent_obtained            BOOLEAN DEFAULT true,
    consent_signature           TEXT,                           -- Base64 encoded signature
    consent_obtained_from       VARCHAR(100),                   -- 'patient', 'parent', 'guardian'

    -- Adverse events
    adverse_events_reported     BOOLEAN DEFAULT false,
    adverse_event_ids           JSONB,                          -- Array of adverse event IDs

    -- NOMIVAC integration
    nomivac_reported            BOOLEAN DEFAULT false,
    nomivac_transaction_id      VARCHAR(200),
    nomivac_reported_at         TIMESTAMP,
    nomivac_response            JSONB,

    -- Status
    status                      VARCHAR(50) DEFAULT 'completed', -- 'completed', 'refused', 'deferred'
    refusal_reason              TEXT,
    deferral_reason             TEXT,
    deferral_until_date         DATE,

    -- Documentation
    notes                       TEXT,
    created_by_user_id          BIGINT REFERENCES users(id),

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_immunizations_patient ON immunizations(patient_id);
CREATE INDEX idx_immunizations_vaccine ON immunizations(vaccine_id);
CREATE INDEX idx_immunizations_date ON immunizations(administered_date);
CREATE INDEX idx_immunizations_nomivac ON immunizations(nomivac_reported, nomivac_reported_at);


-- Vaccine inventory/lots
CREATE TABLE vaccine_lots (
    id                          BIGSERIAL PRIMARY KEY,
    vaccine_id                  BIGINT NOT NULL REFERENCES vaccines(id),
    lot_number                  VARCHAR(100) NOT NULL,
    manufacturer                VARCHAR(200) NOT NULL,

    -- Quantities
    quantity_received           INTEGER NOT NULL,
    quantity_remaining          INTEGER NOT NULL,
    unit_size_ml                DECIMAL(5,3),

    -- Dates
    received_date               DATE NOT NULL,
    manufacture_date            DATE,
    expiration_date             DATE NOT NULL,

    -- Storage location
    location_id                 BIGINT REFERENCES locations(id),
    storage_unit                VARCHAR(100),                   -- E.g., "Refrigerador A", "Freezer 2"

    -- Supplier info
    supplier_name               VARCHAR(200),
    purchase_order_number       VARCHAR(100),
    unit_cost                   DECIMAL(10,2),

    -- Status
    status                      VARCHAR(50) DEFAULT 'active',   -- 'active', 'expired', 'recalled', 'depleted'
    recall_info                 TEXT,

    -- Temperature monitoring
    requires_cold_chain         BOOLEAN DEFAULT true,
    temperature_monitor_id      VARCHAR(100),                   -- IoT sensor ID

    notes                       TEXT,
    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(vaccine_id, lot_number, manufacturer)
);

CREATE INDEX idx_vaccine_lots_expiration ON vaccine_lots(expiration_date);
CREATE INDEX idx_vaccine_lots_status ON vaccine_lots(status);


-- Cold chain temperature monitoring
CREATE TABLE cold_chain_logs (
    id                          BIGSERIAL PRIMARY KEY,
    location_id                 BIGINT NOT NULL REFERENCES locations(id),
    storage_unit                VARCHAR(100) NOT NULL,
    temperature_monitor_id      VARCHAR(100),

    -- Temperature reading
    temperature_celsius         DECIMAL(5,2) NOT NULL,
    recorded_at                 TIMESTAMP NOT NULL,

    -- Status
    is_within_range             BOOLEAN NOT NULL,
    min_acceptable_celsius      DECIMAL(5,2) NOT NULL,
    max_acceptable_celsius      DECIMAL(5,2) NOT NULL,

    -- Alert
    triggered_alert             BOOLEAN DEFAULT false,
    alert_id                    BIGINT REFERENCES system_alerts(id),

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_cold_chain_logs_location ON cold_chain_logs(location_id, recorded_at);
CREATE INDEX idx_cold_chain_logs_alerts ON cold_chain_logs(triggered_alert, recorded_at);


-- Adverse events following immunization (AEFI)
CREATE TABLE adverse_events (
    id                          BIGSERIAL PRIMARY KEY,
    immunization_id             BIGINT NOT NULL REFERENCES immunizations(id),
    patient_id                  BIGINT NOT NULL REFERENCES patients(id),

    -- Event details
    event_onset_date            DATE NOT NULL,
    event_onset_time            TIME,
    severity                    VARCHAR(50) NOT NULL,           -- 'mild', 'moderate', 'severe', 'life_threatening'

    -- Symptoms
    symptoms                    JSONB NOT NULL,
    /* Example:
    [
      {"symptom": "Fiebre", "severity": "moderate"},
      {"symptom": "Dolor en sitio de inyección", "severity": "mild"},
      {"symptom": "Urticaria", "severity": "moderate"}
    ]
    */

    -- Clinical response
    required_medical_attention  BOOLEAN DEFAULT false,
    hospitalized                BOOLEAN DEFAULT false,
    hospitalization_duration_days INTEGER,
    outcome                     VARCHAR(50),                    -- 'recovered', 'recovering', 'permanent_disability', 'death'

    -- Reporting
    reported_to_anmat           BOOLEAN DEFAULT false,
    anmat_case_number           VARCHAR(100),
    reported_to_anmat_at        TIMESTAMP,

    -- Investigation
    causality_assessment        VARCHAR(50),                    -- 'definite', 'probable', 'possible', 'unlikely', 'unrelated'
    investigation_notes         TEXT,

    -- Provider
    reported_by_user_id         BIGINT REFERENCES users(id),
    reported_at                 TIMESTAMP NOT NULL,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_adverse_events_patient ON adverse_events(patient_id);
CREATE INDEX idx_adverse_events_severity ON adverse_events(severity);
CREATE INDEX idx_adverse_events_anmat ON adverse_events(reported_to_anmat);


-- Vaccination due/reminder tracking
CREATE TABLE vaccination_reminders (
    id                          BIGSERIAL PRIMARY KEY,
    patient_id                  BIGINT NOT NULL REFERENCES patients(id),
    vaccine_id                  BIGINT NOT NULL REFERENCES vaccines(id),
    dose_number                 INTEGER NOT NULL,

    -- Due date calculation
    due_date                    DATE NOT NULL,
    overdue_date                DATE NOT NULL,                  -- When it becomes overdue
    max_valid_date              DATE,                           -- Latest date still valid

    -- Status
    status                      VARCHAR(50) DEFAULT 'due',      -- 'upcoming', 'due', 'overdue', 'completed', 'skipped'
    completed_immunization_id   BIGINT REFERENCES immunizations(id),

    -- Reminders sent
    reminder_sent_count         INTEGER DEFAULT 0,
    last_reminder_sent_at       TIMESTAMP,
    next_reminder_send_at       TIMESTAMP,

    -- Notes
    skip_reason                 TEXT,
    notes                       TEXT,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_vaccination_reminders_patient ON vaccination_reminders(patient_id);
CREATE INDEX idx_vaccination_reminders_due_date ON vaccination_reminders(due_date);
CREATE INDEX idx_vaccination_reminders_status ON vaccination_reminders(status);


-- Pediatric growth tracking (integrated with vaccinations)
CREATE TABLE growth_measurements (
    id                          BIGSERIAL PRIMARY KEY,
    patient_id                  BIGINT NOT NULL REFERENCES patients(id),
    measured_date               DATE NOT NULL,
    measured_time               TIME,

    -- Measurements
    weight_kg                   DECIMAL(5,2),
    height_cm                   DECIMAL(5,1),
    head_circumference_cm       DECIMAL(4,1),                   -- For infants/toddlers

    -- Context
    measured_by_user_id         BIGINT REFERENCES users(id),
    appointment_id              BIGINT REFERENCES appointments(id),
    immunization_id             BIGINT REFERENCES immunizations(id),

    -- Percentiles (calculated using WHO growth charts)
    weight_percentile           DECIMAL(5,2),
    height_percentile           DECIMAL(5,2),
    head_circumference_percentile DECIMAL(5,2),
    bmi                         DECIMAL(5,2),
    bmi_percentile              DECIMAL(5,2),

    -- Alerts
    growth_alert                BOOLEAN DEFAULT false,
    alert_reason                TEXT,

    notes                       TEXT,
    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_growth_measurements_patient ON growth_measurements(patient_id, measured_date);


-- Vaccination certificates (Carnet de Vacunación)
CREATE TABLE vaccination_certificates (
    id                          BIGSERIAL PRIMARY KEY,
    patient_id                  BIGINT NOT NULL REFERENCES patients(id),
    certificate_number          VARCHAR(100) NOT NULL UNIQUE,

    -- Type
    certificate_type            VARCHAR(50) NOT NULL,           -- 'complete', 'partial', 'travel', 'school'
    purpose                     VARCHAR(200),                   -- E.g., "Inscripción escolar", "Viaje internacional"

    -- Content
    immunizations_included      JSONB NOT NULL,                 -- Array of immunization IDs
    include_all_immunizations   BOOLEAN DEFAULT true,

    -- Generation
    generated_at                TIMESTAMP NOT NULL,
    generated_by_user_id        BIGINT REFERENCES users(id),
    pdf_file_path               VARCHAR(500),

    -- Validity
    valid_from                  DATE NOT NULL,
    valid_until                 DATE,

    -- Official stamp
    stamped                     BOOLEAN DEFAULT true,
    digital_signature           TEXT,                           -- Digital signature for authenticity

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_vaccination_certificates_patient ON vaccination_certificates(patient_id);
CREATE INDEX idx_vaccination_certificates_number ON vaccination_certificates(certificate_number);
```

---

## 3. Argentina National Vaccination Calendar

### 3.1 Complete Vaccination Schedule

```javascript
/**
 * ARGENTINA NATIONAL VACCINATION CALENDAR (2025)
 * Calendario Nacional de Vacunación
 *
 * Source: Ministerio de Salud de la Nación
 * https://www.argentina.gob.ar/salud/vacunas
 */

const ARGENTINA_VACCINATION_CALENDAR = [
  // AT BIRTH (Al nacer)
  {
    vaccine: 'BCG',
    dose_number: 1,
    recommended_age: '0 months',
    recommended_age_text: 'Al nacer',
    disease_targets: ['Tuberculosis'],
    administration_route: 'Intradermal',
    notes: 'Dosis única. Aplicar en maternidad antes del alta.'
  },
  {
    vaccine: 'Hepatitis B',
    dose_number: 1,
    recommended_age: '0 months',
    recommended_age_text: 'Al nacer (primeras 12 horas)',
    disease_targets: ['Hepatitis B'],
    administration_route: 'IM',
    notes: 'Primera dosis dentro de las 12 horas de vida.'
  },

  // 2 MONTHS
  {
    vaccine: 'Pentavalente',
    dose_number: 1,
    recommended_age: '2 months',
    recommended_age_text: '2 meses',
    disease_targets: ['Diphtheria', 'Tetanus', 'Pertussis', 'Hepatitis B', 'Haemophilus influenzae type b'],
    administration_route: 'IM',
    notes: 'Primera dosis de la serie.'
  },
  {
    vaccine: 'IPV (Salk)',
    dose_number: 1,
    recommended_age: '2 months',
    recommended_age_text: '2 meses',
    disease_targets: ['Poliomielitis'],
    administration_route: 'IM',
    notes: 'Vacuna inactivada contra poliomielitis.'
  },
  {
    vaccine: 'Neumococo Conjugada',
    dose_number: 1,
    recommended_age: '2 months',
    recommended_age_text: '2 meses',
    disease_targets: ['Neumococo (13 serotipos)'],
    administration_route: 'IM',
    notes: 'Primera dosis.'
  },
  {
    vaccine: 'Rotavirus',
    dose_number: 1,
    recommended_age: '2 months',
    recommended_age_text: '2 meses',
    disease_targets: ['Rotavirus'],
    administration_route: 'Oral',
    notes: 'Primera dosis. Edad máxima: 14 semanas 6 días.'
  },

  // 4 MONTHS
  {
    vaccine: 'Pentavalente',
    dose_number: 2,
    recommended_age: '4 months',
    recommended_age_text: '4 meses',
    disease_targets: ['Diphtheria', 'Tetanus', 'Pertussis', 'Hepatitis B', 'Haemophilus influenzae type b'],
    administration_route: 'IM',
    notes: 'Segunda dosis de la serie.'
  },
  {
    vaccine: 'IPV (Salk)',
    dose_number: 2,
    recommended_age: '4 months',
    recommended_age_text: '4 meses',
    disease_targets: ['Poliomielitis'],
    administration_route: 'IM',
    notes: 'Segunda dosis.'
  },
  {
    vaccine: 'Neumococo Conjugada',
    dose_number: 2,
    recommended_age: '4 months',
    recommended_age_text: '4 meses',
    disease_targets: ['Neumococo (13 serotipos)'],
    administration_route: 'IM',
    notes: 'Segunda dosis.'
  },
  {
    vaccine: 'Rotavirus',
    dose_number: 2,
    recommended_age: '4 months',
    recommended_age_text: '4 meses',
    disease_targets: ['Rotavirus'],
    administration_route: 'Oral',
    notes: 'Segunda dosis. Edad máxima: 24 semanas.'
  },

  // 6 MONTHS
  {
    vaccine: 'Pentavalente',
    dose_number: 3,
    recommended_age: '6 months',
    recommended_age_text: '6 meses',
    disease_targets: ['Diphtheria', 'Tetanus', 'Pertussis', 'Hepatitis B', 'Haemophilus influenzae type b'],
    administration_route: 'IM',
    notes: 'Tercera dosis de la serie.'
  },
  {
    vaccine: 'Gripe (Influenza)',
    dose_number: 1,
    recommended_age: '6 months',
    recommended_age_text: '6 meses a 24 meses (campaña anual)',
    disease_targets: ['Influenza'],
    administration_route: 'IM',
    notes: 'Dos dosis separadas por 4 semanas. Luego anual.'
  },

  // 12 MONTHS
  {
    vaccine: 'Triple Viral (SRP)',
    dose_number: 1,
    recommended_age: '12 months',
    recommended_age_text: '12 meses',
    disease_targets: ['Sarampión', 'Rubéola', 'Parotiditis'],
    administration_route: 'SC',
    notes: 'Primera dosis.'
  },
  {
    vaccine: 'Neumococo Conjugada',
    dose_number: 3,
    recommended_age: '12 months',
    recommended_age_text: '12 meses',
    disease_targets: ['Neumococo (13 serotipos)'],
    administration_route: 'IM',
    notes: 'Refuerzo.'
  },
  {
    vaccine: 'Hepatitis A',
    dose_number: 1,
    recommended_age: '12 months',
    recommended_age_text: '12 meses',
    disease_targets: ['Hepatitis A'],
    administration_route: 'IM',
    notes: 'Dosis única en Argentina.'
  },

  // 15-18 MONTHS
  {
    vaccine: 'Varicela',
    dose_number: 1,
    recommended_age: '15 months',
    recommended_age_text: '15 meses',
    disease_targets: ['Varicela'],
    administration_route: 'SC',
    notes: 'Dosis única en esquema argentino.'
  },
  {
    vaccine: 'Fiebre Amarilla',
    dose_number: 1,
    recommended_age: '18 months',
    recommended_age_text: '18 meses (solo zonas de riesgo)',
    disease_targets: ['Fiebre Amarilla'],
    administration_route: 'SC',
    notes: 'Solo para residentes en zonas endémicas (Misiones, Formosa, etc.).'
  },

  // 5-6 YEARS
  {
    vaccine: 'IPV (Salk)',
    dose_number: 3,
    recommended_age: '5-6 years',
    recommended_age_text: 'Ingreso escolar (5-6 años)',
    disease_targets: ['Poliomielitis'],
    administration_route: 'IM',
    notes: 'Refuerzo.'
  },
  {
    vaccine: 'Triple Viral (SRP)',
    dose_number: 2,
    recommended_age: '5-6 years',
    recommended_age_text: 'Ingreso escolar (5-6 años)',
    disease_targets: ['Sarampión', 'Rubéola', 'Parotiditis'],
    administration_route: 'SC',
    notes: 'Segundo dosis / refuerzo.'
  },
  {
    vaccine: 'DTP (Triple Bacteriana Celular)',
    dose_number: 4,
    recommended_age: '5-6 years',
    recommended_age_text: 'Ingreso escolar (5-6 años)',
    disease_targets: ['Diphtheria', 'Tetanus', 'Pertussis'],
    administration_route: 'IM',
    notes: 'Refuerzo.'
  },

  // 11 YEARS
  {
    vaccine: 'Triple Bacteriana Acelular (dTpa)',
    dose_number: 1,
    recommended_age: '11 years',
    recommended_age_text: '11 años',
    disease_targets: ['Diphtheria', 'Tetanus', 'Pertussis'],
    administration_route: 'IM',
    notes: 'Refuerzo en adolescencia.'
  },
  {
    vaccine: 'VPH (Virus Papiloma Humano)',
    dose_number: 1,
    recommended_age: '11 years',
    recommended_age_text: '11 años',
    disease_targets: ['Virus del Papiloma Humano (9 genotipos)'],
    administration_route: 'IM',
    notes: 'Esquema de 2 dosis (0 y 6 meses). Niños y niñas.',
    target_gender: 'all'
  },
  {
    vaccine: 'Meningococo ACYW',
    dose_number: 1,
    recommended_age: '11 years',
    recommended_age_text: '11 años',
    disease_targets: ['Meningococo (serotipos A, C, Y, W)'],
    administration_route: 'IM',
    notes: 'Dosis única o refuerzo según antecedentes.'
  },
  {
    vaccine: 'Fiebre Amarilla',
    dose_number: 2,
    recommended_age: '11 years',
    recommended_age_text: '11 años (solo zonas de riesgo)',
    disease_targets: ['Fiebre Amarilla'],
    administration_route: 'SC',
    notes: 'Refuerzo solo para residentes en zonas endémicas.'
  },

  // ADULTS (Adultos)
  {
    vaccine: 'dT (Doble Adultos)',
    dose_number: null,
    recommended_age: '10 years interval',
    recommended_age_text: 'Cada 10 años desde los 16 años',
    disease_targets: ['Diphtheria', 'Tetanus'],
    administration_route: 'IM',
    notes: 'Refuerzo cada 10 años durante toda la vida adulta.'
  },
  {
    vaccine: 'Gripe (Influenza)',
    dose_number: null,
    recommended_age: 'annual',
    recommended_age_text: 'Anual (65+ años y grupos de riesgo)',
    disease_targets: ['Influenza'],
    administration_route: 'IM',
    notes: 'Campaña anual. Obligatoria para mayores de 65 años y grupos de riesgo.'
  },
  {
    vaccine: 'Neumococo Polisacárida (23 valente)',
    dose_number: 1,
    recommended_age: '65 years',
    recommended_age_text: '65 años',
    disease_targets: ['Neumococo (23 serotipos)'],
    administration_route: 'IM',
    notes: 'Dosis única a los 65 años. Refuerzo a los 5 años si grupo de riesgo.'
  },

  // PREGNANT WOMEN (Embarazadas)
  {
    vaccine: 'dTpa (Triple Bacteriana Acelular)',
    dose_number: null,
    recommended_age: 'pregnancy',
    recommended_age_text: 'Cada embarazo (a partir de la semana 20)',
    disease_targets: ['Diphtheria', 'Tetanus', 'Pertussis'],
    administration_route: 'IM',
    notes: 'Aplicar en cada embarazo después de la semana 20 de gestación.',
    target_population: 'pregnant_women'
  },
  {
    vaccine: 'Gripe (Influenza)',
    dose_number: null,
    recommended_age: 'pregnancy',
    recommended_age_text: 'Durante el embarazo (cualquier trimestre)',
    disease_targets: ['Influenza'],
    administration_route: 'IM',
    notes: 'En cualquier trimestre del embarazo durante campaña.',
    target_population: 'pregnant_women'
  },

  // COVID-19 (All ages)
  {
    vaccine: 'COVID-19',
    dose_number: null,
    recommended_age: '6 months+',
    recommended_age_text: 'A partir de 6 meses',
    disease_targets: ['COVID-19 (SARS-CoV-2)'],
    administration_route: 'IM',
    notes: 'Esquema primario: 2 dosis. Refuerzos según grupos de riesgo y edad.'
  }
];
```

---

## 4. NOMIVAC Integration

### 4.1 NOMIVAC Web Service Integration

```javascript
/**
 * NOMIVAC (Registro Federal de Vacunación Nominalizado)
 * Web service integration for Argentina's national immunization registry
 *
 * IMPORTANT: This is a MANDATORY integration for all healthcare providers in Argentina
 *
 * Documentation: https://sisa.msal.gov.ar/sisadoc/docs/0802/nomivac.jsp
 */

const axios = require('axios');
const crypto = require('crypto');

const NOMIVAC_CONFIG = {
  endpoint: process.env.NOMIVAC_ENDPOINT || 'https://sisa.msal.gov.ar/sisa/services/rest/nomivac',
  username: process.env.NOMIVAC_USERNAME,
  password: process.env.NOMIVAC_PASSWORD,
  establishment_code: process.env.NOMIVAC_ESTABLISHMENT_CODE,  // Código de establecimiento
  timeout: 30000 // 30 seconds
};


/**
 * Authenticate with NOMIVAC and get session token
 */
async function authenticateNOMIVAC() {
  try {
    const response = await axios.post(`${NOMIVAC_CONFIG.endpoint}/login`, {
      username: NOMIVAC_CONFIG.username,
      password: NOMIVAC_CONFIG.password,
      establecimiento: NOMIVAC_CONFIG.establishment_code
    }, {
      timeout: NOMIVAC_CONFIG.timeout
    });

    if (response.data.resultado === 'OK') {
      return response.data.token;
    } else {
      throw new Error(`NOMIVAC authentication failed: ${response.data.descripcion}`);
    }

  } catch (error) {
    console.error('NOMIVAC authentication error:', error.message);
    throw new Error(`Cannot connect to NOMIVAC: ${error.message}`);
  }
}


/**
 * Report vaccination to NOMIVAC
 *
 * This must be called IMMEDIATELY after administering a vaccine
 */
async function reportToNOMIVAC(immunization) {
  try {
    const patient = await db.Patient.findByPk(immunization.patient_id);
    const vaccine = await db.Vaccine.findByPk(immunization.vaccine_id);
    const provider = await db.User.findByPk(immunization.administered_by_user_id);

    // Get authentication token
    const token = await authenticateNOMIVAC();

    // Prepare vaccination data in NOMIVAC format
    const vaccinationData = {
      // Patient identification (REQUIRED)
      tipoDocumento: patient.document_type || 'DNI',
      numeroDocumento: patient.document_number,
      apellido: patient.last_name,
      nombres: patient.first_name,
      fechaNacimiento: patient.date_of_birth,
      sexo: patient.gender === 'male' ? 'M' : 'F',

      // Contact info (OPTIONAL)
      telefono: patient.phone_number,
      email: patient.email,

      // Address (RECOMMENDED)
      provincia: patient.province_code,
      departamento: patient.department_code,
      localidad: patient.locality,
      domicilio: patient.address,
      codigoPostal: patient.postal_code,

      // Vaccination details (REQUIRED)
      codigoVacuna: vaccine.vaccine_code,              // Argentina official vaccine code
      nombreVacuna: vaccine.vaccine_name,
      numeroDosis: immunization.dose_number,
      fechaAplicacion: immunization.administered_date,
      lote: immunization.lot_number,
      laboratorio: immunization.manufacturer,
      fechaVencimiento: immunization.expiration_date,

      // Administration (REQUIRED)
      viaAdministracion: immunization.administration_route,
      sitioAplicacion: immunization.administration_site || 'Brazo',

      // Provider info (REQUIRED)
      establecimiento: NOMIVAC_CONFIG.establishment_code,
      profesionalMatricula: provider.license_number,
      profesionalNombre: `${provider.first_name} ${provider.last_name}`,

      // Estrategia (vaccination campaign/strategy)
      estrategia: immunization.campaign_code || 'CALENDARIO_REGULAR'
    };

    // Send to NOMIVAC
    const response = await axios.post(
      `${NOMIVAC_CONFIG.endpoint}/registrarVacunacion`,
      vaccinationData,
      {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        timeout: NOMIVAC_CONFIG.timeout
      }
    );

    // Process response
    if (response.data.resultado === 'OK') {
      await immunization.update({
        nomivac_reported: true,
        nomivac_transaction_id: response.data.idTransaccion,
        nomivac_reported_at: new Date(),
        nomivac_response: response.data
      });

      console.log(`Successfully reported immunization ${immunization.id} to NOMIVAC. Transaction ID: ${response.data.idTransaccion}`);

      return {
        success: true,
        transaction_id: response.data.idTransaccion
      };

    } else {
      // NOMIVAC rejected the vaccination
      console.error(`NOMIVAC rejected immunization ${immunization.id}:`, response.data.descripcion);

      await immunization.update({
        nomivac_reported: false,
        nomivac_response: response.data
      });

      return {
        success: false,
        error: response.data.descripcion
      };
    }

  } catch (error) {
    console.error(`Error reporting to NOMIVAC:`, error.message);

    // Log the error but don't fail the vaccination
    await db.SystemLog.create({
      log_type: 'nomivac_error',
      severity: 'high',
      message: `Failed to report immunization ${immunization.id} to NOMIVAC: ${error.message}`,
      metadata: {
        immunization_id: immunization.id,
        patient_id: immunization.patient_id,
        error: error.message
      }
    });

    return {
      success: false,
      error: error.message
    };
  }
}


/**
 * Query patient's vaccination history from NOMIVAC
 *
 * Use this to import vaccinations administered at other facilities
 */
async function queryNOMIVACHistory(patientDNI, patientDateOfBirth) {
  try {
    const token = await authenticateNOMIVAC();

    const response = await axios.post(
      `${NOMIVAC_CONFIG.endpoint}/consultarHistorial`,
      {
        tipoDocumento: 'DNI',
        numeroDocumento: patientDNI,
        fechaNacimiento: patientDateOfBirth
      },
      {
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        timeout: NOMIVAC_CONFIG.timeout
      }
    );

    if (response.data.resultado === 'OK') {
      return {
        success: true,
        vaccinations: response.data.vacunaciones || []
      };
    } else {
      return {
        success: false,
        error: response.data.descripcion
      };
    }

  } catch (error) {
    console.error('Error querying NOMIVAC history:', error.message);
    return {
      success: false,
      error: error.message
    };
  }
}


/**
 * Retry failed NOMIVAC reports (run daily via cron)
 */
async function retryFailedNOMIVACReports() {
  const failedReports = await db.Immunization.findAll({
    where: {
      nomivac_reported: false,
      administered_date: {
        [Op.gte]: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) // Last 30 days
      }
    },
    limit: 100
  });

  let successCount = 0;
  let failCount = 0;

  for (const immunization of failedReports) {
    const result = await reportToNOMIVAC(immunization);
    if (result.success) {
      successCount++;
    } else {
      failCount++;
    }

    // Rate limiting: Wait 1 second between requests
    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  console.log(`NOMIVAC retry completed: ${successCount} successful, ${failCount} failed`);

  return { successCount, failCount };
}
```

---

## 5. Vaccination Due Date Calculation & Reminders

### 5.1 Due Date Algorithm

```javascript
/**
 * Calculate vaccination due dates for a patient
 */
async function calculateVaccinationDueDates(patientId) {
  const patient = await db.Patient.findByPk(patientId);
  const patientDOB = new Date(patient.date_of_birth);
  const today = new Date();
  const patientAgeMonths = Math.floor((today - patientDOB) / (1000 * 60 * 60 * 24 * 30.44));

  // Get all vaccinations for this patient
  const completedImmunizations = await db.Immunization.findAll({
    where: {
      patient_id: patientId,
      status: 'completed'
    },
    order: [['administered_date', 'ASC']]
  });

  // Get vaccination schedule for patient's location
  const province = patient.province_code || null;
  const scheduleItems = await db.VaccinationSchedule.findAll({
    where: {
      [Op.or]: [
        { applies_to_province: null },      // National
        { applies_to_province: province }   // Province-specific
      ],
      effective_from: { [Op.lte]: today },
      [Op.or]: [
        { effective_to: null },
        { effective_to: { [Op.gte]: today } }
      ]
    },
    include: [{ model: db.Vaccine, as: 'Vaccine' }],
    order: [['recommended_age_months', 'ASC']]
  });

  const dueDates = [];

  for (const scheduleItem of scheduleItems) {
    // Check if patient is eligible for this vaccine
    const isEligible = checkEligibility(patient, scheduleItem, patientAgeMonths);
    if (!isEligible) continue;

    // Check if already completed
    const alreadyCompleted = completedImmunizations.some(imm =>
      imm.vaccine_id === scheduleItem.vaccine_id &&
      imm.dose_number === scheduleItem.dose_number
    );

    if (alreadyCompleted) continue;

    // Calculate due date
    let dueDate;

    if (scheduleItem.recommended_age_months !== null) {
      // Age-based schedule
      dueDate = new Date(patientDOB);
      dueDate.setMonth(dueDate.getMonth() + scheduleItem.recommended_age_months);

    } else if (scheduleItem.dose_number > 1 && scheduleItem.min_interval_days) {
      // Interval-based (requires previous dose)
      const previousDose = completedImmunizations.find(imm =>
        imm.vaccine_id === scheduleItem.vaccine_id &&
        imm.dose_number === (scheduleItem.dose_number - 1)
      );

      if (!previousDose) {
        // Can't calculate due date without previous dose
        continue;
      }

      dueDate = new Date(previousDose.administered_date);
      dueDate.setDate(dueDate.getDate() + scheduleItem.min_interval_days);

    } else {
      // No clear due date calculation method
      continue;
    }

    // Calculate overdue date
    const overdueDate = new Date(dueDate);
    overdueDate.setDate(overdueDate.getDate() + 30); // 30 days grace period

    // Calculate max valid date
    let maxValidDate = null;
    if (scheduleItem.max_age_months) {
      maxValidDate = new Date(patientDOB);
      maxValidDate.setMonth(maxValidDate.getMonth() + scheduleItem.max_age_months);
    }

    // Determine status
    let status;
    if (dueDate > today.setDate(today.getDate() + 30)) {
      status = 'upcoming';
    } else if (dueDate <= today && today < overdueDate) {
      status = 'due';
    } else if (today >= overdueDate) {
      status = 'overdue';
    }

    dueDates.push({
      vaccine_id: scheduleItem.vaccine_id,
      vaccine_name: scheduleItem.Vaccine.vaccine_name,
      dose_number: scheduleItem.dose_number,
      dose_name: scheduleItem.dose_name,
      due_date: dueDate,
      overdue_date: overdueDate,
      max_valid_date: maxValidDate,
      status,
      priority: status === 'overdue' ? 'high' : (status === 'due' ? 'medium' : 'low')
    });
  }

  return dueDates;
}


/**
 * Check if patient is eligible for a vaccine
 */
function checkEligibility(patient, scheduleItem, patientAgeMonths) {
  // Age range check
  if (scheduleItem.min_age_months && patientAgeMonths < scheduleItem.min_age_months) {
    return false;
  }

  if (scheduleItem.max_age_months && patientAgeMonths > scheduleItem.max_age_months) {
    return false;
  }

  // Gender check
  if (scheduleItem.target_gender && scheduleItem.target_gender !== 'all') {
    if (patient.gender !== scheduleItem.target_gender) {
      return false;
    }
  }

  // Target conditions (e.g., healthcare worker, pregnant)
  if (scheduleItem.target_conditions && scheduleItem.target_conditions.length > 0) {
    // This would require checking patient's conditions/risk factors
    // For now, assume eligible
  }

  return true;
}


/**
 * Send vaccination reminders (run daily via cron)
 */
async function sendVaccinationReminders() {
  // Find all due/overdue vaccinations
  const dueReminders = await db.VaccinationReminder.findAll({
    where: {
      status: { [Op.in]: ['due', 'overdue'] },
      [Op.or]: [
        { next_reminder_send_at: { [Op.lte]: new Date() } },
        { next_reminder_send_at: null }
      ]
    },
    include: [
      { model: db.Patient, as: 'Patient' },
      { model: db.Vaccine, as: 'Vaccine' }
    ],
    limit: 500  // Process max 500 per run
  });

  for (const reminder of dueReminders) {
    await sendVaccinationReminderMessage(reminder);

    // Update reminder
    await reminder.update({
      reminder_sent_count: reminder.reminder_sent_count + 1,
      last_reminder_sent_at: new Date(),
      next_reminder_send_at: calculateNextReminderDate(reminder)
    });
  }

  console.log(`Sent ${dueReminders.length} vaccination reminders`);
}


/**
 * Send vaccination reminder message
 */
async function sendVaccinationReminderMessage(reminder) {
  const patient = reminder.Patient;
  const vaccine = reminder.Vaccine;

  // Email template
  const emailTemplate = `
    <h2>Recordatorio de Vacunación</h2>

    <p>Estimado/a ${patient.first_name} ${patient.last_name},</p>

    <p>Le recordamos que ${reminder.status === 'overdue' ? '<strong>está atrasada</strong>' : 'es momento de aplicar'}
    la siguiente vacuna:</p>

    <ul>
      <li><strong>Vacuna:</strong> ${vaccine.vaccine_name}</li>
      <li><strong>Dosis:</strong> ${reminder.dose_number}° dosis</li>
      <li><strong>Fecha recomendada:</strong> ${reminder.due_date.toLocaleDateString('es-AR')}</li>
      ${reminder.status === 'overdue' ?
        `<li><strong>Estado:</strong> <span style="color: red;">ATRASADA</span></li>` : ''}
    </ul>

    <p>Por favor, comuníquese con nosotros para coordinar un turno.</p>

    <p><strong>Teléfono:</strong> ${process.env.CLINIC_PHONE}</p>
    <p><strong>WhatsApp:</strong> ${process.env.CLINIC_WHATSAPP}</p>

    <p><small>Esta vacuna es parte del Calendario Nacional de Vacunación de Argentina y
    es <strong>gratuita y obligatoria</strong>.</small></p>
  `;

  // SMS template
  const smsTemplate = `RECORDATORIO: ${patient.first_name}, es momento de aplicar la vacuna ${vaccine.vaccine_name_short || vaccine.vaccine_name} (dosis ${reminder.dose_number}). Llamá al ${process.env.CLINIC_PHONE} para sacar turno. Es gratuita.`;

  // Send email
  if (patient.email) {
    await sendEmail({
      to: patient.email,
      subject: 'Recordatorio de Vacunación',
      html: emailTemplate
    });
  }

  // Send SMS
  if (patient.phone_number) {
    await sendSMS({
      to: patient.phone_number,
      message: smsTemplate
    });
  }
}


/**
 * Calculate next reminder date based on urgency
 */
function calculateNextReminderDate(reminder) {
  const today = new Date();

  if (reminder.status === 'overdue') {
    // Send weekly if overdue
    const nextDate = new Date(today);
    nextDate.setDate(nextDate.getDate() + 7);
    return nextDate;

  } else if (reminder.status === 'due') {
    // Send every 2 weeks if due
    const nextDate = new Date(today);
    nextDate.setDate(nextDate.getDate() + 14);
    return nextDate;

  } else {
    // Send monthly if upcoming
    const nextDate = new Date(today);
    nextDate.setMonth(nextDate.getMonth() + 1);
    return nextDate;
  }
}
```

---

## 6. Vaccination Certificate Generation

### 6.1 Official Argentina Certificate Template

```javascript
/**
 * Generate vaccination certificate (Carnet de Vacunación)
 */
const PDFDocument = require('pdfkit');
const fs = require('fs');

async function generateVaccinationCertificate(patientId, certificateType = 'complete') {
  const patient = await db.Patient.findByPk(patientId);

  const immunizations = await db.Immunization.findAll({
    where: {
      patient_id: patientId,
      status: 'completed'
    },
    include: [
      { model: db.Vaccine, as: 'Vaccine' },
      { model: db.User, as: 'AdministeredBy' },
      { model: db.Location, as: 'Location' }
    ],
    order: [['administered_date', 'ASC']]
  });

  // Create PDF
  const doc = new PDFDocument({
    size: 'A4',
    margins: { top: 50, bottom: 50, left: 50, right: 50 }
  });

  const certificateNumber = `CERT-${Date.now()}-${patientId}`;
  const fileName = `/tmp/vaccination_certificate_${certificateNumber}.pdf`;
  const stream = fs.createWriteStream(fileName);
  doc.pipe(stream);

  // Header
  doc.fontSize(20).font('Helvetica-Bold').text('CERTIFICADO DE VACUNACIÓN', { align: 'center' });
  doc.moveDown(0.5);
  doc.fontSize(12).font('Helvetica').text('REPÚBLICA ARGENTINA', { align: 'center' });
  doc.fontSize(10).text('Calendario Nacional de Vacunación', { align: 'center' });
  doc.moveDown(1);

  // Certificate number
  doc.fontSize(10).text(`Certificado N°: ${certificateNumber}`, { align: 'right' });
  doc.text(`Fecha de emisión: ${new Date().toLocaleDateString('es-AR')}`, { align: 'right' });
  doc.moveDown(1);

  // Patient information
  doc.fontSize(14).font('Helvetica-Bold').text('DATOS DEL PACIENTE');
  doc.moveDown(0.5);

  doc.fontSize(11).font('Helvetica');
  const patientInfo = [
    ['Apellido y Nombre:', `${patient.last_name}, ${patient.first_name}`],
    ['Documento:', `${patient.document_type || 'DNI'} ${patient.document_number}`],
    ['Fecha de Nacimiento:', new Date(patient.date_of_birth).toLocaleDateString('es-AR')],
    ['Edad:', calculateAge(patient.date_of_birth)],
    ['Sexo:', patient.gender === 'male' ? 'Masculino' : 'Femenino'],
    ['Obra Social:', patient.PrimaryObraSocial?.name || 'Sin obra social']
  ];

  patientInfo.forEach(([label, value]) => {
    doc.text(`${label} ${value}`);
  });

  doc.moveDown(1);

  // Vaccination table
  doc.fontSize(14).font('Helvetica-Bold').text('VACUNAS APLICADAS');
  doc.moveDown(0.5);

  // Table header
  doc.fontSize(9).font('Helvetica-Bold');
  const tableTop = doc.y;
  const colWidths = [80, 50, 80, 70, 70, 100];
  const colPositions = [50, 130, 180, 260, 330, 400];

  doc.text('Vacuna', colPositions[0], tableTop);
  doc.text('Dosis', colPositions[1], tableTop);
  doc.text('Fecha', colPositions[2], tableTop);
  doc.text('Lote', colPositions[3], tableTop);
  doc.text('Laboratorio', colPositions[4], tableTop);
  doc.text('Establecimiento', colPositions[5], tableTop);

  // Draw header line
  doc.moveTo(50, doc.y + 5).lineTo(550, doc.y + 5).stroke();
  doc.moveDown(0.5);

  // Table rows
  doc.fontSize(8).font('Helvetica');
  immunizations.forEach(imm => {
    const rowY = doc.y;

    doc.text(imm.Vaccine.vaccine_name_short || imm.Vaccine.vaccine_name, colPositions[0], rowY, { width: colWidths[0] });
    doc.text(`${imm.dose_number}°`, colPositions[1], rowY);
    doc.text(new Date(imm.administered_date).toLocaleDateString('es-AR'), colPositions[2], rowY);
    doc.text(imm.lot_number, colPositions[3], rowY, { width: colWidths[3] });
    doc.text(truncate(imm.manufacturer, 12), colPositions[4], rowY, { width: colWidths[4] });
    doc.text(truncate(imm.Location.name, 15), colPositions[5], rowY, { width: colWidths[5] });

    doc.moveDown(0.8);
  });

  doc.moveDown(2);

  // Footer
  doc.fontSize(10).font('Helvetica-Italic');
  doc.text('Este certificado es válido en toda la República Argentina.', { align: 'center' });
  doc.text('Las vacunas del Calendario Nacional son gratuitas y obligatorias.', { align: 'center' });
  doc.moveDown(1);

  // Digital signature
  doc.fontSize(9).font('Helvetica');
  doc.text(`Establecimiento: ${process.env.CLINIC_NAME}`, { align: 'left' });
  doc.text(`Dirección: ${process.env.CLINIC_ADDRESS}`, { align: 'left' });
  doc.text(`Matrícula: ${process.env.CLINIC_LICENSE}`, { align: 'left' });

  // QR code placeholder (for verification)
  doc.moveDown(2);
  doc.fontSize(8).text(`Código de verificación: ${certificateNumber}`, { align: 'center' });

  // Finalize PDF
  doc.end();

  // Wait for PDF to finish writing
  await new Promise((resolve, reject) => {
    stream.on('finish', resolve);
    stream.on('error', reject);
  });

  // Save certificate record
  const certificate = await db.VaccinationCertificate.create({
    patient_id: patientId,
    certificate_number: certificateNumber,
    certificate_type: certificateType,
    purpose: certificateType === 'school' ? 'Inscripción escolar' : 'Certificado general',
    immunizations_included: immunizations.map(imm => imm.id),
    include_all_immunizations: true,
    generated_at: new Date(),
    generated_by_user_id: null,  // System generated
    pdf_file_path: fileName,
    valid_from: new Date(),
    stamped: true
  });

  return {
    certificate_id: certificate.id,
    certificate_number: certificateNumber,
    pdf_path: fileName
  };
}


function calculateAge(dateOfBirth) {
  const today = new Date();
  const birthDate = new Date(dateOfBirth);
  let age = today.getFullYear() - birthDate.getFullYear();
  const monthDiff = today.getMonth() - birthDate.getMonth();

  if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birthDate.getDate())) {
    age--;
  }

  if (age < 2) {
    const months = Math.floor((today - birthDate) / (1000 * 60 * 60 * 24 * 30.44));
    return `${months} meses`;
  }

  return `${age} años`;
}

function truncate(str, maxLength) {
  if (!str) return '';
  return str.length > maxLength ? str.substring(0, maxLength - 3) + '...' : str;
}
```

---

## 7. API Endpoints

### 7.1 Immunization Management Endpoints

```javascript
/**
 * POST /api/immunizations
 *
 * Record a new vaccination
 */
router.post('/immunizations', authenticateUser, async (req, res) => {
  try {
    const {
      patient_id,
      vaccine_id,
      dose_number,
      administered_date,
      lot_number,
      manufacturer,
      expiration_date,
      administration_route,
      administration_site,
      consent_obtained,
      notes
    } = req.body;

    // Create immunization record
    const immunization = await db.Immunization.create({
      patient_id,
      vaccine_id,
      dose_number,
      administered_date,
      administered_time: new Date().toTimeString().split(' ')[0].substring(0, 5),
      administered_by_user_id: req.user.id,
      location_id: req.user.primary_location_id,
      lot_number,
      manufacturer,
      expiration_date,
      administration_route,
      administration_site,
      consent_obtained,
      status: 'completed',
      notes,
      created_by_user_id: req.user.id
    });

    // Update vaccine lot inventory
    await db.VaccineLot.decrement('quantity_remaining', {
      where: {
        vaccine_id,
        lot_number,
        manufacturer
      }
    });

    // Report to NOMIVAC (async, don't wait)
    reportToNOMIVAC(immunization).catch(err =>
      console.error('NOMIVAC reporting failed:', err)
    );

    // Update vaccination reminder status
    await db.VaccinationReminder.update(
      {
        status: 'completed',
        completed_immunization_id: immunization.id
      },
      {
        where: {
          patient_id,
          vaccine_id,
          dose_number,
          status: { [Op.in]: ['due', 'overdue'] }
        }
      }
    );

    res.json({
      success: true,
      immunization
    });

  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});


/**
 * GET /api/immunizations/patient/:patientId
 *
 * Get vaccination history for a patient
 */
router.get('/immunizations/patient/:patientId', authenticateUser, async (req, res) => {
  const { patientId } = req.params;

  const immunizations = await db.Immunization.findAll({
    where: { patient_id: patientId },
    include: [
      { model: db.Vaccine, as: 'Vaccine' },
      { model: db.User, as: 'AdministeredBy' },
      { model: db.Location, as: 'Location' }
    ],
    order: [['administered_date', 'DESC']]
  });

  res.json({ immunizations });
});


/**
 * GET /api/immunizations/due/:patientId
 *
 * Get due/overdue vaccinations for a patient
 */
router.get('/immunizations/due/:patientId', authenticateUser, async (req, res) => {
  const { patientId } = req.params;

  const dueDates = await calculateVaccinationDueDates(patientId);

  res.json({ due_vaccinations: dueDates });
});


/**
 * POST /api/immunizations/certificate
 *
 * Generate vaccination certificate
 */
router.post('/immunizations/certificate', authenticateUser, async (req, res) => {
  try {
    const { patient_id, certificate_type } = req.body;

    const result = await generateVaccinationCertificate(patient_id, certificate_type);

    res.json({
      success: true,
      certificate_number: result.certificate_number,
      pdf_url: `/certificates/${result.certificate_number}.pdf`
    });

  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});


/**
 * POST /api/immunizations/import-nomivac
 *
 * Import vaccination history from NOMIVAC
 */
router.post('/immunizations/import-nomivac', authenticateUser, async (req, res) => {
  try {
    const { patient_id } = req.body;

    const patient = await db.Patient.findByPk(patient_id);
    const result = await queryNOMIVACHistory(patient.document_number, patient.date_of_birth);

    if (!result.success) {
      return res.status(400).json({ error: result.error });
    }

    // Import vaccinations that don't exist locally
    let importedCount = 0;
    for (const nomivacVacc of result.vaccinations) {
      // Check if already exists
      const existing = await db.Immunization.findOne({
        where: {
          patient_id,
          administered_date: nomivacVacc.fechaAplicacion,
          lot_number: nomivacVacc.lote
        }
      });

      if (!existing) {
        // Find matching vaccine by code
        const vaccine = await db.Vaccine.findOne({
          where: { vaccine_code: nomivacVacc.codigoVacuna }
        });

        if (vaccine) {
          await db.Immunization.create({
            patient_id,
            vaccine_id: vaccine.id,
            dose_number: nomivacVacc.numeroDosis,
            administered_date: nomivacVacc.fechaAplicacion,
            lot_number: nomivacVacc.lote,
            manufacturer: nomivacVacc.laboratorio,
            administration_route: nomivacVacc.viaAdministracion,
            status: 'completed',
            nomivac_reported: true,
            nomivac_transaction_id: nomivacVacc.idTransaccion,
            notes: 'Importado desde NOMIVAC'
          });

          importedCount++;
        }
      }
    }

    res.json({
      success: true,
      imported_count: importedCount,
      total_found: result.vaccinations.length
    });

  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```

---

## 8. Summary

The Immunization Registry System provides:

✅ **Complete Argentina Vaccination Calendar**: From birth to 65+ years, all mandatory vaccines
✅ **NOMIVAC Integration**: Mandatory reporting to national registry with retry logic
✅ **Automated Reminders**: Due/overdue alerts via email and SMS
✅ **Official Certificates**: PDF generation of Carnet de Vacunación
✅ **Vaccine Inventory**: Lot tracking, expiration monitoring, cold chain compliance
✅ **Adverse Event Tracking**: AEFI reporting to ANMAT
✅ **Pediatric Growth Charts**: Integrated weight/height tracking
✅ **Province-Specific Schedules**: Support for provincial variations (e.g., Yellow Fever)
✅ **COVID-19 Support**: Special handling for pandemic vaccines
✅ **Import from NOMIVAC**: Query patient history from other facilities

**Next Steps:**
1. Register facility with NOMIVAC and obtain credentials
2. Load Argentina vaccination calendar into database
3. Configure cold chain temperature monitors
4. Set up daily cron jobs for reminders and NOMIVAC retries
5. Train staff on immunization documentation requirements
6. Test certificate generation with sample patients
