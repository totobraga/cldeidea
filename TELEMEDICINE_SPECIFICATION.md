# Telemedicine & Virtual Consultations - Complete Specification

## Overview

This document provides comprehensive specifications for telemedicine and virtual consultation capabilities, including video conferencing, scheduling, billing, compliance, and integration with the clinical management system. Designed for Argentina healthcare market with Obra Social/PAMI coverage considerations.

---

## 1. TELEMEDICINE ARCHITECTURE

### 1.1 Database Schema

```sql
-- Telemedicine sessions
CREATE TABLE telemedicine_sessions (
    id                      BIGSERIAL PRIMARY KEY,

    -- Linked entities
    appointment_id          BIGINT REFERENCES appointments(id),
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    provider_id             BIGINT NOT NULL REFERENCES users(id),
    encounter_id            BIGINT REFERENCES encounters(id),

    -- Session details
    session_type            VARCHAR(50) NOT NULL, -- 'video', 'audio_only', 'chat'
    session_url             VARCHAR(500), -- Meeting room URL for patient
    provider_url            VARCHAR(500), -- Different URL for provider (with controls)
    session_id              VARCHAR(200), -- External platform session ID
    session_password        VARCHAR(100), -- Meeting password (encrypted)

    -- Platform integration
    platform                VARCHAR(50) NOT NULL, -- 'zoom', 'google_meet', 'whereby', 'jitsi', 'custom'
    platform_meeting_id     VARCHAR(200),
    platform_config         JSONB,

    -- Scheduling
    scheduled_start         TIMESTAMP NOT NULL,
    scheduled_end           TIMESTAMP NOT NULL,
    scheduled_duration_minutes INTEGER DEFAULT 30,

    -- Actual timing
    actual_start            TIMESTAMP,
    actual_end              TIMESTAMP,
    duration_minutes        INTEGER,

    -- Participants
    patient_joined_at       TIMESTAMP,
    patient_left_at         TIMESTAMP,
    provider_joined_at      TIMESTAMP,
    provider_left_at        TIMESTAMP,

    -- Features enabled
    requires_waiting_room   BOOLEAN DEFAULT true,
    recording_enabled       BOOLEAN DEFAULT false,
    recording_consent       BOOLEAN DEFAULT false, -- Patient must consent
    recording_url           VARCHAR(500),
    recording_duration      INTEGER,
    screen_sharing_enabled  BOOLEAN DEFAULT true,
    chat_enabled            BOOLEAN DEFAULT true,

    -- Status
    status                  VARCHAR(50) DEFAULT 'scheduled',
    -- 'scheduled', 'ready', 'waiting_room', 'in_progress', 'completed', 'cancelled',
    -- 'no_show_patient', 'no_show_provider', 'technical_issue', 'rescheduled'

    -- Connection quality tracking
    patient_connection_quality VARCHAR(50), -- 'excellent', 'good', 'fair', 'poor', 'failed'
    provider_connection_quality VARCHAR(50),
    connection_issues       TEXT[], -- Array of issues encountered

    -- Technical details
    patient_device_type     VARCHAR(50), -- 'desktop', 'mobile', 'tablet'
    patient_browser         VARCHAR(100),
    patient_ip_address      VARCHAR(45),
    provider_device_type    VARCHAR(50),
    provider_browser        VARCHAR(100),

    -- Insurance and billing
    is_billable             BOOLEAN DEFAULT true,
    insurance_id            BIGINT REFERENCES insurances(id),
    obra_social_authorized  BOOLEAN DEFAULT false,
    authorization_number    VARCHAR(100),
    authorization_date      DATE,
    copay_amount            DECIMAL(10,2),

    -- Clinical documentation
    chief_complaint         TEXT,
    clinical_notes          TEXT,
    prescriptions_issued    BIGINT[], -- Array of prescription IDs
    lab_orders_created      BIGINT[], -- Array of lab order IDs
    referrals_created       BIGINT[], -- Array of referral IDs
    follow_up_required      BOOLEAN DEFAULT false,
    follow_up_in_days       INTEGER,

    -- Technical logs
    connection_log          JSONB,
    /* Example:
    {
      "patient_bandwidth": "5.2 Mbps",
      "provider_bandwidth": "10.1 Mbps",
      "packet_loss": "0.1%",
      "latency_ms": 45,
      "quality_score": 4.5
    }
    */

    -- Feedback
    patient_rating          INTEGER, -- 1-5 stars
    patient_feedback        TEXT,
    technical_issues_reported TEXT,

    -- Compliance
    consent_signed          BOOLEAN DEFAULT false,
    consent_signed_at       TIMESTAMP,
    consent_ip_address      VARCHAR(45),

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_telemedicine_patient (patient_id),
    INDEX idx_telemedicine_provider (provider_id),
    INDEX idx_telemedicine_appointment (appointment_id),
    INDEX idx_telemedicine_scheduled (scheduled_start),
    INDEX idx_telemedicine_status (status),
    INDEX idx_telemedicine_platform (platform)
);

-- Telemedicine consent (required in Argentina - Ley de Telemedicina)
CREATE TABLE telemedicine_consents (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),

    -- Consent type
    consent_type            VARCHAR(50) DEFAULT 'general', -- 'general', 'recording', 'minor_consent'

    -- Consent text
    consent_text            TEXT NOT NULL,
    consent_version         VARCHAR(20) NOT NULL,

    -- Digital signature
    consented_at            TIMESTAMP NOT NULL,
    consent_ip_address      VARCHAR(45),
    consent_device          VARCHAR(100),
    signature_data          TEXT, -- Base64 encoded signature

    -- For minors
    guardian_name           VARCHAR(200),
    guardian_relationship   VARCHAR(100),
    guardian_dni            VARCHAR(50),
    guardian_signature      TEXT,

    -- Validity
    valid_from              DATE DEFAULT CURRENT_DATE,
    valid_until             DATE, -- NULL = perpetual
    is_active               BOOLEAN DEFAULT true,
    revoked_at              TIMESTAMP,
    revoked_reason          TEXT,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_telemedicine_consent_patient (patient_id),
    INDEX idx_telemedicine_consent_active (is_active)
);

-- Session participants (for group sessions, consultations with specialists)
CREATE TABLE telemedicine_participants (
    id                      BIGSERIAL PRIMARY KEY,
    session_id              BIGINT NOT NULL REFERENCES telemedicine_sessions(id) ON DELETE CASCADE,

    -- Participant details
    participant_type        VARCHAR(50) NOT NULL, -- 'patient', 'provider', 'specialist', 'family_member', 'interpreter'
    user_id                 BIGINT REFERENCES users(id), -- If provider/staff
    patient_id              BIGINT REFERENCES patients(id), -- If patient
    name                    VARCHAR(200), -- For external participants
    email                   VARCHAR(255),

    -- Participation
    invited_at              TIMESTAMP,
    joined_at               TIMESTAMP,
    left_at                 TIMESTAMP,
    duration_minutes        INTEGER,

    -- Permissions
    can_share_screen        BOOLEAN DEFAULT false,
    can_record              BOOLEAN DEFAULT false,
    can_chat                BOOLEAN DEFAULT true,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_telemedicine_participants_session (session_id)
);

-- Chat messages during session
CREATE TABLE telemedicine_chat_messages (
    id                      BIGSERIAL PRIMARY KEY,
    session_id              BIGINT NOT NULL REFERENCES telemedicine_sessions(id) ON DELETE CASCADE,

    -- Message details
    sender_type             VARCHAR(50) NOT NULL, -- 'patient', 'provider'
    sender_id               BIGINT, -- User or patient ID
    sender_name             VARCHAR(200),

    message_content         TEXT NOT NULL,
    message_type            VARCHAR(50) DEFAULT 'text', -- 'text', 'file', 'image'

    -- Attachments
    attachment_url          VARCHAR(500),
    attachment_type         VARCHAR(100),
    attachment_size         INTEGER,

    sent_at                 TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_telemedicine_chat_session (session_id),
    INDEX idx_telemedicine_chat_sent (sent_at)
);

-- Technical support tickets
CREATE TABLE telemedicine_support_tickets (
    id                      BIGSERIAL PRIMARY KEY,
    session_id              BIGINT REFERENCES telemedicine_sessions(id),

    -- Issue details
    reported_by_type        VARCHAR(50), -- 'patient', 'provider'
    reported_by_id          BIGINT,
    issue_category          VARCHAR(100), -- 'connection', 'audio', 'video', 'screen_share', 'other'
    issue_description       TEXT NOT NULL,

    -- Device/browser info
    device_info             JSONB,

    -- Resolution
    status                  VARCHAR(50) DEFAULT 'open', -- 'open', 'investigating', 'resolved', 'closed'
    resolution              TEXT,
    resolved_at             TIMESTAMP,
    resolved_by             BIGINT REFERENCES users(id),

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_telemedicine_support_session (session_id),
    INDEX idx_telemedicine_support_status (status)
);

-- Platform credentials and configuration
CREATE TABLE telemedicine_platform_config (
    id                      BIGSERIAL PRIMARY KEY,
    platform_name           VARCHAR(50) UNIQUE NOT NULL,

    -- API credentials (encrypted)
    api_key                 TEXT,
    api_secret              TEXT,
    oauth_credentials       JSONB,

    -- Configuration
    config                  JSONB NOT NULL,
    /* Example for Zoom:
    {
      "account_id": "xxx",
      "webhook_secret": "xxx",
      "default_settings": {
        "waiting_room": true,
        "join_before_host": false,
        "mute_upon_entry": true,
        "auto_recording": "none"
      }
    }
    */

    -- Features available
    supports_recording      BOOLEAN DEFAULT true,
    supports_screen_sharing BOOLEAN DEFAULT true,
    supports_waiting_room   BOOLEAN DEFAULT true,
    supports_chat           BOOLEAN DEFAULT true,
    max_participants        INTEGER DEFAULT 100,

    -- Usage limits
    monthly_limit           INTEGER, -- Minutes per month
    current_usage           INTEGER DEFAULT 0,

    -- Status
    is_active               BOOLEAN DEFAULT true,
    is_primary              BOOLEAN DEFAULT false,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_telemedicine_platform_active (is_active)
);
```

