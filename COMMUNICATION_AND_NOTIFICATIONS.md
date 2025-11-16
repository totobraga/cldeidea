# Clinical Management System - Communication & Notification Configuration

## Overview

This document provides comprehensive specifications for patient communication, appointment reminders, notifications, and all related configuration options.

---

## 1. APPOINTMENT REMINDER SYSTEM

### 1.1 System-Wide Reminder Configuration

#### 1.1.1 Admin Reminder Settings

```sql
CREATE TABLE reminder_configurations (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Applicability
    applies_to              VARCHAR(50) NOT NULL, -- 'all', 'appointment_type', 'insurance', 'provider', 'location'
    appointment_type_id     BIGINT REFERENCES appointment_types(id),
    insurance_id            BIGINT REFERENCES insurances(id),
    provider_id             BIGINT REFERENCES users(id),
    location_id             BIGINT REFERENCES locations(id),

    -- Reminder schedule
    reminders               JSONB NOT NULL,
    /* Example:
    [
      {
        "timing_value": 24,
        "timing_unit": "hours",
        "channels": ["sms", "email"],
        "template_id": 1,
        "priority": 1
      },
      {
        "timing_value": 2,
        "timing_unit": "hours",
        "channels": ["sms", "whatsapp"],
        "template_id": 2,
        "priority": 2
      },
      {
        "timing_value": 1,
        "timing_unit": "weeks",
        "channels": ["email"],
        "template_id": 3,
        "priority": 0
      }
    ]
    */

    -- Confirmation workflow
    enable_confirmation     BOOLEAN DEFAULT false,
    confirmation_methods    VARCHAR(50)[], -- ['sms_reply', 'whatsapp', 'portal', 'phone']
    confirmation_deadline   INTEGER, -- Hours before appointment

    -- Auto-cancel if not confirmed
    auto_cancel_unconfirmed BOOLEAN DEFAULT false,
    cancel_threshold        INTEGER, -- Hours before appointment

    -- Status
    is_active               BOOLEAN DEFAULT true,
    priority                INTEGER DEFAULT 0, -- Higher priority overrides

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_reminder_config_type (appointment_type_id),
    INDEX idx_reminder_config_insurance (insurance_id),
    INDEX idx_reminder_config_provider (provider_id)
);
```

#### 1.1.2 Default Reminder Configurations

```sql
-- Default configuration: All appointments
INSERT INTO reminder_configurations (name, applies_to, reminders, enable_confirmation) VALUES
('Default Appointment Reminders', 'all',
'[
  {
    "timing_value": 24,
    "timing_unit": "hours",
    "channels": ["sms", "email"],
    "template_id": 1,
    "priority": 1
  },
  {
    "timing_value": 2,
    "timing_unit": "hours",
    "channels": ["sms"],
    "template_id": 2,
    "priority": 2
  }
]'::jsonb, true);

-- PAMI patients: Earlier reminders
INSERT INTO reminder_configurations (name, applies_to, insurance_id, reminders) VALUES
('PAMI Reminders', 'insurance', 5, -- PAMI insurance ID
'[
  {
    "timing_value": 3,
    "timing_unit": "days",
    "channels": ["phone"], -- Phone call preferred for elderly
    "template_id": 10,
    "priority": 0
  },
  {
    "timing_value": 24,
    "timing_unit": "hours",
    "channels": ["sms"],
    "template_id": 11,
    "priority": 1
  }
]'::jsonb);

-- First consultation: More reminders
INSERT INTO reminder_configurations (name, applies_to, appointment_type_id, reminders, enable_confirmation) VALUES
('New Patient Reminders', 'appointment_type', 1,
'[
  {
    "timing_value": 1,
    "timing_unit": "weeks",
    "channels": ["email", "sms"],
    "template_id": 20,
    "priority": 0
  },
  {
    "timing_value": 3,
    "timing_unit": "days",
    "channels": ["email"],
    "template_id": 21,
    "priority": 1
  },
  {
    "timing_value": 24,
    "timing_unit": "hours",
    "channels": ["sms", "whatsapp"],
    "template_id": 22,
    "priority": 2
  }
]'::jsonb, true);
```

### 1.2 Communication Templates

#### 1.2.1 Template Management

```sql
CREATE TABLE communication_templates (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Template type
    template_type           VARCHAR(50) NOT NULL, -- 'appointment_reminder', 'confirmation', 'cancellation', 'lab_result', 'payment_reminder'

    -- Channel-specific content
    sms_content             TEXT,
    email_subject           VARCHAR(500),
    email_html              TEXT,
    email_text              TEXT,
    whatsapp_content        TEXT,

    -- WhatsApp template (if using approved templates)
    whatsapp_template_id    VARCHAR(200), -- From WhatsApp Business API
    whatsapp_template_vars  JSONB, -- Variable mapping

    -- Template variables available
    available_variables     TEXT[], -- ['patient_name', 'appointment_date', 'appointment_time', etc.]

    -- Language
    language                VARCHAR(10) DEFAULT 'es', -- 'es', 'en'

    -- Categories
    category                VARCHAR(50), -- For organization
    tags                    VARCHAR(100)[],

    -- Status
    is_active               BOOLEAN DEFAULT true,
    is_system               BOOLEAN DEFAULT false, -- System templates can't be deleted

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_templates_type (template_type),
    INDEX idx_templates_language (language)
);
```

#### 1.2.2 Default Templates (Spanish - Argentina)