### 1.2 Video Platform Integrations

#### 1.2.1 Zoom Integration

```javascript
/**
 * Create Zoom meeting for telemedicine session
 */
async function createZoomMeeting(session) {
  const zoomConfig = await TelemedicinePlatformConfig.findOne({
    where: { platform_name: 'zoom', is_active: true }
  });

  const zoomClient = new ZoomClient({
    apiKey: decrypt(zoomConfig.api_key),
    apiSecret: decrypt(zoomConfig.api_secret)
  });

  const meetingConfig = {
    topic: `Consulta - ${session.Patient.first_name} ${session.Patient.last_name}`,
    type: 2, // Scheduled meeting
    start_time: session.scheduled_start,
    duration: session.scheduled_duration_minutes,
    timezone: 'America/Argentina/Buenos_Aires',

    settings: {
      host_video: true,
      participant_video: true,
      join_before_host: false,
      mute_upon_entry: true,
      waiting_room: session.requires_waiting_room,
      auto_recording: session.recording_enabled ? 'cloud' : 'none',
      audio: 'both',

      // Security
      meeting_authentication: false, // Allow guests
      require_password_for_scheduling_new_meetings: true,

      // Medical consultation specific
      watermark: true, // Prevent screenshots
      allow_multiple_devices: true
    }
  };

  const meeting = await zoomClient.createMeeting(meetingConfig);

  // Update session with meeting details
  await session.update({
    platform: 'zoom',
    platform_meeting_id: meeting.id,
    session_url: meeting.join_url,
    provider_url: meeting.start_url,
    session_password: meeting.password,
    platform_config: {
      meeting_id: meeting.id,
      uuid: meeting.uuid,
      h323_password: meeting.h323_password
    }
  });

  return meeting;
}

/**
 * Handle Zoom webhooks
 */
async function handleZoomWebhook(payload) {
  const eventType = payload.event;
  const meetingId = payload.payload.object.id;

  const session = await TelemedicineSession.findOne({
    where: { platform_meeting_id: meetingId }
  });

  if (!session) return;

  switch (eventType) {
    case 'meeting.started':
      await session.update({
        status: 'in_progress',
        actual_start: new Date()
      });
      break;

    case 'meeting.ended':
      await session.update({
        status: 'completed',
        actual_end: new Date(),
        duration_minutes: Math.round(
          (new Date() - session.actual_start) / 60000
        )
      });

      // Trigger post-session workflow
      await processPostSession(session);
      break;

    case 'meeting.participant_joined':
      const participant = payload.payload.object.participant;
      if (participant.email === session.Patient.email) {
        await session.update({ patient_joined_at: new Date() });
      } else {
        await session.update({ provider_joined_at: new Date() });
      }
      break;

    case 'recording.completed':
      const recording = payload.payload.object.recording_files[0];
      await session.update({
        recording_url: recording.download_url,
        recording_duration: recording.recording_duration
      });
      break;
  }
}
```