```sql
-- Template 1: 24-hour reminder
INSERT INTO communication_templates (
    name, template_type, sms_content, email_subject, email_html, email_text, whatsapp_content,
    available_variables, language, is_system
) VALUES (
    'Recordatorio 24hs',
    'appointment_reminder',

    -- SMS (160 chars limit)
    'Hola {{patient_first_name}}, recordatorio: turno mañana {{appointment_date}} a las {{appointment_time}} con {{doctor_name}}. Clínica {{clinic_name}}. Confirmar: responder C',

    -- Email subject
    'Recordatorio: Turno mañana {{appointment_date}}',

    -- Email HTML
    '<html>
    <body style="font-family: Arial, sans-serif; color: #333;">
        <div style="max-width: 600px; margin: 0 auto; padding: 20px; border: 1px solid #ddd; border-radius: 5px;">
            <h2 style="color: #2563eb;">Recordatorio de Turno</h2>
            <p>Hola <strong>{{patient_full_name}}</strong>,</p>
            <p>Le recordamos que tiene un turno agendado:</p>
            <div style="background: #f3f4f6; padding: 15px; border-radius: 5px; margin: 20px 0;">
                <p><strong>Fecha:</strong> {{appointment_date_long}}</p>
                <p><strong>Hora:</strong> {{appointment_time}}</p>
                <p><strong>Profesional:</strong> {{doctor_title}} {{doctor_name}}</p>
                <p><strong>Especialidad:</strong> {{doctor_specialty}}</p>
                <p><strong>Consultorio:</strong> {{location_name}}</p>
                <p><strong>Dirección:</strong> {{location_address}}</p>
            </div>
            <p><strong>Importante:</strong></p>
            <ul>
                <li>Por favor llegue 10 minutos antes</li>
                <li>Traiga su DNI y carnet de obra social</li>
                <li>Si necesita cancelar, avise con 24hs de anticipación</li>
            </ul>
            <p>Para confirmar su asistencia, puede:</p>
            <ul>
                <li>Responder este email con "CONFIRMO"</li>
                <li>Llamar al {{clinic_phone}}</li>
                <li>Ingresar al portal de pacientes</li>
            </ul>
            <p>Saludos,<br>
            <strong>{{clinic_name}}</strong><br>
            {{clinic_address}}<br>
            Tel: {{clinic_phone}}</p>
        </div>
    </body>
    </html>',

    -- Email text (plain)
    'Hola {{patient_full_name}},

Le recordamos que tiene un turno agendado:

Fecha: {{appointment_date_long}}
Hora: {{appointment_time}}
Profesional: {{doctor_title}} {{doctor_name}}
Especialidad: {{doctor_specialty}}
Consultorio: {{location_name}}
Dirección: {{location_address}}

IMPORTANTE:
- Llegue 10 minutos antes
- Traiga DNI y carnet de obra social
- Cancele con 24hs de anticipación si no puede asistir

Para confirmar, responda "CONFIRMO" o llame al {{clinic_phone}}

Saludos,
{{clinic_name}}
{{clinic_address}}
Tel: {{clinic_phone}}',

    -- WhatsApp
    'Hola {{patient_first_name}} 👋

📅 Recordatorio de turno:
🗓️ {{appointment_date_long}}
🕐 {{appointment_time}}
👨‍⚕️ {{doctor_title}} {{doctor_name}}
📍 {{location_name}}

*Importante:*
✓ Llegar 10 min antes
✓ Traer DNI y carnet obra social

Para confirmar, responde *C*
Para reprogramar, responde *R*
Para cancelar, responde *X*

{{clinic_name}}',

    ARRAY['patient_first_name', 'patient_full_name', 'patient_dni', 'appointment_date', 'appointment_date_long',
          'appointment_time', 'doctor_name', 'doctor_title', 'doctor_specialty', 'location_name', 'location_address',
          'clinic_name', 'clinic_phone', 'clinic_address', 'confirmation_link'],

    'es',
    true
);

-- Template 2: 2-hour reminder
INSERT INTO communication_templates (
    name, template_type, sms_content, whatsapp_content,
    available_variables, language, is_system
) VALUES (
    'Recordatorio 2hs',
    'appointment_reminder',

    -- SMS
    '⏰ Recordatorio: su turno es HOY a las {{appointment_time}} con {{doctor_name}} en {{location_name}}. {{clinic_name}}',

    -- WhatsApp
    '⏰ *Recordatorio*

Su turno es *HOY* a las {{appointment_time}}

👨‍⚕️ {{doctor_title}} {{doctor_name}}
📍 {{location_name}} - {{location_address}}

Nos vemos en un rato! 🏥',

    ARRAY['appointment_time', 'doctor_name', 'doctor_title', 'location_name', 'location_address', 'clinic_name'],

    'es',
    true
);

-- Template 3: Confirmation received
INSERT INTO communication_templates (
    name, template_type, sms_content, email_subject, email_text, whatsapp_content,
    available_variables, language, is_system
) VALUES (
    'Confirmación recibida',
    'confirmation',

    -- SMS
    '✓ Turno confirmado para {{appointment_date}} {{appointment_time}} con {{doctor_name}}. Gracias! {{clinic_name}}',

    -- Email subject
    'Turno confirmado - {{appointment_date}}',

    -- Email text
    'Hola {{patient_full_name}},

Su turno ha sido confirmado exitosamente.

Fecha: {{appointment_date_long}}
Hora: {{appointment_time}}
Profesional: {{doctor_title}} {{doctor_name}}
Ubicación: {{location_name}}

Gracias por confirmar.

Saludos,
{{clinic_name}}',

    -- WhatsApp
    '✅ *Turno confirmado*

{{appointment_date_long}}
{{appointment_time}}
{{doctor_title}} {{doctor_name}}

Gracias por confirmar! Nos vemos pronto 😊',

    ARRAY['patient_first_name', 'patient_full_name', 'appointment_date', 'appointment_date_long',
          'appointment_time', 'doctor_name', 'doctor_title', 'location_name', 'clinic_name'],

    'es',
    true
);

-- Template 4: Lab results ready
INSERT INTO communication_templates (
    name, template_type, sms_content, email_subject, email_html, whatsapp_content,
    available_variables, language, is_system
) VALUES (
    'Resultados de laboratorio disponibles',
    'lab_result',

    -- SMS
    'Hola {{patient_first_name}}, sus resultados de laboratorio están listos. Puede verlos en el portal o retirarlos en {{clinic_name}}. Tel: {{clinic_phone}}',

    -- Email subject
    'Resultados de laboratorio disponibles',

    -- Email HTML
    '<html>
    <body style="font-family: Arial, sans-serif;">
        <div style="max-width: 600px; margin: 0 auto; padding: 20px;">
            <h2 style="color: #2563eb;">Resultados Disponibles</h2>
            <p>Hola <strong>{{patient_full_name}}</strong>,</p>
            <p>Le informamos que sus resultados de laboratorio del {{order_date}} ya están disponibles.</p>
            <p><strong>Estudio:</strong> {{test_name}}</p>
            <p>Puede:</p>
            <ul>
                <li>Ver los resultados en el portal de pacientes</li>
                <li>Retirar los resultados impresos en recepción</li>
                <li>Solicitar envío por email (PDF)</li>
            </ul>
            <p>Si tiene dudas sobre los resultados, por favor agende una consulta con su médico.</p>
            <p><a href="{{portal_link}}" style="background: #2563eb; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px; display: inline-block;">Ver Resultados</a></p>
            <p>Saludos,<br>{{clinic_name}}<br>Tel: {{clinic_phone}}</p>
        </div>
    </body>
    </html>',

    -- WhatsApp
    '🔬 *Resultados Disponibles*

Hola {{patient_first_name}},

Sus resultados de laboratorio del {{order_date}} están listos:

📋 {{test_name}}

Puede verlos en el portal o retirarlos en la clínica.

Para consultas, agende turno con su médico.

{{clinic_name}}',

    ARRAY['patient_first_name', 'patient_full_name', 'test_name', 'order_date',
          'clinic_name', 'clinic_phone', 'portal_link'],

    'es',
    true
);

-- Template 5: Payment reminder
INSERT INTO communication_templates (
    name, template_type, sms_content, email_subject, email_html, whatsapp_content,
    available_variables, language, is_system
) VALUES (
    'Recordatorio de pago',
    'payment_reminder',

    -- SMS
    'Recordatorio: tiene un saldo pendiente de ${{amount}} (Factura {{invoice_number}}). Puede pagar en {{clinic_name}} o por Mercado Pago. Tel: {{clinic_phone}}',

    -- Email subject
    'Recordatorio de pago - Factura {{invoice_number}}',

    -- Email HTML
    '<html>
    <body style="font-family: Arial, sans-serif;">
        <div style="max-width: 600px; margin: 0 auto; padding: 20px;">
            <h2 style="color: #2563eb;">Recordatorio de Pago</h2>
            <p>Hola <strong>{{patient_full_name}}</strong>,</p>
            <p>Le recordamos que tiene un saldo pendiente:</p>
            <div style="background: #f3f4f6; padding: 15px; border-radius: 5px; margin: 20px 0;">
                <p><strong>Factura:</strong> {{invoice_number}}</p>
                <p><strong>Fecha:</strong> {{invoice_date}}</p>
                <p><strong>Concepto:</strong> {{invoice_description}}</p>
                <p><strong>Monto:</strong> ${{amount}}</p>
                <p><strong>Vencimiento:</strong> {{due_date}}</p>
            </div>
            <p><strong>Formas de pago:</strong></p>
            <ul>
                <li>En recepción: efectivo, débito, crédito</li>
                <li>Transferencia bancaria</li>
                <li>Mercado Pago (link abajo)</li>
            </ul>
            <p><a href="{{payment_link}}" style="background: #00a650; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px; display: inline-block;">Pagar con Mercado Pago</a></p>
            <p>Cualquier consulta, comuníquese al {{clinic_phone}}</p>
            <p>Saludos,<br>{{clinic_name}}</p>
        </div>
    </body>
    </html>',

    -- WhatsApp
    '💳 *Recordatorio de Pago*

Hola {{patient_first_name}},

Tiene un saldo pendiente:

📄 Factura: {{invoice_number}}
💰 Monto: ${{amount}}
📅 Vencimiento: {{due_date}}

*Formas de pago:*
• En clínica (efectivo, tarjeta)
• Transferencia
• Mercado Pago: {{payment_link}}

{{clinic_name}}
Tel: {{clinic_phone}}',

    ARRAY['patient_first_name', 'patient_full_name', 'invoice_number', 'invoice_date',
          'invoice_description', 'amount', 'due_date', 'payment_link', 'clinic_name', 'clinic_phone'],

    'es',
    true
);
```