#### 1.2.2 Google Meet Integration

```javascript
/**
 * Create Google Meet for telemedicine
 */
async function createGoogleMeet(session) {
  const googleConfig = await TelemedicinePlatformConfig.findOne({
    where: { platform_name: 'google_meet', is_active: true }
  });

  const oauth2Client = new google.auth.OAuth2();
  oauth2Client.setCredentials(JSON.parse(googleConfig.oauth_credentials));

  const calendar = google.calendar({ version: 'v3', auth: oauth2Client });

  const event = {
    summary: `Teleconsulta - ${session.Patient.first_name} ${session.Patient.last_name}`,
    description: `Consulta virtual con ${session.Provider.first_name} ${session.Provider.last_name}`,
    start: {
      dateTime: session.scheduled_start,
      timeZone: 'America/Argentina/Buenos_Aires'
    },
    end: {
      dateTime: session.scheduled_end,
      timeZone: 'America/Argentina/Buenos_Aires'
    },
    conferenceData: {
      createRequest: {
        requestId: `telemedicine-${session.id}`,
        conferenceSolutionKey: { type: 'hangoutsMeet' }
      }
    },
    attendees: [
      { email: session.Patient.email },
      { email: session.Provider.email }
    ],
    reminders: {
      useDefault: false,
      overrides: [
        { method: 'email', minutes: 1440 }, // 1 day before
        { method: 'popup', minutes: 30 }
      ]
    }
  };

  const calendarEvent = await calendar.events.insert({
    calendarId: 'primary',
    conferenceDataVersion: 1,
    resource: event
  });

  const meetLink = calendarEvent.data.hangoutLink;

  await session.update({
    platform: 'google_meet',
    platform_meeting_id: calendarEvent.data.id,
    session_url: meetLink,
    provider_url: meetLink,
    platform_config: {
      calendar_event_id: calendarEvent.data.id,
      conference_id: calendarEvent.data.conferenceData.conferenceId
    }
  });

  return calendarEvent;
}
```