### 1.3 Patient Communication Preferences

#### 1.3.1 Patient Preference Settings

```sql
CREATE TABLE patient_communication_preferences (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id) ON DELETE CASCADE,

    -- Global opt-in/out
    allow_communications    BOOLEAN DEFAULT true,

    -- Channel preferences (priority order)
    preferred_channels      VARCHAR(50)[] DEFAULT ARRAY['whatsapp', 'sms', 'email'], -- Priority order

    -- Channel-specific opt-in
    allow_sms               BOOLEAN DEFAULT true,
    allow_email             BOOLEAN DEFAULT true,
    allow_whatsapp          BOOLEAN DEFAULT true,
    allow_phone_calls       BOOLEAN DEFAULT false, -- Opt-in for calls

    -- Contact information
    sms_phone               VARCHAR(50), -- May differ from main phone
    whatsapp_phone          VARCHAR(50),

    -- Quiet hours
    quiet_hours_enabled     BOOLEAN DEFAULT false,
    quiet_hours_start       TIME DEFAULT '22:00',
    quiet_hours_end         TIME DEFAULT '08:00',

    -- Notification types
    appointment_reminders   BOOLEAN DEFAULT true,
    appointment_confirmations BOOLEAN DEFAULT true,
    lab_results             BOOLEAN DEFAULT true,
    prescription_refills    BOOLEAN DEFAULT false,
    payment_reminders       BOOLEAN DEFAULT true,
    marketing_messages      BOOLEAN DEFAULT false,
    health_tips             BOOLEAN DEFAULT false,

    -- Reminder timing preferences
    reminder_timing         VARCHAR(50) DEFAULT 'default', -- 'default', 'early', 'minimal'
    /*
    - default: 24h + 2h
    - early: 1 week + 3 days + 24h
    - minimal: 24h only
    */

    -- Language preference
    language                VARCHAR(10) DEFAULT 'es',

    -- Special needs
    needs_large_text        BOOLEAN DEFAULT false,
    needs_phone_call        BOOLEAN DEFAULT false, -- Elderly, can't read SMS

    -- Metadata
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by              BIGINT REFERENCES users(id),

    UNIQUE(patient_id),
    INDEX idx_patient_comm_prefs_patient (patient_id)
);

-- Default preferences for all patients (trigger on patient creation)
CREATE OR REPLACE FUNCTION create_default_communication_preferences()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO patient_communication_preferences (patient_id)
    VALUES (NEW.id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_create_comm_preferences
AFTER INSERT ON patients
FOR EACH ROW
EXECUTE FUNCTION create_default_communication_preferences();
```

### 1.4 Reminder Tracking and Logs

#### 1.4.1 Communication Log

```sql
CREATE TABLE communication_logs (
    id                      BIGSERIAL PRIMARY KEY,

    -- Reference
    patient_id              BIGINT REFERENCES patients(id),
    appointment_id          BIGINT REFERENCES appointments(id),
    invoice_id              BIGINT REFERENCES invoices(id),

    -- Communication details
    type                    VARCHAR(50) NOT NULL, -- 'appointment_reminder', 'confirmation', 'lab_result', etc.
    channel                 VARCHAR(50) NOT NULL, -- 'sms', 'email', 'whatsapp', 'phone'

    -- Template used
    template_id             BIGINT REFERENCES communication_templates(id),

    -- Content sent
    subject                 VARCHAR(500),
    message_content         TEXT,

    -- Recipient
    recipient_phone         VARCHAR(50),
    recipient_email         VARCHAR(255),

    -- Delivery status
    status                  VARCHAR(50) DEFAULT 'pending', -- 'pending', 'sent', 'delivered', 'failed', 'bounced'

    -- External references (from Twilio, SendGrid, etc.)
    external_id             VARCHAR(200), -- Message SID, email ID, etc.
    external_status         VARCHAR(100),

    -- Delivery timestamps
    sent_at                 TIMESTAMP,
    delivered_at            TIMESTAMP,
    opened_at               TIMESTAMP, -- Email opened
    clicked_at              TIMESTAMP, -- Link clicked
    failed_at               TIMESTAMP,

    -- Error details
    error_code              VARCHAR(50),
    error_message           TEXT,

    -- Response tracking (for 2-way communication)
    response_received       BOOLEAN DEFAULT false,
    response_content        TEXT,
    response_at             TIMESTAMP,

    -- Cost tracking
    cost                    DECIMAL(10,4), -- Cost per message

    -- Metadata
    sent_by                 BIGINT REFERENCES users(id), -- If manually sent
    is_automated            BOOLEAN DEFAULT true,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_comm_logs_patient (patient_id),
    INDEX idx_comm_logs_appointment (appointment_id),
    INDEX idx_comm_logs_status (status),
    INDEX idx_comm_logs_created (created_at),
    INDEX idx_comm_logs_type (type),
    INDEX idx_comm_logs_channel (channel)
);
```

### 1.5 Confirmation Workflow

#### 1.5.1 Appointment Confirmation Tracking