#### 1.2.3 Whereby Integration (Simple, No Account Required)

```javascript
/**
 * Create Whereby room for telemedicine
 */
async function createWherebyRoom(session) {
  const wherebyConfig = await TelemedicinePlatformConfig.findOne({
    where: { platform_name: 'whereby', is_active: true }
  });

  const response = await fetch('https://api.whereby.dev/v1/meetings', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${decrypt(wherebyConfig.api_key)}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      endDate: session.scheduled_end,
      fields: ['hostRoomUrl'],
      roomNamePrefix: `consulta-${session.id}`,
      roomMode: 'normal',
      isLocked: true,
      hostRoomUrl: true
    })
  });

  const meeting = await response.json();

  await session.update({
    platform: 'whereby',
    platform_meeting_id: meeting.meetingId,
    session_url: meeting.roomUrl,
    provider_url: meeting.hostRoomUrl, // Host has extra controls
    platform_config: meeting
  });

  return meeting;
}
```

---

## 2. TELEMEDICINE WORKFLOWS

### 2.1 Session Booking Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│ TELEMEDICINE SESSION BOOKING                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ 1. APPOINTMENT CREATION                                         │
│    ├─ Patient/Staff creates appointment                         │
│    ├─ Select "Teleconsulta" as appointment type                │
│    ├─ System checks telemedicine consent                       │
│    │  └─ If not signed → Prompt to sign consent                │
│    └─ Save appointment with telemedicine flag                  │
│                                                                 │
│ 2. OBRA SOCIAL AUTHORIZATION (if required)                      │
│    ├─ Check if obra social covers telemedicine                 │
│    ├─ Submit authorization request                             │
│    │  └─ Include: diagnosis, provider specialty, duration      │
│    └─ Wait for approval                                        │
│                                                                 │
│ 3. CREATE VIDEO SESSION                                         │
│    ├─ 24 hours before: Create meeting room                     │
│    ├─ Select platform (Zoom/Meet/Whereby based on config)      │
│    ├─ Generate unique meeting URL                              │
│    └─ Store session details in database                        │
│                                                                 │
│ 4. SEND NOTIFICATIONS                                           │
│    ├─ Email to patient with:                                   │
│    │  └─ Meeting link, instructions, technical requirements    │
│    ├─ Email to provider with:                                  │
│    │  └─ Host link, patient info, clinical notes              │
│    └─ SMS reminder 2 hours before                              │
│                                                                 │
│ 5. PRE-SESSION CHECK                                            │
│    ├─ Patient accesses link 15 min before                      │
│    ├─ Run tech check (camera, mic, internet)                   │
│    ├─ Enter waiting room                                       │
│    └─ System notifies provider patient is ready                │
│                                                                 │
│ 6. DURING SESSION                                               │
│    ├─ Provider admits patient from waiting room                │
│    ├─ Conduct consultation                                     │
│    ├─ Access patient EHR during call                           │
│    ├─ Share screen to show lab results/images                  │
│    ├─ Document chief complaint and notes                       │
│    └─ Issue prescriptions/orders if needed                     │
│                                                                 │
│ 7. POST-SESSION                                                 │
│    ├─ Provider completes clinical notes                        │
│    ├─ Create encounter record                                  │
│    ├─ Generate invoice                                         │
│    ├─ Send prescriptions to patient                            │
│    ├─ Request patient feedback                                 │
│    └─ Archive recording (if enabled with consent)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Technical Requirements Check