```sql
-- Add to appointments table
ALTER TABLE appointments ADD COLUMN IF NOT EXISTS confirmation_status VARCHAR(50) DEFAULT 'pending';
-- Values: 'pending', 'confirmed', 'declined', 'expired'

ALTER TABLE appointments ADD COLUMN IF NOT EXISTS confirmed_at TIMESTAMP;
ALTER TABLE appointments ADD COLUMN IF NOT EXISTS confirmed_by VARCHAR(50); -- 'patient', 'staff', 'auto'
ALTER TABLE appointments ADD COLUMN IF NOT EXISTS confirmation_method VARCHAR(50); -- 'sms', 'whatsapp', 'portal', 'phone'
```

#### 1.5.2 Confirmation Processing Logic

```javascript
/**
 * Process confirmation response from patient
 */
async function processConfirmationResponse(response) {
  // SMS response parsing
  const smsKeywords = {
    confirm: ['C', 'CONFIRMO', 'SI', 'OK', 'CONFIRMAR', 'YES'],
    cancel: ['X', 'CANCELAR', 'NO', 'CANCEL'],
    reschedule: ['R', 'REPROGRAMAR', 'CAMBIAR', 'RESCHEDULE']
  };

  const responseText = response.body.toUpperCase().trim();
  const appointmentId = extractAppointmentId(response); // From context

  if (smsKeywords.confirm.includes(responseText)) {
    await confirmAppointment(appointmentId, 'patient', 'sms');
    await sendConfirmationReceipt(appointmentId);
    return 'confirmed';
  }

  if (smsKeywords.cancel.includes(responseText)) {
    await cancelAppointment(appointmentId, 'patient_requested');
    await sendCancellationReceipt(appointmentId);
    await notifyStaff(appointmentId, 'cancelled_by_patient');
    return 'cancelled';
  }

  if (smsKeywords.reschedule.includes(responseText)) {
    await sendRescheduleInstructions(appointmentId);
    return 'reschedule_requested';
  }

  // Unrecognized response
  await sendHelpMessage(response.from);
  return 'unrecognized';
}

/**
 * Confirm appointment
 */
async function confirmAppointment(appointmentId, confirmedBy, method) {
  await Appointment.update({
    confirmation_status: 'confirmed',
    confirmed_at: new Date(),
    confirmed_by: confirmedBy,
    confirmation_method: method
  }, {
    where: { id: appointmentId }
  });

  // Log the confirmation
  await AuditLog.create({
    entity_type: 'appointment',
    entity_id: appointmentId,
    action: 'confirmed',
    details: { confirmed_by: confirmedBy, method: method }
  });
}
```

### 1.6 Automated Reminder Job

#### 1.6.1 Reminder Processing Algorithm

```javascript
/**
 * Cron job: Run every hour to send reminders
 */
async function processReminders() {
  const now = new Date();

  // Get all active reminder configurations
  const reminderConfigs = await ReminderConfiguration.findAll({
    where: { is_active: true },
    order: [['priority', 'DESC']]
  });

  // Get appointments that need reminders
  const appointments = await Appointment.findAll({
    where: {
      scheduled_at: {
        [Op.gte]: now,
        [Op.lte]: new Date(now.getTime() + 7 * 24 * 60 * 60 * 1000) // Next 7 days
      },
      status: {
        [Op.notIn]: ['cancelled', 'completed', 'no_show']
      }
    },
    include: [
      { model: Patient, include: [PatientCommunicationPreferences] },
      { model: User, as: 'Provider' },
      { model: AppointmentType },
      { model: Location }
    ]
  });

  for (const appointment of appointments) {
    // Find applicable reminder configuration
    const config = findApplicableConfig(appointment, reminderConfigs);
    if (!config) continue;

    // Check patient preferences
    const prefs = appointment.Patient.PatientCommunicationPreference;
    if (!prefs || !prefs.allow_communications || !prefs.appointment_reminders) {
      continue;
    }

    // Process each reminder in configuration
    for (const reminder of config.reminders) {
      const shouldSend = await shouldSendReminder(
        appointment,
        reminder,
        now
      );

      if (shouldSend) {
        await sendReminder(appointment, reminder, prefs);
      }
    }
  }
}

/**
 * Check if reminder should be sent
 */
async function shouldSendReminder(appointment, reminder, now) {
  const appointmentTime = new Date(appointment.scheduled_at);
  const timingMs = convertToMilliseconds(reminder.timing_value, reminder.timing_unit);
  const targetTime = new Date(appointmentTime.getTime() - timingMs);

  // Check if we're in the right time window (±30 min)
  const windowStart = new Date(targetTime.getTime() - 30 * 60 * 1000);
  const windowEnd = new Date(targetTime.getTime() + 30 * 60 * 1000);

  if (now < windowStart || now > windowEnd) {
    return false;
  }

  // Check if already sent
  const alreadySent = await CommunicationLog.findOne({
    where: {
      appointment_id: appointment.id,
      type: 'appointment_reminder',
      template_id: reminder.template_id,
      status: { [Op.in]: ['sent', 'delivered'] }
    }
  });

  if (alreadySent) {
    return false;
  }

  return true;
}

/**
 * Send reminder via preferred channels
 */
async function sendReminder(appointment, reminder, preferences) {
  const template = await CommunicationTemplate.findByPk(reminder.template_id);

  // Determine channels to use based on config and patient preference
  const channels = reminder.channels.filter(channel => {
    if (channel === 'sms') return preferences.allow_sms;
    if (channel === 'email') return preferences.allow_email;
    if (channel === 'whatsapp') return preferences.allow_whatsapp;
    return false;
  });

  // Use patient's preferred channel order
  const orderedChannels = orderByPreference(channels, preferences.preferred_channels);

  // Check quiet hours
  if (preferences.quiet_hours_enabled && isInQuietHours(preferences)) {
    // Schedule for end of quiet hours
    await scheduleReminder(appointment, reminder, preferences.quiet_hours_end);
    return;
  }

  // Send via each channel
  for (const channel of orderedChannels) {
    try {
      const content = renderTemplate(template, appointment, channel);

      if (channel === 'sms') {
        await sendSMS(
          preferences.sms_phone || appointment.Patient.phone,
          content,
          appointment.id,
          template.id
        );
      } else if (channel === 'email') {
        await sendEmail(
          appointment.Patient.email,
          template.email_subject,
          content,
          appointment.id,
          template.id
        );
      } else if (channel === 'whatsapp') {
        await sendWhatsApp(
          preferences.whatsapp_phone || appointment.Patient.phone,
          content,
          appointment.id,
          template.id
        );
      }

    } catch (error) {
      console.error(`Failed to send ${channel} reminder:`, error);
      // Log error but continue with other channels
      await CommunicationLog.create({
        appointment_id: appointment.id,
        type: 'appointment_reminder',
        channel: channel,
        template_id: template.id,
        status: 'failed',
        error_message: error.message
      });
    }
  }
}

/**
 * Render template with appointment data
 */
function renderTemplate(template, appointment, channel) {
  const variables = {
    patient_first_name: appointment.Patient.first_name,
    patient_full_name: `${appointment.Patient.first_name} ${appointment.Patient.last_name}`,
    patient_dni: appointment.Patient.dni,
    appointment_date: formatDate(appointment.scheduled_at, 'DD/MM/YYYY'),
    appointment_date_long: formatDate(appointment.scheduled_at, 'dddd DD [de] MMMM [de] YYYY'),
    appointment_time: formatTime(appointment.scheduled_at),
    doctor_name: `${appointment.Provider.first_name} ${appointment.Provider.last_name}`,
    doctor_title: appointment.Provider.title || 'Dr.',
    doctor_specialty: appointment.Provider.specialty,
    appointment_type: appointment.AppointmentType?.name,
    location_name: appointment.Location?.name,
    location_address: appointment.Location?.address,
    clinic_name: getSystemSetting('clinic_name'),
    clinic_phone: getSystemSetting('clinic_phone'),
    clinic_address: getSystemSetting('clinic_address'),
    confirmation_link: generateConfirmationLink(appointment.id),
    portal_link: getSystemSetting('portal_url')
  };

  let content;
  if (channel === 'sms') {
    content = template.sms_content;
  } else if (channel === 'email') {
    content = template.email_html || template.email_text;
  } else if (channel === 'whatsapp') {
    content = template.whatsapp_content;
  }

  // Replace variables
  for (const [key, value] of Object.entries(variables)) {
    const regex = new RegExp(`{{${key}}}`, 'g');
    content = content.replace(regex, value || '');
  }

  return content;
}
```

---

## 2. MASS COMMUNICATION

### 2.1 Bulk Messaging

```sql
CREATE TABLE bulk_messages (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Message details
    template_id             BIGINT REFERENCES communication_templates(id),
    type                    VARCHAR(50) NOT NULL, -- 'announcement', 'marketing', 'health_campaign', 'emergency'

    -- Channel
    channels                VARCHAR(50)[] NOT NULL, -- ['sms', 'email', 'whatsapp']

    -- Recipients
    recipient_type          VARCHAR(50) NOT NULL, -- 'all', 'filter', 'custom_list'
    recipient_filter        JSONB, -- Filter criteria
    /* Example:
    {
      "age_min": 40,
      "age_max": 65,
      "insurance_ids": [1, 2, 3],
      "last_visit_after": "2024-01-01",
      "gender": "F",
      "has_diagnosis": "E11" // Diabetes ICD-10
    }
    */
    recipient_count         INTEGER,

    -- Scheduling
    schedule_type           VARCHAR(50) DEFAULT 'immediate', -- 'immediate', 'scheduled', 'recurring'
    scheduled_at            TIMESTAMP,
    recurrence_pattern      JSONB, -- For recurring messages

    -- Status
    status                  VARCHAR(50) DEFAULT 'draft', -- 'draft', 'scheduled', 'sending', 'sent', 'cancelled'
    sent_count              INTEGER DEFAULT 0,
    delivered_count         INTEGER DEFAULT 0,
    failed_count            INTEGER DEFAULT 0,

    -- Metadata
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    sent_at                 TIMESTAMP,
    completed_at            TIMESTAMP,

    INDEX idx_bulk_messages_status (status),
    INDEX idx_bulk_messages_scheduled (scheduled_at)
);

CREATE TABLE bulk_message_recipients (
    id                      BIGSERIAL PRIMARY KEY,
    bulk_message_id         BIGINT NOT NULL REFERENCES bulk_messages(id) ON DELETE CASCADE,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),

    -- Delivery status per recipient
    status                  VARCHAR(50) DEFAULT 'pending',
    communication_log_id    BIGINT REFERENCES communication_logs(id),

    sent_at                 TIMESTAMP,

    INDEX idx_bulk_recipients_message (bulk_message_id),
    INDEX idx_bulk_recipients_patient (patient_id)
);
```

### 2.2 Campaign Examples