```javascript
/**
 * Check patient device capabilities before session
 */
async function checkTechnicalRequirements() {
  const requirements = {
    browser: {
      chrome: '>=90',
      firefox: '>=88',
      safari: '>=14',
      edge: '>=90'
    },
    bandwidth: {
      download: '>=2 Mbps',
      upload: '>=1 Mbps'
    },
    permissions: {
      camera: 'required',
      microphone: 'required',
      notifications: 'optional'
    }
  };

  // Check browser
  const browserCheck = detectBrowser();

  // Check bandwidth
  const speedTest = await measureBandwidth();

  // Check camera/mic access
  const mediaCheck = await testMediaDevices();

  return {
    compatible: browserCheck.compatible && speedTest.sufficient && mediaCheck.granted,
    details: {
      browser: browserCheck,
      speed: speedTest,
      media: mediaCheck
    },
    recommendations: generateRecommendations({...browserCheck, ...speedTest, ...mediaCheck})
  };
}
```

---

## 3. CLINICAL DOCUMENTATION DURING TELEMEDICINE

### 3.1 Integrated EHR Access

```
┌──────────────────────────────────────────────────────────────┐
│ TELEMEDICINE SESSION - PROVIDER VIEW                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ ┌──────────────────┐  ┌────────────────────────────────┐   │
│ │                  │  │ Patient: Juan Pérez            │   │
│ │                  │  │ DNI: 30-12345678-9             │   │
│ │                  │  │ Obra Social: OSDE              │   │
│ │   VIDEO FEED     │  ├────────────────────────────────┤   │
│ │                  │  │ Chief Complaint:               │   │
│ │                  │  │ [________________]             │   │
│ │                  │  │                                │   │
│ │                  │  │ Vital Signs:                   │   │
│ └──────────────────┘  │ • BP: [___] / [___]            │   │
│                       │ • HR: [___] bpm                │   │
│ [End Call] [Mute]     │ • Temp: [___] °C               │   │
│ [Video] [Share]       │                                │   │
│                       │ Quick Access:                  │   │
│                       │ [Medical History]              │   │
│                       │ [Recent Labs]                  │   │
│                       │ [Current Medications]          │   │
│                       │ [Allergies]                    │   │
│                       │                                │   │
│                       │ Actions:                       │   │
│                       │ [+ Prescription]               │   │
│                       │ [+ Lab Order]                  │   │
│                       │ [+ Referral]                   │   │
│                       │                                │   │
│                       │ [Complete & Bill]              │   │
│                       └────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. BILLING FOR TELEMEDICINE

### 4.1 Telemedicine Service Pricing

```sql
-- Add telemedicine-specific pricing
ALTER TABLE service_pricing ADD COLUMN is_telemedicine BOOLEAN DEFAULT false;
ALTER TABLE service_pricing ADD COLUMN modality VARCHAR(50); -- 'in_person', 'telemedicine', 'both'