```sql
-- Example: Flu vaccination campaign
INSERT INTO bulk_messages (
    name, description, type, channels, recipient_type, recipient_filter, scheduled_at, created_by
) VALUES (
    'Campaña Vacunación Antigripal 2025',
    'Invitación a pacientes mayores de 60 años para vacunación antigripal',
    'health_campaign',
    ARRAY['sms', 'email'],
    'filter',
    '{
      "age_min": 60,
      "last_visit_after": "2024-01-01"
    }'::jsonb,
    '2025-03-01 09:00:00',
    1
);

-- Example: Appointment availability announcement
INSERT INTO bulk_messages (
    name, type, channels, recipient_type, recipient_filter
) VALUES (
    'Nuevos turnos disponibles - Dr. Martinez',
    'announcement',
    ARRAY['sms', 'whatsapp'],
    'filter',
    '{
      "last_appointment_provider_id": 5,
      "last_visit_after": "2024-06-01"
    }'::jsonb
);
```

---

## 3. TWO-WAY COMMUNICATION

### 3.1 Patient Messaging (Internal)

```sql
CREATE TABLE patient_messages (
    id                      BIGSERIAL PRIMARY KEY,

    -- Thread
    thread_id               VARCHAR(100) NOT NULL, -- Group related messages
    parent_message_id       BIGINT REFERENCES patient_messages(id), -- For replies

    -- Participants
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    staff_id                BIGINT REFERENCES users(id),

    -- Message details
    direction               VARCHAR(50) NOT NULL, -- 'inbound' (from patient), 'outbound' (to patient)
    channel                 VARCHAR(50) NOT NULL, -- 'portal', 'sms', 'whatsapp', 'email'

    subject                 VARCHAR(500),
    message_content         TEXT NOT NULL,

    -- Attachments
    has_attachments         BOOLEAN DEFAULT false,
    attachment_ids          BIGINT[], -- References documents table

    -- Status
    status                  VARCHAR(50) DEFAULT 'sent', -- 'draft', 'sent', 'delivered', 'read'
    is_urgent               BOOLEAN DEFAULT false,

    -- Related entities
    appointment_id          BIGINT REFERENCES appointments(id),
    encounter_id            BIGINT REFERENCES encounters(id),

    -- Timestamps
    sent_at                 TIMESTAMP,
    delivered_at            TIMESTAMP,
    read_at                 TIMESTAMP,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_patient_messages_thread (thread_id),
    INDEX idx_patient_messages_patient (patient_id),
    INDEX idx_patient_messages_staff (staff_id),
    INDEX idx_patient_messages_status (status),
    INDEX idx_patient_messages_created (created_at DESC)
);
```

### 3.2 Message Assignment and Routing

```sql
CREATE TABLE message_routing_rules (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,

    -- Conditions
    keyword_triggers        TEXT[], -- Array of keywords
    sender_insurance_id     BIGINT REFERENCES insurances(id),
    sender_provider_id      BIGINT REFERENCES users(id), -- Patient's assigned provider
    message_type            VARCHAR(50), -- 'appointment', 'billing', 'clinical', 'general'

    -- Routing action
    assign_to_user_id       BIGINT REFERENCES users(id),
    assign_to_role          VARCHAR(50),

    -- Auto-response
    send_auto_response      BOOLEAN DEFAULT false,
    auto_response_template_id BIGINT REFERENCES communication_templates(id),

    priority                INTEGER DEFAULT 0,
    is_active               BOOLEAN DEFAULT true,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Example rules
INSERT INTO message_routing_rules (name, keyword_triggers, assign_to_role, send_auto_response) VALUES
('Appointment requests', ARRAY['turno', 'cita', 'appointment', 'agendar'], 'RECEPTIONIST', true),
('Billing inquiries', ARRAY['pago', 'factura', 'payment', 'invoice'], 'BILLING', true),
('Clinical questions', ARRAY['resultado', 'receta', 'prescription', 'result'], 'NURSE', true);
```

---

## 4. NOTIFICATION CENTER (Staff)

### 4.1 Staff Notification System

```sql
-- Already exists in DATABASE_SCHEMA.md, but enhanced here
ALTER TABLE notifications ADD COLUMN IF NOT EXISTS priority VARCHAR(50) DEFAULT 'normal';
-- Values: 'low', 'normal', 'high', 'urgent'

ALTER TABLE notifications ADD COLUMN IF NOT EXISTS action_url VARCHAR(500);
-- URL to navigate to when clicking notification

ALTER TABLE notifications ADD COLUMN IF NOT EXISTS action_label VARCHAR(100);
-- Button text, e.g., "View Appointment", "Approve Order"

ALTER TABLE notifications ADD COLUMN IF NOT EXISTS category VARCHAR(50);
-- 'appointment', 'clinical', 'billing', 'system', 'task'

ALTER TABLE notifications ADD COLUMN IF NOT EXISTS expires_at TIMESTAMP;
-- Notifications can expire (e.g., "Patient checked in" expires after appointment time)
```

### 4.2 Notification Types

```sql
-- Notification type definitions
CREATE TABLE notification_types (
    id                      BIGSERIAL PRIMARY KEY,
    code                    VARCHAR(100) UNIQUE NOT NULL,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    category                VARCHAR(50) NOT NULL,
    default_priority        VARCHAR(50) DEFAULT 'normal',

    -- Template
    title_template          VARCHAR(500),
    message_template        TEXT,

    -- Auto-clear conditions
    auto_clear_on_action    BOOLEAN DEFAULT false,
    expires_after_minutes   INTEGER, -- Auto-expire

    -- Delivery preferences
    in_app                  BOOLEAN DEFAULT true,
    email                   BOOLEAN DEFAULT false,
    sms                     BOOLEAN DEFAULT false,

    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Default notification types
INSERT INTO notification_types (code, name, category, default_priority, title_template, message_template, in_app, email) VALUES
('patient_checked_in', 'Patient Checked In', 'appointment', 'high',
 'Patient Arrived', '{{patient_name}} has checked in for {{appointment_time}} appointment', true, false),

('appointment_cancelled', 'Appointment Cancelled', 'appointment', 'normal',
 'Appointment Cancelled', '{{patient_name}} cancelled appointment on {{appointment_date}}', true, true),

('lab_result_critical', 'Critical Lab Result', 'clinical', 'urgent',
 'CRITICAL: Lab Result', 'Critical result for {{patient_name}}: {{test_name}}', true, true),

('lab_result_ready', 'Lab Result Ready', 'clinical', 'normal',
 'Lab Result Available', 'Results ready for {{patient_name}}: {{test_name}}', true, false),

('prescription_refill_request', 'Prescription Refill Request', 'clinical', 'normal',
 'Refill Requested', '{{patient_name}} requested refill for {{medication_name}}', true, false),

('payment_received', 'Payment Received', 'billing', 'low',
 'Payment Processed', 'Payment of ${{amount}} received from {{patient_name}}', true, false),

('invoice_overdue', 'Overdue Invoice', 'billing', 'normal',
 'Overdue Payment', '{{patient_name}} invoice #{{invoice_number}} is {{days_overdue}} days overdue', true, false),

('low_stock_alert', 'Low Stock', 'system', 'high',
 'Low Inventory', '{{item_name}} is low in stock ({{quantity}} remaining)', true, true),

('task_assigned', 'Task Assigned', 'task', 'normal',
 'New Task', 'You have been assigned: {{task_title}}', true, false),

('task_overdue', 'Overdue Task', 'task', 'high',
 'Task Overdue', 'Task "{{task_title}}" is overdue by {{days}} days', true, true);
```

---

## 5. EMERGENCY COMMUNICATIONS

### 5.1 Emergency Alert System

```sql
CREATE TABLE emergency_alerts (
    id                      BIGSERIAL PRIMARY KEY,

    -- Alert details
    title                   VARCHAR(200) NOT NULL,
    message                 TEXT NOT NULL,
    severity                VARCHAR(50) NOT NULL, -- 'info', 'warning', 'critical'

    -- Recipients
    recipient_type          VARCHAR(50) NOT NULL, -- 'all_staff', 'role', 'specific_users', 'all_patients', 'patient_filter'
    recipient_roles         VARCHAR(50)[],
    recipient_user_ids      BIGINT[],
    recipient_patient_filter JSONB,

    -- Channels (force send even if opted out)
    channels                VARCHAR(50)[] DEFAULT ARRAY['sms', 'email', 'push'],

    -- Status
    status                  VARCHAR(50) DEFAULT 'draft',
    sent_count              INTEGER DEFAULT 0,

    -- Metadata
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    sent_at                 TIMESTAMP,

    INDEX idx_emergency_alerts_status (status)
);

-- Example: Clinic closure due to emergency
INSERT INTO emergency_alerts (title, message, severity, recipient_type, channels, created_by) VALUES
('Cierre de clínica - Emergencia',
 'Estimados pacientes: La clínica permanecerá cerrada hoy por razones de fuerza mayor. Los turnos serán reprogramados. Disculpe las molestias.',
 'critical',
 'all_patients',
 ARRAY['sms', 'email', 'whatsapp'],
 1
);
```

---

## 6. ANALYTICS AND REPORTING

### 6.1 Communication Analytics

```sql
CREATE VIEW communication_analytics AS
SELECT
    DATE(created_at) as date,
    type,
    channel,
    COUNT(*) as total_sent,
    COUNT(*) FILTER (WHERE status = 'delivered') as delivered,
    COUNT(*) FILTER (WHERE status = 'failed') as failed,
    COUNT(*) FILTER (WHERE opened_at IS NOT NULL) as opened,
    COUNT(*) FILTER (WHERE clicked_at IS NOT NULL) as clicked,
    COUNT(*) FILTER (WHERE response_received) as responses,
    SUM(cost) as total_cost,
    ROUND(AVG(EXTRACT(EPOCH FROM (delivered_at - sent_at))), 2) as avg_delivery_time_seconds
FROM communication_logs
GROUP BY DATE(created_at), type, channel;

-- Reminder effectiveness report
CREATE VIEW reminder_effectiveness AS
SELECT
    a.id as appointment_id,
    a.scheduled_at,
    a.status as appointment_status,
    a.confirmation_status,
    COUNT(cl.id) as reminders_sent,
    COUNT(cl.id) FILTER (WHERE cl.status = 'delivered') as reminders_delivered,
    BOOL_OR(cl.response_received) as patient_responded
FROM appointments a
LEFT JOIN communication_logs cl ON cl.appointment_id = a.id AND cl.type = 'appointment_reminder'
WHERE a.scheduled_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY a.id, a.scheduled_at, a.status, a.confirmation_status;
```

---

*Communication & Notification Configuration Version: 1.0*
*Last Updated: 2025-11-16*
*Comprehensive communication and notification system for patient engagement*