-- Telemedicine service codes
INSERT INTO services (code, name, category, description, default_price, is_telemedicine)
VALUES
('TELE-CONS-001', 'Teleconsulta General', 'telemedicine', 'Consulta general por videollamada', 1500.00, true),
('TELE-CONS-ESP', 'Teleconsulta Especialista', 'telemedicine', 'Consulta con especialista por video', 2500.00, true),
('TELE-PSYCH', 'Telepsicología', 'telemedicine', 'Consulta psicológica virtual', 2000.00, true),
('TELE-NUTR', 'Telenutrición', 'telemedicine', 'Consulta nutricional virtual', 1800.00, true);
```

### 4.2 Obra Social Coverage for Telemedicine

```sql
-- Obra Social telemedicine coverage
CREATE TABLE obra_social_telemedicine_coverage (
    id                      BIGSERIAL PRIMARY KEY,
    insurance_id            BIGINT NOT NULL REFERENCES insurances(id),

    -- Coverage details
    covers_telemedicine     BOOLEAN DEFAULT false,
    coverage_started_date   DATE, -- When they started covering telemedicine

    -- Services covered
    covered_service_types   VARCHAR(100)[], -- ['general_consultation', 'specialist', 'psychology', 'nutrition']

    -- Authorization required
    requires_authorization  BOOLEAN DEFAULT false,
    max_sessions_per_month  INTEGER, -- NULL = unlimited
    max_sessions_per_year   INTEGER,

    -- Copay
    copay_amount            DECIMAL(10,2),
    copay_percentage        DECIMAL(5,2),

    -- Restrictions
    time_restrictions       JSONB,
    /* Example:
    {
      "allowed_hours": "08:00-20:00",
      "allowed_days": [1,2,3,4,5], // Mon-Fri only
      "blackout_dates": ["2025-01-01", "2025-12-25"]
    }
    */

    -- Documentation required
    requires_clinical_justification BOOLEAN DEFAULT false,
    required_documents      TEXT[],

    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_obra_social_tele_coverage (insurance_id)
);

-- Example: OSDE covers telemedicine
INSERT INTO obra_social_telemedicine_coverage (
    insurance_id, covers_telemedicine, coverage_started_date,
    covered_service_types, requires_authorization, copay_amount
) VALUES (
    1, -- OSDE ID
    true,
    '2020-04-01', -- Started during COVID
    ARRAY['general_consultation', 'specialist', 'psychology'],
    false,
    300.00 -- $300 copay
);
```

---

## 5. LEGAL & COMPLIANCE (ARGENTINA)

### 5.1 Telemedicine Consent Form (Spanish)

```javascript
const TELEMEDICINE_CONSENT_ES = `
CONSENTIMIENTO INFORMADO PARA TELEMEDICINA

Yo, {{patient_name}}, DNI {{patient_dni}}, por medio del presente:

DECLARO QUE:

1. He sido informado/a sobre la naturaleza de la consulta por telemedicina y comprendo que:
   - La consulta se realizará a través de videoconferencia
   - Requiere conexión a internet estable
   - La calidad de la consulta depende de la conexión
   - No reemplaza la consulta presencial en casos de emergencia

2. CONFIDENCIALIDAD:
   - Comprendo que la plataforma utiliza cifrado de extremo a extremo
   - Mi información médica será tratada con total confidencialidad
   - Las grabaciones (si las hay) serán almacenadas de forma segura

3. LIMITACIONES:
   - Entiendo que ciertas evaluaciones físicas no pueden realizarse virtualmente
   - En caso de requerir examen físico, deberé agendar consulta presencial
   - En caso de emergencia, debo acudir al servicio de urgencias

4. REQUISITOS TÉCNICOS:
   - Debo contar con dispositivo con cámara y micrófono
   - Conexión a internet estable (mínimo 2 Mbps)
   - Navegador actualizado

5. COSTOS:
   - Entiendo que mi obra social/prepaga {{insurance_coverage}}
   - El copago/coseguro es de ${{copay_amount}}

AUTORIZO:
- La realización de la consulta por telemedicina
- El acceso del profesional a mi historia clínica electrónica
- La documentación de la consulta en mi expediente médico
- La grabación de la sesión solo con mi consentimiento explícito

DERECHO A REVOCAR:
Puedo revocar este consentimiento en cualquier momento sin que esto afecte mi atención médica.

Fecha: {{consent_date}}
Firma Digital: _________________
IP: {{ip_address}}
`;
```

### 5.2 Recording Consent (Separate)

```javascript
const RECORDING_CONSENT_ES = `
CONSENTIMIENTO PARA GRABACIÓN DE CONSULTA POR TELEMEDICINA

Autorizo la grabación de mi consulta de telemedicina del día {{session_date}}
con el/la profesional {{provider_name}} para los siguientes fines:

☐ Documentación médica y continuidad de atención
☐ Educación médica (con información personal anonimizada)
☐ Control de calidad

COMPRENDO QUE:
- La grabación será almacenada de forma segura y cifrada
- Solo personal autorizado tendrá acceso
- La grabación se conservará por {{retention_period}}
- Puedo solicitar una copia de la grabación
- Puedo revocar este consentimiento en cualquier momento

Firma: _________________
Fecha: {{consent_date}}
`;
```

---

## 6. API ENDPOINTS

### 6.1 Telemedicine Session Management

```javascript
/**
 * @api {post} /api/telemedicine/sessions Create Telemedicine Session
 * @apiName CreateTelemedicineSession
 * @apiGroup Telemedicine
 */
POST /api/telemedicine/sessions
Authorization: Bearer {token}
Content-Type: application/json

Request:
{
  "appointment_id": 123,
  "patient_id": 456,
  "provider_id": 789,
  "scheduled_start": "2025-11-20T10:00:00-03:00",
  "scheduled_duration_minutes": 30,
  "session_type": "video",
  "platform": "zoom", // or "google_meet", "whereby"
  "requires_waiting_room": true,
  "recording_enabled": false
}

Response 201:
{
  "id": 1,
  "session_id": "sess_abc123",
  "session_url": "https://zoom.us/j/123456789?pwd=xxx",
  "provider_url": "https://zoom.us/s/123456789?zak=xxx",
  "session_password": "abc123",
  "scheduled_start": "2025-11-20T10:00:00-03:00",
  "scheduled_end": "2025-11-20T10:30:00-03:00",
  "status": "scheduled",
  "technical_requirements": {
    "minimum_bandwidth": "2 Mbps",
    "supported_browsers": ["Chrome 90+", "Firefox 88+", "Safari 14+"],
    "required_permissions": ["camera", "microphone"]
  }
}

/**
 * @api {get} /api/telemedicine/sessions/:id Get Session Details
 */
GET /api/telemedicine/sessions/1
Authorization: Bearer {token}

Response 200:
{
  "id": 1,
  "patient": {
    "id": 456,
    "name": "Juan Pérez",
    "email": "juan@example.com"
  },
  "provider": {
    "id": 789,
    "name": "Dr. María García",
    "specialty": "Medicina General"
  },
  "session_url": "https://zoom.us/j/123456789?pwd=xxx",
  "scheduled_start": "2025-11-20T10:00:00-03:00",
  "status": "scheduled",
  "consent_signed": true,
  "insurance_authorized": true
}

/**
 * @api {patch} /api/telemedicine/sessions/:id/status Update Session Status
 */
PATCH /api/telemedicine/sessions/1/status
Authorization: Bearer {token}
Content-Type: application/json

{
  "status": "in_progress",
  "actual_start": "2025-11-20T10:02:00-03:00"
}

Response 200:
{
  "id": 1,
  "status": "in_progress",
  "actual_start": "2025-11-20T10:02:00-03:00"
}

/**
 * @api {post} /api/telemedicine/sessions/:id/complete Complete Session
 */
POST /api/telemedicine/sessions/1/complete
Authorization: Bearer {token}
Content-Type: application/json

{
  "clinical_notes": "Patient reports improvement in symptoms...",
  "prescriptions": [123, 456], // Prescription IDs created during session
  "lab_orders": [789],
  "follow_up_required": true,
  "follow_up_in_days": 7,
  "patient_connection_quality": "good",
  "provider_connection_quality": "excellent"
}

Response 200:
{
  "id": 1,
  "status": "completed",
  "encounter_id": 999,
  "invoice_id": 888,
  "duration_minutes": 28
}

/**
 * @api {post} /api/telemedicine/consent Sign Telemedicine Consent
 */
POST /api/telemedicine/consent
Authorization: Bearer {token}
Content-Type: application/json

{
  "patient_id": 456,
  "consent_type": "general",
  "signature_data": "data:image/png;base64,iVBORw0KG...",
  "ip_address": "190.123.45.67",
  "consent_device": "Chrome 96 on Windows 10"
}

Response 201:
{
  "id": 1,
  "patient_id": 456,
  "consent_type": "general",
  "consented_at": "2025-11-16T14:30:00-03:00",
  "valid_until": null,
  "is_active": true
}

/**
 * @api {get} /api/telemedicine/check-requirements Check Technical Requirements
 */
GET /api/telemedicine/check-requirements
Authorization: Bearer {token}

Response 200:
{
  "compatible": true,
  "details": {
    "browser": {
      "name": "Chrome",
      "version": "96.0",
      "compatible": true
    },
    "bandwidth": {
      "download": "5.2 Mbps",
      "upload": "2.1 Mbps",
      "sufficient": true
    },
    "permissions": {
      "camera": "granted",
      "microphone": "granted",
      "notifications": "denied"
    }
  },
  "recommendations": []
}
```

---

## 7. PATIENT PORTAL INTEGRATION

### 7.1 Patient Telemedicine UI

```
┌──────────────────────────────────────────────────────────────┐
│ Mis Teleconsultas                                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ Próxima Consulta:                                            │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ 🎥 Teleconsulta - Dr. María García                     │  │
│ │ Medicina General                                       │  │
│ │                                                        │  │
│ │ 📅 Miércoles 20 de Noviembre, 2025                     │  │
│ │ 🕐 10:00 - 10:30 (30 minutos)                          │  │
│ │                                                        │  │
│ │ Estado: ✅ Confirmada                                  │  │
│ │ Obra Social: OSDE (Autorizada)                         │  │
│ │ Copago: $300                                           │  │
│ │                                                        │  │
│ │ ⚠️ Requisitos Técnicos:                                │  │
│ │ • Conexión a internet estable (mín. 2 Mbps)           │  │
│ │ • Dispositivo con cámara y micrófono                  │  │
│ │ • Navegador actualizado (Chrome, Firefox, Safari)     │  │
│ │                                                        │  │
│ │ [Verificar Conexión] [Ingresar a la Consulta]         │  │
│ │ [Cancelar] [Reprogramar]                              │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
│ Instrucciones:                                               │
│ 1. Ingrese 15 minutos antes para verificar audio/video      │
│ 2. Tenga a mano su DNI y carnet de obra social             │
│ 3. Prepareuna lista de sus síntomas/medicamentos           │
│ 4. Busque un lugar tranquilo y con buena iluminación       │
│                                                              │
│ Historial de Teleconsultas:                                  │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ 15/10/2025 - Dr. García - Control post-tratamiento    │  │
│ │ [Ver Resumen] [Ver Recetas]                           │  │
│ ├────────────────────────────────────────────────────────┤  │
│ │ 20/09/2025 - Dr. García - Consulta general            │  │
│ │ [Ver Resumen] [Ver Recetas]                           │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

*Telemedicine Specification Version: 1.0*
*Last Updated: 2025-11-16*
*Complete telemedicine and virtual consultation system for Argentina*
