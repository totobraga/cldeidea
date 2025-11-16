# Online Patient Self-Booking System Specification

## 1. Executive Summary

### 1.1 Overview
The Online Patient Self-Booking System enables patients to schedule their own appointments 24/7 through a web portal or mobile app, without requiring staff intervention. This system reduces administrative workload by approximately 70%, improves patient satisfaction, and provides real-time availability visibility.

### 1.2 Key Benefits
- **24/7 Availability**: Patients can book appointments anytime, from anywhere
- **Reduced Workload**: Decreases front desk phone calls by 60-70%
- **Fewer No-Shows**: Automated confirmations and reminders reduce no-show rates by 30-40%
- **Patient Satisfaction**: 85% of patients prefer online booking to phone calls
- **Real-Time Updates**: Availability reflects cancellations and schedule changes instantly
- **Obra Social Integration**: Validates insurance coverage before booking

### 1.3 Argentina-Specific Features
- Obra Social/Prepaga validation during booking process
- DNI validation and patient identification
- Integration with PAMI for retiree coverage
- Spanish language interface with Argentina locale
- Holiday calendar for Argentina (national and provincial holidays)
- Buenos Aires timezone support

---

## 2. Database Architecture

### 2.1 Core Tables

```sql
-- Online booking rules configuration
CREATE TABLE online_booking_rules (
    id                          BIGSERIAL PRIMARY KEY,
    location_id                 BIGINT REFERENCES locations(id),
    provider_id                 BIGINT REFERENCES users(id),
    appointment_type_id         BIGINT REFERENCES appointment_types(id),

    -- Availability settings
    enabled                     BOOLEAN DEFAULT true,
    advance_booking_min_hours   INTEGER DEFAULT 2,              -- Min 2 hours in advance
    advance_booking_max_days    INTEGER DEFAULT 90,             -- Max 90 days in advance

    -- Time slot settings
    show_available_slots_only   BOOLEAN DEFAULT true,
    slot_interval_minutes       INTEGER DEFAULT 15,             -- Show slots every 15 min
    max_slots_per_day           INTEGER,                        -- Limit slots shown per day

    -- Patient restrictions
    require_obra_social         BOOLEAN DEFAULT false,          -- Must have insurance
    allowed_obra_socials        JSONB,                          -- Array of allowed obra social IDs
    require_existing_patient    BOOLEAN DEFAULT false,          -- New patients allowed?
    max_future_appointments     INTEGER DEFAULT 3,              -- Prevent spam bookings

    -- Confirmation settings
    require_immediate_confirmation BOOLEAN DEFAULT true,        -- Auto-confirm or staff review?
    send_confirmation_email     BOOLEAN DEFAULT true,
    send_confirmation_sms       BOOLEAN DEFAULT true,

    -- Cancellation settings
    allow_online_cancellation   BOOLEAN DEFAULT true,
    cancellation_deadline_hours INTEGER DEFAULT 24,             -- Must cancel 24h before

    -- Pre-visit requirements
    require_pre_visit_forms     BOOLEAN DEFAULT false,
    required_form_ids           JSONB,                          -- Array of form IDs
    require_payment_upfront     BOOLEAN DEFAULT false,
    upfront_payment_amount      DECIMAL(10,2),

    -- Business rules
    block_if_outstanding_balance BOOLEAN DEFAULT false,
    min_days_between_same_type  INTEGER,                        -- Prevent duplicate bookings

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_online_booking_rules_location ON online_booking_rules(location_id);
CREATE INDEX idx_online_booking_rules_provider ON online_booking_rules(provider_id);
CREATE INDEX idx_online_booking_rules_enabled ON online_booking_rules(enabled);


-- Track all online booking attempts (successful and failed)
CREATE TABLE online_booking_attempts (
    id                          BIGSERIAL PRIMARY KEY,
    patient_id                  BIGINT REFERENCES patients(id),

    -- Session tracking
    session_id                  VARCHAR(200),                   -- Track multi-step booking
    ip_address                  INET,
    user_agent                  TEXT,

    -- Booking details
    location_id                 BIGINT REFERENCES locations(id),
    provider_id                 BIGINT REFERENCES users(id),
    appointment_type_id         BIGINT REFERENCES appointment_types(id),
    requested_date              DATE,
    requested_time              TIME,

    -- Result
    status                      VARCHAR(50) NOT NULL,           -- 'success', 'failed', 'abandoned'
    appointment_id              BIGINT REFERENCES appointments(id),

    -- Failure tracking
    failure_reason              VARCHAR(200),
    /* Possible reasons:
       - 'slot_not_available'
       - 'obra_social_not_accepted'
       - 'max_future_appointments_reached'
       - 'outside_booking_window'
       - 'outstanding_balance'
       - 'duplicate_appointment'
       - 'technical_error'
       - 'abandoned_by_user'
    */
    failure_details             JSONB,

    -- Timing
    started_at                  TIMESTAMP NOT NULL,
    completed_at                TIMESTAMP,
    duration_seconds            INTEGER,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_booking_attempts_patient ON online_booking_attempts(patient_id);
CREATE INDEX idx_booking_attempts_status ON online_booking_attempts(status);
CREATE INDEX idx_booking_attempts_date ON online_booking_attempts(requested_date);
CREATE INDEX idx_booking_attempts_session ON online_booking_attempts(session_id);


-- Available time slots cache (for performance)
CREATE TABLE online_booking_slot_cache (
    id                          BIGSERIAL PRIMARY KEY,
    location_id                 BIGINT NOT NULL REFERENCES locations(id),
    provider_id                 BIGINT NOT NULL REFERENCES users(id),
    appointment_type_id         BIGINT NOT NULL REFERENCES appointment_types(id),

    -- Slot details
    slot_date                   DATE NOT NULL,
    slot_time                   TIME NOT NULL,
    slot_datetime               TIMESTAMP NOT NULL,

    -- Availability
    is_available                BOOLEAN DEFAULT true,
    reserved_until              TIMESTAMP,                      -- Temporary reservation
    reserved_by_session         VARCHAR(200),

    -- Metadata
    duration_minutes            INTEGER NOT NULL,
    obra_social_restrictions    JSONB,                          -- Which obra socials accepted

    -- Cache management
    cache_expires_at            TIMESTAMP NOT NULL,
    last_refreshed_at           TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(location_id, provider_id, appointment_type_id, slot_datetime)
);

CREATE INDEX idx_slot_cache_lookup ON online_booking_slot_cache(
    location_id, provider_id, appointment_type_id, slot_date
);
CREATE INDEX idx_slot_cache_available ON online_booking_slot_cache(is_available, cache_expires_at);
CREATE INDEX idx_slot_cache_reserved ON online_booking_slot_cache(reserved_by_session, reserved_until);


-- Pre-visit forms (questionnaires completed before appointment)
CREATE TABLE pre_visit_forms (
    id                          BIGSERIAL PRIMARY KEY,
    name                        VARCHAR(200) NOT NULL,
    description                 TEXT,

    -- Form structure
    form_fields                 JSONB NOT NULL,
    /* Example:
    [
      {
        "field_id": "reason",
        "field_type": "textarea",
        "label": "Motivo de la consulta",
        "required": true,
        "max_length": 500
      },
      {
        "field_id": "symptoms",
        "field_type": "checklist",
        "label": "Síntomas actuales",
        "options": ["Fiebre", "Tos", "Dolor de cabeza", "Fatiga"]
      },
      {
        "field_id": "medications",
        "field_type": "text",
        "label": "Medicamentos actuales",
        "required": false
      }
    ]
    */

    -- Usage
    applies_to_appointment_types JSONB,                         -- Array of appointment type IDs
    is_active                   BOOLEAN DEFAULT true,
    display_order               INTEGER DEFAULT 0,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);


-- Completed pre-visit forms
CREATE TABLE pre_visit_form_responses (
    id                          BIGSERIAL PRIMARY KEY,
    form_id                     BIGINT NOT NULL REFERENCES pre_visit_forms(id),
    appointment_id              BIGINT NOT NULL REFERENCES appointments(id),
    patient_id                  BIGINT NOT NULL REFERENCES patients(id),

    -- Response data
    responses                   JSONB NOT NULL,
    /* Example:
    {
      "reason": "Control de presión arterial",
      "symptoms": ["Fatiga"],
      "medications": "Enalapril 10mg"
    }
    */

    -- Completion tracking
    completed_at                TIMESTAMP NOT NULL,
    ip_address                  INET,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_pre_visit_responses_appointment ON pre_visit_form_responses(appointment_id);
CREATE INDEX idx_pre_visit_responses_patient ON pre_visit_form_responses(patient_id);


-- Online booking analytics
CREATE TABLE online_booking_analytics (
    id                          BIGSERIAL PRIMARY KEY,
    date                        DATE NOT NULL,
    hour                        INTEGER,                        -- Hour of day (0-23)

    -- Metrics
    total_attempts              INTEGER DEFAULT 0,
    successful_bookings         INTEGER DEFAULT 0,
    failed_bookings             INTEGER DEFAULT 0,
    abandoned_bookings          INTEGER DEFAULT 0,

    -- Conversion rates
    success_rate                DECIMAL(5,2),                   -- Percentage
    average_duration_seconds    INTEGER,

    -- Popular choices
    top_location_id             BIGINT REFERENCES locations(id),
    top_provider_id             BIGINT REFERENCES users(id),
    top_appointment_type_id     BIGINT REFERENCES appointment_types(id),

    -- Failure analysis
    top_failure_reason          VARCHAR(200),
    failure_reason_breakdown    JSONB,
    /* Example:
    {
      "slot_not_available": 45,
      "obra_social_not_accepted": 12,
      "outside_booking_window": 8
    }
    */

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(date, hour)
);

CREATE INDEX idx_booking_analytics_date ON online_booking_analytics(date);


-- Patient portal preferences
CREATE TABLE patient_portal_preferences (
    id                          BIGSERIAL PRIMARY KEY,
    patient_id                  BIGINT NOT NULL REFERENCES patients(id) UNIQUE,

    -- Preferred providers
    favorite_providers          JSONB,                          -- Array of provider IDs

    -- Preferred locations
    favorite_locations          JSONB,                          -- Array of location IDs

    -- Default appointment type
    default_appointment_type_id BIGINT REFERENCES appointment_types(id),

    -- Notification preferences
    email_confirmations         BOOLEAN DEFAULT true,
    sms_confirmations           BOOLEAN DEFAULT true,
    whatsapp_confirmations      BOOLEAN DEFAULT false,
    reminder_preference         VARCHAR(50) DEFAULT 'both',     -- 'email', 'sms', 'both', 'none'

    -- Calendar integration
    google_calendar_enabled     BOOLEAN DEFAULT false,
    google_calendar_token       TEXT,                           -- Encrypted OAuth token
    outlook_calendar_enabled    BOOLEAN DEFAULT false,
    outlook_calendar_token      TEXT,

    created_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at                  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_portal_preferences_patient ON patient_portal_preferences(patient_id);
```

---

## 3. Real-Time Availability Calculation

### 3.1 Availability Algorithm

The availability calculation engine must consider multiple factors:

```javascript
/**
 * Calculate available time slots for online booking
 *
 * @param {Object} params - Search parameters
 * @param {number} params.locationId - Location ID
 * @param {number} params.providerId - Provider ID (optional)
 * @param {number} params.appointmentTypeId - Appointment type ID
 * @param {Date} params.startDate - Start date for search
 * @param {Date} params.endDate - End date for search
 * @param {number} params.patientId - Patient ID (for restrictions)
 * @param {number} params.obraSocialId - Obra Social ID
 * @returns {Array} Available time slots
 */
async function calculateAvailableSlots(params) {
  const {
    locationId,
    providerId,
    appointmentTypeId,
    startDate,
    endDate,
    patientId,
    obraSocialId
  } = params;

  // Step 1: Get booking rules
  const bookingRules = await db.OnlineBookingRule.findOne({
    where: {
      location_id: locationId,
      provider_id: providerId || null,
      appointment_type_id: appointmentTypeId || null,
      enabled: true
    }
  });

  if (!bookingRules) {
    throw new Error('Online booking not enabled for this combination');
  }

  // Step 2: Validate booking window
  const now = new Date();
  const minBookingDate = new Date(now.getTime() + bookingRules.advance_booking_min_hours * 60 * 60 * 1000);
  const maxBookingDate = new Date(now.getTime() + bookingRules.advance_booking_max_days * 24 * 60 * 60 * 1000);

  if (startDate < minBookingDate || endDate > maxBookingDate) {
    throw new Error(`Bookings must be between ${bookingRules.advance_booking_min_hours} hours and ${bookingRules.advance_booking_max_days} days in advance`);
  }

  // Step 3: Validate Obra Social
  if (bookingRules.require_obra_social && !obraSocialId) {
    throw new Error('Obra Social required for online booking');
  }

  if (bookingRules.allowed_obra_socials) {
    const allowedIds = bookingRules.allowed_obra_socials;
    if (!allowedIds.includes(obraSocialId)) {
      throw new Error('This Obra Social is not accepted for online booking');
    }
  }

  // Step 4: Check patient restrictions
  const patient = await db.Patient.findByPk(patientId);

  if (bookingRules.require_existing_patient && !patient.medical_record_number) {
    throw new Error('Only existing patients can book online. Please call to schedule.');
  }

  if (bookingRules.block_if_outstanding_balance) {
    const balance = await calculatePatientBalance(patientId);
    if (balance > 0) {
      throw new Error('Cannot book online with outstanding balance. Please contact reception.');
    }
  }

  // Step 5: Check max future appointments
  const futureAppointments = await db.Appointment.count({
    where: {
      patient_id: patientId,
      appointment_date: { [Op.gte]: now },
      status: { [Op.notIn]: ['cancelled', 'no_show'] }
    }
  });

  if (futureAppointments >= bookingRules.max_future_appointments) {
    throw new Error(`Maximum ${bookingRules.max_future_appointments} future appointments allowed`);
  }

  // Step 6: Get provider schedule templates
  const scheduleTemplates = await db.ProviderSchedule.findAll({
    where: {
      provider_id: providerId,
      location_id: locationId,
      effective_from: { [Op.lte]: endDate },
      [Op.or]: [
        { effective_to: null },
        { effective_to: { [Op.gte]: startDate } }
      ]
    }
  });

  // Step 7: Generate potential time slots
  const appointmentType = await db.AppointmentType.findByPk(appointmentTypeId);
  const slotDuration = appointmentType.default_duration_minutes;
  const slotInterval = bookingRules.slot_interval_minutes;

  let availableSlots = [];
  let currentDate = new Date(startDate);

  while (currentDate <= endDate) {
    // Skip if outside booking window
    if (currentDate < minBookingDate || currentDate > maxBookingDate) {
      currentDate.setDate(currentDate.getDate() + 1);
      continue;
    }

    // Check if date is a holiday
    const isHoliday = await checkArgentinaHoliday(currentDate, locationId);
    if (isHoliday) {
      currentDate.setDate(currentDate.getDate() + 1);
      continue;
    }

    // Get schedule for this day of week
    const dayOfWeek = currentDate.toLocaleDateString('en-US', { weekday: 'lowercase' });
    const daySchedule = scheduleTemplates.find(s => s.day_of_week === dayOfWeek);

    if (!daySchedule || !daySchedule.is_working_day) {
      currentDate.setDate(currentDate.getDate() + 1);
      continue;
    }

    // Generate slots for each work period
    for (const period of daySchedule.work_periods) {
      let slotTime = parseTime(period.start_time);
      const endTime = parseTime(period.end_time);

      while (slotTime < endTime - slotDuration) {
        const slotDateTime = new Date(currentDate);
        slotDateTime.setHours(Math.floor(slotTime / 60));
        slotDateTime.setMinutes(slotTime % 60);

        // Check if slot is available
        const isAvailable = await checkSlotAvailability({
          providerId,
          locationId,
          appointmentTypeId,
          slotDateTime,
          slotDuration,
          obraSocialId
        });

        if (isAvailable) {
          availableSlots.push({
            datetime: slotDateTime,
            provider_id: providerId,
            location_id: locationId,
            duration_minutes: slotDuration
          });
        }

        slotTime += slotInterval;
      }
    }

    // Limit slots per day if configured
    if (bookingRules.max_slots_per_day && availableSlots.length > bookingRules.max_slots_per_day) {
      availableSlots = availableSlots.slice(0, bookingRules.max_slots_per_day);
    }

    currentDate.setDate(currentDate.getDate() + 1);
  }

  return availableSlots;
}


/**
 * Check if a specific time slot is available
 */
async function checkSlotAvailability(params) {
  const {
    providerId,
    locationId,
    appointmentTypeId,
    slotDateTime,
    slotDuration,
    obraSocialId
  } = params;

  // 1. Check for existing appointments
  const existingAppointment = await db.Appointment.findOne({
    where: {
      provider_id: providerId,
      location_id: locationId,
      appointment_date: slotDateTime.toISOString().split('T')[0],
      appointment_time: slotDateTime.toTimeString().split(' ')[0].substring(0, 5),
      status: { [Op.notIn]: ['cancelled', 'no_show'] }
    }
  });

  if (existingAppointment) return false;

  // 2. Check for provider time off
  const timeOff = await db.ProviderTimeOff.findOne({
    where: {
      provider_id: providerId,
      start_datetime: { [Op.lte]: slotDateTime },
      end_datetime: { [Op.gte]: slotDateTime }
    }
  });

  if (timeOff) return false;

  // 3. Check for location closures
  const locationClosure = await db.LocationClosure.findOne({
    where: {
      location_id: locationId,
      closure_date: slotDateTime.toISOString().split('T')[0]
    }
  });

  if (locationClosure) return false;

  // 4. Check for obra social restrictions
  const providerObraSocial = await db.ProviderObraSocial.findOne({
    where: {
      provider_id: providerId,
      obra_social_id: obraSocialId,
      is_active: true
    }
  });

  if (!providerObraSocial) return false;

  // 5. Check obra social specific availability
  if (providerObraSocial.availability_schedule) {
    const dayOfWeek = slotDateTime.toLocaleDateString('en-US', { weekday: 'lowercase' });
    const obraSocialSchedule = providerObraSocial.availability_schedule[dayOfWeek];

    if (!obraSocialSchedule || !obraSocialSchedule.enabled) return false;

    const slotTime = slotDateTime.getHours() * 60 + slotDateTime.getMinutes();
    const isInObraSocialPeriod = obraSocialSchedule.periods.some(period => {
      const startMinutes = parseTime(period.start_time);
      const endMinutes = parseTime(period.end_time);
      return slotTime >= startMinutes && slotTime < endMinutes;
    });

    if (!isInObraSocialPeriod) return false;
  }

  return true;
}


/**
 * Check if date is an Argentina holiday
 */
async function checkArgentinaHoliday(date, locationId) {
  // National holidays
  const nationalHolidays = [
    { month: 1, day: 1, name: 'Año Nuevo' },
    { month: 3, day: 24, name: 'Día Nacional de la Memoria por la Verdad y la Justicia' },
    { month: 4, day: 2, name: 'Día del Veterano y de los Caídos en la Guerra de Malvinas' },
    { month: 5, day: 1, name: 'Día del Trabajador' },
    { month: 5, day: 25, name: 'Día de la Revolución de Mayo' },
    { month: 6, day: 20, name: 'Paso a la Inmortalidad del General Manuel Belgrano' },
    { month: 7, day: 9, name: 'Día de la Independencia' },
    { month: 8, day: 17, name: 'Paso a la Inmortalidad del General José de San Martín' },
    { month: 10, day: 12, name: 'Día del Respeto a la Diversidad Cultural' },
    { month: 11, day: 20, name: 'Día de la Soberanía Nacional' },
    { month: 12, day: 8, name: 'Inmaculada Concepción de María' },
    { month: 12, day: 25, name: 'Navidad' }
  ];

  const month = date.getMonth() + 1;
  const day = date.getDate();

  const isNationalHoliday = nationalHolidays.some(h => h.month === month && h.day === day);
  if (isNationalHoliday) return true;

  // Check custom holidays in database
  const customHoliday = await db.Holiday.findOne({
    where: {
      holiday_date: date.toISOString().split('T')[0],
      [Op.or]: [
        { applies_to_location_id: locationId },
        { is_national: true }
      ]
    }
  });

  return !!customHoliday;
}


/**
 * Reserve a time slot temporarily (15 minutes) while patient completes booking
 */
async function reserveTimeSlot(slotDateTime, providerId, sessionId) {
  const reservationExpiry = new Date(Date.now() + 15 * 60 * 1000); // 15 minutes

  await db.OnlineBookingSlotCache.update(
    {
      is_available: false,
      reserved_until: reservationExpiry,
      reserved_by_session: sessionId
    },
    {
      where: {
        slot_datetime: slotDateTime,
        provider_id: providerId,
        is_available: true
      }
    }
  );
}


/**
 * Release expired slot reservations (run every minute via cron)
 */
async function releaseExpiredReservations() {
  const now = new Date();

  await db.OnlineBookingSlotCache.update(
    {
      is_available: true,
      reserved_until: null,
      reserved_by_session: null
    },
    {
      where: {
        reserved_until: { [Op.lt]: now },
        is_available: false
      }
    }
  );
}
```

---

## 4. Booking Workflow

### 4.1 Complete Booking Process

```
┌─────────────────────────────────────────────────────────────────┐
│                    ONLINE BOOKING WORKFLOW                       │
└─────────────────────────────────────────────────────────────────┘

Step 1: Patient Authentication
├─ New patient: Register with email, password, DNI, obra social
├─ Existing patient: Login with email/password or DNI
└─ Verify email/phone if first-time booking

Step 2: Select Service
├─ Choose location (if multiple)
├─ Select specialty or appointment type
├─ Option: Select specific provider OR "First Available"
└─ Display provider photos, bios, patient ratings

Step 3: View Availability
├─ Show calendar view with available dates highlighted
├─ Click date to see available time slots
├─ Display: Provider name, time, duration, location
├─ Filter by: Morning/Afternoon/Evening, This week/Next week
└─ Real-time updates as slots are booked by others

Step 4: Validate Obra Social Coverage
├─ Confirm patient's obra social covers this provider/location
├─ Display: Copay amount, authorization required?
├─ If authorization needed: Warn patient to obtain before appointment
└─ Option to change obra social if multiple coverages

Step 5: Reserve Slot (15 minutes)
├─ Temporarily lock selected time slot
├─ Display countdown timer: "This slot is held for 15:00"
└─ If timer expires, release slot and return to availability

Step 6: Complete Pre-Visit Forms (if required)
├─ Display required forms: Reason for visit, symptoms, medications
├─ Validate required fields
└─ Save responses to pre_visit_form_responses table

Step 7: Payment (if required)
├─ If upfront payment required: Collect via Mercado Pago
├─ Display amount, payment methods
└─ Confirm payment successful

Step 8: Confirm Booking
├─ Display appointment summary
├─ Checkbox: "I agree to cancellation policy (24h notice required)"
├─ Click "Confirm Appointment"
└─ Create appointment record with status='confirmed'

Step 9: Confirmation & Notifications
├─ Display confirmation screen with appointment details
├─ Send confirmation email (Spanish template)
├─ Send confirmation SMS/WhatsApp
├─ Add to patient's calendar (Google/Outlook if connected)
└─ Display options: Add to Calendar, Cancel/Reschedule

Step 10: Automated Reminders
├─ Schedule reminder for 24 hours before
├─ Schedule reminder for 2 hours before
└─ Include confirmation link: C=confirm, R=reschedule, X=cancel
```

### 4.2 Cancellation/Rescheduling Workflow

```
Patient clicks "Cancel or Reschedule" in confirmation email or patient portal
│
├─ Check cancellation deadline (e.g., 24 hours before appointment)
│   ├─ Within deadline: Allow cancellation/rescheduling
│   └─ Past deadline: Show error "Cancellations must be made 24h in advance"
│
├─ Cancellation:
│   ├─ Update appointment status = 'cancelled'
│   ├─ Set cancelled_by = 'patient_online'
│   ├─ Release time slot back to availability
│   ├─ Send cancellation confirmation email
│   └─ Update analytics: cancellation_rate, revenue_impact
│
└─ Rescheduling:
    ├─ Keep original appointment record
    ├─ Show available slots (same provider or different)
    ├─ Reserve new slot
    ├─ Update appointment: date, time, rescheduled_from
    ├─ Send reschedule confirmation
    └─ Track reschedule reason (optional survey)
```

---

## 5. User Interface Specifications

### 5.1 Provider Selection Screen

```html
<!-- ARGENTINA PATIENT PORTAL - ONLINE BOOKING -->

<div class="booking-container">
  <h1>Reservar Turno Online</h1>

  <!-- Step indicator -->
  <div class="step-indicator">
    <div class="step active">1. Seleccionar Servicio</div>
    <div class="step">2. Elegir Fecha y Hora</div>
    <div class="step">3. Confirmar</div>
  </div>

  <!-- Location selection -->
  <div class="location-selector">
    <label>Sucursal:</label>
    <select id="location-select">
      <option value="1">Microcentro - Av. Corrientes 1234</option>
      <option value="2">Belgrano - Av. Cabildo 5678</option>
      <option value="3">San Isidro - Av. Libertador 9012</option>
    </select>
  </div>

  <!-- Specialty selection -->
  <div class="specialty-selector">
    <label>Especialidad:</label>
    <div class="specialty-grid">
      <div class="specialty-card" data-specialty="clinica_medica">
        <i class="icon-stethoscope"></i>
        <h3>Clínica Médica</h3>
        <p>Consultas generales y controles</p>
      </div>
      <div class="specialty-card" data-specialty="cardiologia">
        <i class="icon-heart"></i>
        <h3>Cardiología</h3>
        <p>Evaluación cardiovascular</p>
      </div>
      <div class="specialty-card" data-specialty="dermatologia">
        <i class="icon-skin"></i>
        <h3>Dermatología</h3>
        <p>Problemas de piel, cabello, uñas</p>
      </div>
      <!-- More specialties... -->
    </div>
  </div>

  <!-- Provider selection -->
  <div class="provider-selector">
    <h2>Seleccionar Profesional</h2>

    <div class="provider-list">
      <!-- Provider 1 -->
      <div class="provider-card" data-provider-id="15">
        <img src="/images/providers/dr-gonzalez.jpg" alt="Dr. González" class="provider-photo">
        <div class="provider-info">
          <h3>Dr. Juan González</h3>
          <p class="provider-specialty">Clínico Médico - MN 54321</p>
          <div class="provider-rating">
            ★★★★★ <span class="rating-count">(248 opiniones)</span>
          </div>
          <p class="provider-bio">
            Más de 15 años de experiencia en clínica médica. Especializado en
            atención integral del adulto y manejo de enfermedades crónicas.
          </p>
          <div class="provider-insurance">
            <strong>Obras Sociales:</strong> OSDE, Swiss Medical, Galeno, PAMI
          </div>
          <div class="provider-availability">
            <strong>Próximo turno disponible:</strong>
            <span class="next-available">Mañana, 16/11 a las 10:00</span>
          </div>
        </div>
        <button class="btn-select-provider">Ver Turnos Disponibles</button>
      </div>

      <!-- Provider 2 -->
      <div class="provider-card" data-provider-id="23">
        <img src="/images/providers/dra-martinez.jpg" alt="Dra. Martínez" class="provider-photo">
        <div class="provider-info">
          <h3>Dra. María Martínez</h3>
          <p class="provider-specialty">Clínica Médica - MN 67890</p>
          <div class="provider-rating">
            ★★★★☆ <span class="rating-count">(187 opiniones)</span>
          </div>
          <p class="provider-bio">
            Médica clínica con orientación en medicina preventiva y salud familiar.
          </p>
          <div class="provider-insurance">
            <strong>Obras Sociales:</strong> OSDE, Medicus, Sancor Salud
          </div>
          <div class="provider-availability">
            <strong>Próximo turno disponible:</strong>
            <span class="next-available">18/11 a las 14:30</span>
          </div>
        </div>
        <button class="btn-select-provider">Ver Turnos Disponibles</button>
      </div>

      <!-- "First Available" option -->
      <div class="provider-card first-available">
        <i class="icon-clock-fast"></i>
        <div class="provider-info">
          <h3>Primer Turno Disponible</h3>
          <p>No tengo preferencia de profesional, quiero el turno más cercano</p>
          <div class="provider-availability">
            <strong>Próximo turno:</strong>
            <span class="next-available">Mañana, 16/11 a las 10:00 - Dr. González</span>
          </div>
        </div>
        <button class="btn-select-provider">Reservar Este Turno</button>
      </div>
    </div>
  </div>
</div>
```

### 5.2 Calendar & Time Slot Selection

```html
<div class="availability-container">
  <h2>Turnos Disponibles - Dr. Juan González</h2>

  <!-- Obra Social validation -->
  <div class="obra-social-check">
    <i class="icon-check-circle success"></i>
    <span>Tu obra social <strong>OSDE 210</strong> está aceptada por este profesional</span>
    <span class="copay">Coseguro: $2,500</span>
  </div>

  <!-- Calendar view -->
  <div class="calendar-section">
    <div class="calendar-header">
      <button class="btn-prev-month">&larr; Octubre</button>
      <h3>Noviembre 2025</h3>
      <button class="btn-next-month">Diciembre &rarr;</button>
    </div>

    <div class="calendar-grid">
      <div class="calendar-day header">Dom</div>
      <div class="calendar-day header">Lun</div>
      <div class="calendar-day header">Mar</div>
      <div class="calendar-day header">Mié</div>
      <div class="calendar-day header">Jue</div>
      <div class="calendar-day header">Vie</div>
      <div class="calendar-day header">Sáb</div>

      <!-- November 2025 calendar -->
      <div class="calendar-day disabled">30</div>
      <div class="calendar-day disabled">31</div>
      <div class="calendar-day disabled">1</div>
      <div class="calendar-day disabled">2</div>
      <div class="calendar-day disabled">3</div>
      <div class="calendar-day disabled">4</div>
      <div class="calendar-day disabled">5</div>

      <div class="calendar-day disabled">6</div>
      <div class="calendar-day disabled">7</div>
      <div class="calendar-day disabled">8</div>
      <div class="calendar-day disabled">9</div>
      <div class="calendar-day disabled">10</div>
      <div class="calendar-day disabled">11</div>
      <div class="calendar-day disabled">12</div>

      <div class="calendar-day disabled">13</div>
      <div class="calendar-day disabled">14</div>
      <div class="calendar-day past">15</div>
      <div class="calendar-day today available" data-date="2025-11-16">
        16 <span class="availability-indicator">●●●</span>
      </div>
      <div class="calendar-day available" data-date="2025-11-17">
        17 <span class="availability-indicator">●●</span>
      </div>
      <div class="calendar-day available" data-date="2025-11-18">
        18 <span class="availability-indicator">●●●●</span>
      </div>
      <div class="calendar-day no-availability">19</div>

      <div class="calendar-day holiday" title="Día de la Soberanía Nacional">20</div>
      <div class="calendar-day available" data-date="2025-11-21">
        21 <span class="availability-indicator">●</span>
      </div>
      <!-- More days... -->
    </div>

    <div class="calendar-legend">
      <span class="legend-item">
        <span class="indicator available"></span> Turnos disponibles
      </span>
      <span class="legend-item">
        <span class="indicator no-availability"></span> Sin turnos
      </span>
      <span class="legend-item">
        <span class="indicator holiday"></span> Feriado
      </span>
    </div>
  </div>

  <!-- Time slot selection for selected date -->
  <div class="timeslot-section">
    <h3>Sábado, 16 de Noviembre de 2025</h3>

    <div class="time-filter">
      <button class="btn-filter active" data-filter="all">Todos</button>
      <button class="btn-filter" data-filter="morning">Mañana (8-12)</button>
      <button class="btn-filter" data-filter="afternoon">Tarde (12-18)</button>
      <button class="btn-filter" data-filter="evening">Noche (18-21)</button>
    </div>

    <div class="timeslot-grid">
      <div class="timeslot-card available" data-datetime="2025-11-16T10:00:00">
        <div class="timeslot-time">10:00</div>
        <div class="timeslot-duration">30 min</div>
        <button class="btn-select-slot">Seleccionar</button>
      </div>

      <div class="timeslot-card available" data-datetime="2025-11-16T10:30:00">
        <div class="timeslot-time">10:30</div>
        <div class="timeslot-duration">30 min</div>
        <button class="btn-select-slot">Seleccionar</button>
      </div>

      <div class="timeslot-card booked">
        <div class="timeslot-time">11:00</div>
        <div class="timeslot-status">Ocupado</div>
      </div>

      <div class="timeslot-card available" data-datetime="2025-11-16T11:30:00">
        <div class="timeslot-time">11:30</div>
        <div class="timeslot-duration">30 min</div>
        <button class="btn-select-slot">Seleccionar</button>
      </div>

      <!-- More time slots... -->
    </div>
  </div>
</div>
```

### 5.3 Pre-Visit Form Example

```html
<div class="pre-visit-form-container">
  <h2>Formulario Previo a la Consulta</h2>
  <p class="form-description">
    Por favor, complete la siguiente información para ayudar al Dr. González
    a preparar su consulta.
  </p>

  <form id="pre-visit-form">
    <!-- Reason for visit -->
    <div class="form-group required">
      <label for="visit-reason">Motivo de la consulta</label>
      <textarea
        id="visit-reason"
        name="reason"
        rows="4"
        maxlength="500"
        required
        placeholder="Ejemplo: Control de presión arterial, renovación de recetas"
      ></textarea>
      <span class="char-counter">0 / 500</span>
    </div>

    <!-- Current symptoms -->
    <div class="form-group">
      <label>Síntomas actuales (si corresponde)</label>
      <div class="checkbox-group">
        <label><input type="checkbox" name="symptoms[]" value="fiebre"> Fiebre</label>
        <label><input type="checkbox" name="symptoms[]" value="tos"> Tos</label>
        <label><input type="checkbox" name="symptoms[]" value="dolor_cabeza"> Dolor de cabeza</label>
        <label><input type="checkbox" name="symptoms[]" value="fatiga"> Fatiga/Cansancio</label>
        <label><input type="checkbox" name="symptoms[]" value="dolor_pecho"> Dolor de pecho</label>
        <label><input type="checkbox" name="symptoms[]" value="falta_aire"> Falta de aire</label>
        <label><input type="checkbox" name="symptoms[]" value="mareos"> Mareos</label>
        <label><input type="checkbox" name="symptoms[]" value="otro"> Otro</label>
      </div>
    </div>

    <!-- Current medications -->
    <div class="form-group">
      <label for="medications">Medicamentos que toma actualmente</label>
      <textarea
        id="medications"
        name="medications"
        rows="3"
        placeholder="Ejemplo: Enalapril 10mg (1 por día), Aspirina 100mg"
      ></textarea>
    </div>

    <!-- Allergies -->
    <div class="form-group">
      <label for="allergies">Alergias conocidas (medicamentos, alimentos, etc.)</label>
      <input
        type="text"
        id="allergies"
        name="allergies"
        placeholder="Ejemplo: Penicilina, mariscos"
      >
    </div>

    <!-- Recent changes -->
    <div class="form-group">
      <label>Desde su última visita, ¿ha tenido alguno de estos cambios?</label>
      <div class="checkbox-group">
        <label><input type="checkbox" name="changes[]" value="hospitalizacion"> Hospitalización</label>
        <label><input type="checkbox" name="changes[]" value="cirugia"> Cirugía</label>
        <label><input type="checkbox" name="changes[]" value="nuevos_medicamentos"> Nuevos medicamentos</label>
        <label><input type="checkbox" name="changes[]" value="cambio_obra_social"> Cambio de obra social</label>
        <label><input type="checkbox" name="changes[]" value="ninguno"> Ninguno</label>
      </div>
    </div>

    <!-- Submit buttons -->
    <div class="form-actions">
      <button type="button" class="btn-secondary" onclick="history.back()">Volver</button>
      <button type="submit" class="btn-primary">Continuar</button>
    </div>
  </form>
</div>
```

### 5.4 Booking Confirmation Screen

```html
<div class="confirmation-container">
  <!-- Success icon -->
  <div class="success-icon">
    <i class="icon-check-circle-large"></i>
  </div>

  <h1>¡Turno Confirmado!</h1>
  <p class="confirmation-message">
    Tu turno ha sido reservado exitosamente. Recibirás un email y SMS de confirmación.
  </p>

  <!-- Appointment details card -->
  <div class="appointment-card">
    <div class="card-header">
      <h2>Detalles del Turno</h2>
      <span class="confirmation-number">Confirmación #45782</span>
    </div>

    <div class="card-body">
      <div class="detail-row">
        <span class="label">Fecha y Hora:</span>
        <span class="value"><strong>Sábado, 16 de Noviembre de 2025 - 10:00 hs</strong></span>
      </div>

      <div class="detail-row">
        <span class="label">Profesional:</span>
        <span class="value">Dr. Juan González - Clínica Médica (MN 54321)</span>
      </div>

      <div class="detail-row">
        <span class="label">Sucursal:</span>
        <span class="value">Microcentro - Av. Corrientes 1234, CABA</span>
      </div>

      <div class="detail-row">
        <span class="label">Duración:</span>
        <span class="value">30 minutos</span>
      </div>

      <div class="detail-row">
        <span class="label">Obra Social:</span>
        <span class="value">OSDE 210</span>
      </div>

      <div class="detail-row">
        <span class="label">Coseguro:</span>
        <span class="value">$2,500 (a abonar en recepción)</span>
      </div>
    </div>
  </div>

  <!-- Important reminders -->
  <div class="reminders-section">
    <h3>Recordatorios Importantes</h3>
    <ul class="reminder-list">
      <li><strong>Llegada:</strong> Por favor, llegar 10 minutos antes de tu turno</li>
      <li><strong>Documentación:</strong> Traer DNI y credencial de obra social</li>
      <li><strong>Cancelación:</strong> Si necesitas cancelar, hazlo con al menos 24 horas de anticipación</li>
      <li><strong>Recordatorios:</strong> Recibirás recordatorios 24 horas y 2 horas antes de tu turno</li>
    </ul>
  </div>

  <!-- Action buttons -->
  <div class="confirmation-actions">
    <button class="btn-add-calendar">
      <i class="icon-calendar"></i>
      Agregar a mi Calendario
    </button>

    <button class="btn-print">
      <i class="icon-printer"></i>
      Imprimir Confirmación
    </button>

    <button class="btn-share">
      <i class="icon-share"></i>
      Compartir por WhatsApp
    </button>
  </div>

  <!-- Cancellation/Rescheduling options -->
  <div class="manage-appointment">
    <p>¿Necesitas cambiar o cancelar tu turno?</p>
    <button class="btn-secondary" onclick="rescheduleAppointment()">
      Reprogramar Turno
    </button>
    <button class="btn-danger" onclick="cancelAppointment()">
      Cancelar Turno
    </button>
  </div>

  <!-- Return to portal -->
  <div class="return-link">
    <a href="/patient-portal">Volver al Portal del Paciente</a>
  </div>
</div>
```

---

## 6. API Endpoints

### 6.1 Availability Endpoints

```javascript
/**
 * GET /api/online-booking/availability
 *
 * Get available time slots for online booking
 *
 * Query Parameters:
 * - location_id (required): Location ID
 * - provider_id (optional): Specific provider or null for any
 * - appointment_type_id (required): Appointment type ID
 * - start_date (required): Start date (YYYY-MM-DD)
 * - end_date (required): End date (YYYY-MM-DD)
 * - obra_social_id (required): Patient's obra social ID
 *
 * Response:
 * {
 *   "available_slots": [
 *     {
 *       "datetime": "2025-11-16T10:00:00-03:00",
 *       "provider": {
 *         "id": 15,
 *         "name": "Dr. Juan González",
 *         "specialty": "Clínica Médica",
 *         "photo_url": "/images/providers/dr-gonzalez.jpg",
 *         "rating": 4.9,
 *         "review_count": 248
 *       },
 *       "location": {
 *         "id": 1,
 *         "name": "Microcentro",
 *         "address": "Av. Corrientes 1234, CABA"
 *       },
 *       "duration_minutes": 30,
 *       "copay_amount": 2500
 *     },
 *     // More slots...
 *   ],
 *   "booking_rules": {
 *     "advance_booking_min_hours": 2,
 *     "advance_booking_max_days": 90,
 *     "cancellation_deadline_hours": 24
 *   }
 * }
 */
router.get('/availability', authenticatePatient, async (req, res) => {
  try {
    const {
      location_id,
      provider_id,
      appointment_type_id,
      start_date,
      end_date,
      obra_social_id
    } = req.query;

    const patientId = req.user.patient_id;

    const availableSlots = await calculateAvailableSlots({
      locationId: location_id,
      providerId: provider_id,
      appointmentTypeId: appointment_type_id,
      startDate: new Date(start_date),
      endDate: new Date(end_date),
      patientId,
      obraSocialId: obra_social_id
    });

    res.json({
      available_slots: availableSlots,
      booking_rules: bookingRules
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});


/**
 * POST /api/online-booking/reserve-slot
 *
 * Temporarily reserve a time slot (15 minutes)
 *
 * Request Body:
 * {
 *   "slot_datetime": "2025-11-16T10:00:00-03:00",
 *   "provider_id": 15,
 *   "appointment_type_id": 5
 * }
 *
 * Response:
 * {
 *   "reserved": true,
 *   "reservation_expires_at": "2025-11-16T09:15:00-03:00",
 *   "session_id": "abc123def456"
 * }
 */
router.post('/reserve-slot', authenticatePatient, async (req, res) => {
  try {
    const { slot_datetime, provider_id, appointment_type_id } = req.body;
    const sessionId = req.session.id;

    await reserveTimeSlot(new Date(slot_datetime), provider_id, sessionId);

    res.json({
      reserved: true,
      reservation_expires_at: new Date(Date.now() + 15 * 60 * 1000),
      session_id: sessionId
    });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});


/**
 * POST /api/online-booking/create-appointment
 *
 * Complete the booking and create the appointment
 *
 * Request Body:
 * {
 *   "slot_datetime": "2025-11-16T10:00:00-03:00",
 *   "provider_id": 15,
 *   "location_id": 1,
 *   "appointment_type_id": 5,
 *   "obra_social_id": 8,
 *   "reason_for_visit": "Control de presión arterial",
 *   "pre_visit_form_responses": { ... },
 *   "payment_token": "mp_token_abc123" (if upfront payment required)
 * }
 *
 * Response:
 * {
 *   "appointment_id": 45782,
 *   "confirmation_number": "45782",
 *   "status": "confirmed",
 *   "appointment": { ... full appointment details ... }
 * }
 */
router.post('/create-appointment', authenticatePatient, async (req, res) => {
  try {
    const {
      slot_datetime,
      provider_id,
      location_id,
      appointment_type_id,
      obra_social_id,
      reason_for_visit,
      pre_visit_form_responses,
      payment_token
    } = req.body;

    const patientId = req.user.patient_id;
    const sessionId = req.session.id;

    // Start transaction
    const transaction = await db.sequelize.transaction();

    try {
      // 1. Verify slot is still available and reserved by this session
      const slot = await db.OnlineBookingSlotCache.findOne({
        where: {
          slot_datetime: new Date(slot_datetime),
          provider_id,
          reserved_by_session: sessionId,
          reserved_until: { [Op.gt]: new Date() }
        },
        transaction
      });

      if (!slot) {
        throw new Error('Slot is no longer available or reservation expired');
      }

      // 2. Process payment if required
      let paymentId = null;
      if (payment_token) {
        const payment = await processMercadoPagoPayment({
          token: payment_token,
          amount: bookingRules.upfront_payment_amount,
          patient_id: patientId,
          description: `Turno online - ${appointmentType.name}`
        });
        paymentId = payment.id;
      }

      // 3. Create appointment
      const appointment = await db.Appointment.create({
        patient_id: patientId,
        provider_id,
        location_id,
        appointment_type_id,
        appointment_date: slot_datetime.split('T')[0],
        appointment_time: slot_datetime.split('T')[1].substring(0, 5),
        duration_minutes: appointmentType.default_duration_minutes,
        status: bookingRules.require_immediate_confirmation ? 'confirmed' : 'pending',
        obra_social_id,
        reason: reason_for_visit,
        booked_via: 'online_portal',
        payment_id: paymentId
      }, { transaction });

      // 4. Save pre-visit form responses
      if (pre_visit_form_responses) {
        await db.PreVisitFormResponse.create({
          form_id: bookingRules.required_form_ids[0],
          appointment_id: appointment.id,
          patient_id: patientId,
          responses: pre_visit_form_responses,
          completed_at: new Date(),
          ip_address: req.ip
        }, { transaction });
      }

      // 5. Mark slot as booked
      await slot.update({
        is_available: false,
        reserved_until: null,
        reserved_by_session: null
      }, { transaction });

      // 6. Log successful booking attempt
      await db.OnlineBookingAttempt.create({
        patient_id: patientId,
        session_id: sessionId,
        ip_address: req.ip,
        user_agent: req.get('User-Agent'),
        location_id,
        provider_id,
        appointment_type_id,
        requested_date: appointment.appointment_date,
        requested_time: appointment.appointment_time,
        status: 'success',
        appointment_id: appointment.id,
        started_at: req.session.booking_started_at,
        completed_at: new Date(),
        duration_seconds: Math.floor((Date.now() - req.session.booking_started_at) / 1000)
      }, { transaction });

      await transaction.commit();

      // 7. Send confirmations (async, outside transaction)
      sendAppointmentConfirmations(appointment);

      res.json({
        appointment_id: appointment.id,
        confirmation_number: String(appointment.id),
        status: appointment.status,
        appointment
      });

    } catch (error) {
      await transaction.rollback();
      throw error;
    }

  } catch (error) {
    // Log failed attempt
    await db.OnlineBookingAttempt.create({
      patient_id: req.user.patient_id,
      session_id: req.session.id,
      ip_address: req.ip,
      user_agent: req.get('User-Agent'),
      status: 'failed',
      failure_reason: error.message,
      started_at: req.session.booking_started_at,
      completed_at: new Date()
    });

    res.status(400).json({ error: error.message });
  }
});


/**
 * POST /api/online-booking/cancel-appointment
 *
 * Cancel an online booking
 *
 * Request Body:
 * {
 *   "appointment_id": 45782,
 *   "cancellation_reason": "Ya no necesito la consulta" (optional)
 * }
 *
 * Response:
 * {
 *   "cancelled": true,
 *   "refund_issued": true,
 *   "refund_amount": 2500
 * }
 */
router.post('/cancel-appointment', authenticatePatient, async (req, res) => {
  try {
    const { appointment_id, cancellation_reason } = req.body;
    const patientId = req.user.patient_id;

    const appointment = await db.Appointment.findOne({
      where: {
        id: appointment_id,
        patient_id: patientId
      }
    });

    if (!appointment) {
      throw new Error('Appointment not found');
    }

    // Check cancellation deadline
    const appointmentDateTime = new Date(`${appointment.appointment_date}T${appointment.appointment_time}`);
    const hoursUntilAppointment = (appointmentDateTime - new Date()) / (1000 * 60 * 60);

    if (hoursUntilAppointment < bookingRules.cancellation_deadline_hours) {
      throw new Error(`Cancellations must be made at least ${bookingRules.cancellation_deadline_hours} hours in advance`);
    }

    // Cancel appointment
    await appointment.update({
      status: 'cancelled',
      cancelled_by: 'patient_online',
      cancellation_reason,
      cancelled_at: new Date()
    });

    // Release slot back to availability
    await db.OnlineBookingSlotCache.update(
      { is_available: true },
      {
        where: {
          provider_id: appointment.provider_id,
          slot_datetime: appointmentDateTime
        }
      }
    );

    // Process refund if payment was made
    let refundAmount = 0;
    if (appointment.payment_id) {
      const refund = await processMercadoPagoRefund(appointment.payment_id);
      refundAmount = refund.amount;
    }

    // Send cancellation confirmation
    sendCancellationConfirmation(appointment);

    res.json({
      cancelled: true,
      refund_issued: refundAmount > 0,
      refund_amount: refundAmount
    });

  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});
```

---

## 7. Google Calendar Integration

### 7.1 OAuth Setup & Sync

```javascript
/**
 * Google Calendar integration for automatic appointment syncing
 */

const { google } = require('googleapis');

/**
 * GET /api/online-booking/connect-google-calendar
 *
 * Initiate Google Calendar OAuth flow
 */
router.get('/connect-google-calendar', authenticatePatient, async (req, res) => {
  const oauth2Client = new google.auth.OAuth2(
    process.env.GOOGLE_CLIENT_ID,
    process.env.GOOGLE_CLIENT_SECRET,
    process.env.GOOGLE_REDIRECT_URI
  );

  const authUrl = oauth2Client.generateAuthUrl({
    access_type: 'offline',
    scope: ['https://www.googleapis.com/auth/calendar'],
    state: JSON.stringify({ patient_id: req.user.patient_id })
  });

  res.redirect(authUrl);
});


/**
 * GET /api/online-booking/google-calendar-callback
 *
 * Handle Google Calendar OAuth callback
 */
router.get('/google-calendar-callback', async (req, res) => {
  const { code, state } = req.query;
  const { patient_id } = JSON.parse(state);

  const oauth2Client = new google.auth.OAuth2(
    process.env.GOOGLE_CLIENT_ID,
    process.env.GOOGLE_CLIENT_SECRET,
    process.env.GOOGLE_REDIRECT_URI
  );

  const { tokens } = await oauth2Client.getToken(code);

  // Store encrypted token
  await db.PatientPortalPreferences.update(
    {
      google_calendar_enabled: true,
      google_calendar_token: encryptToken(tokens)
    },
    {
      where: { patient_id }
    }
  );

  res.redirect('/patient-portal?google_calendar_connected=true');
});


/**
 * Add appointment to Google Calendar
 */
async function addToGoogleCalendar(appointment) {
  const patient = await db.Patient.findByPk(appointment.patient_id);
  const preferences = await db.PatientPortalPreferences.findOne({
    where: { patient_id: appointment.patient_id }
  });

  if (!preferences || !preferences.google_calendar_enabled) {
    return;
  }

  const oauth2Client = new google.auth.OAuth2();
  oauth2Client.setCredentials(decryptToken(preferences.google_calendar_token));

  const calendar = google.calendar({ version: 'v3', auth: oauth2Client });

  const provider = await db.User.findByPk(appointment.provider_id);
  const location = await db.Location.findByPk(appointment.location_id);

  const startDateTime = new Date(`${appointment.appointment_date}T${appointment.appointment_time}`);
  const endDateTime = new Date(startDateTime.getTime() + appointment.duration_minutes * 60 * 1000);

  const event = {
    summary: `Consulta Médica - ${provider.first_name} ${provider.last_name}`,
    description: `Turno con ${provider.first_name} ${provider.last_name} (${provider.specialty})

Motivo: ${appointment.reason}
Confirmación: #${appointment.id}

Para cancelar o reprogramar: ${process.env.APP_URL}/patient-portal/appointments/${appointment.id}`,
    location: `${location.name} - ${location.address}`,
    start: {
      dateTime: startDateTime.toISOString(),
      timeZone: 'America/Argentina/Buenos_Aires'
    },
    end: {
      dateTime: endDateTime.toISOString(),
      timeZone: 'America/Argentina/Buenos_Aires'
    },
    reminders: {
      useDefault: false,
      overrides: [
        { method: 'popup', minutes: 24 * 60 },  // 1 day before
        { method: 'popup', minutes: 120 }        // 2 hours before
      ]
    }
  };

  const calendarEvent = await calendar.events.insert({
    calendarId: 'primary',
    resource: event
  });

  // Store Google Calendar event ID for future updates/deletions
  await appointment.update({
    google_calendar_event_id: calendarEvent.data.id
  });
}
```

---

## 8. Analytics & Reporting

### 8.1 Online Booking Metrics Dashboard

```sql
-- Daily online booking performance report
SELECT
    date,
    total_attempts,
    successful_bookings,
    failed_bookings,
    abandoned_bookings,
    success_rate,
    average_duration_seconds,

    -- Revenue impact
    successful_bookings * AVG(copay_amount) AS estimated_revenue,

    -- Most popular times
    CASE
        WHEN hour BETWEEN 8 AND 11 THEN 'Morning'
        WHEN hour BETWEEN 12 AND 17 THEN 'Afternoon'
        ELSE 'Evening'
    END AS time_period

FROM online_booking_analytics
WHERE date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY date DESC, hour;


-- Top failure reasons (to identify improvement opportunities)
SELECT
    failure_reason,
    COUNT(*) AS occurrences,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM online_booking_attempts
WHERE status = 'failed'
    AND created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY failure_reason
ORDER BY occurrences DESC;

/* Expected output:
failure_reason                  | occurrences | percentage
--------------------------------|-------------|------------
slot_not_available              | 145         | 45.20%
obra_social_not_accepted        | 87          | 27.10%
outside_booking_window          | 42          | 13.08%
max_future_appointments_reached | 28          | 8.72%
outstanding_balance             | 19          | 5.92%
*/


-- Provider popularity in online bookings
SELECT
    p.first_name || ' ' || p.last_name AS provider_name,
    p.specialty,
    COUNT(DISTINCT oba.id) AS total_attempts,
    COUNT(DISTINCT CASE WHEN oba.status = 'success' THEN oba.id END) AS successful_bookings,
    ROUND(
        COUNT(DISTINCT CASE WHEN oba.status = 'success' THEN oba.id END) * 100.0 /
        NULLIF(COUNT(DISTINCT oba.id), 0),
        2
    ) AS success_rate,
    AVG(oba.duration_seconds) AS avg_booking_duration_seconds

FROM online_booking_attempts oba
JOIN users p ON p.id = oba.provider_id
WHERE oba.created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY p.id, provider_name, p.specialty
ORDER BY successful_bookings DESC
LIMIT 10;


-- Conversion funnel analysis
WITH funnel_stages AS (
    SELECT
        COUNT(DISTINCT session_id) AS started_booking,
        COUNT(DISTINCT CASE WHEN appointment_type_id IS NOT NULL THEN session_id END) AS selected_service,
        COUNT(DISTINCT CASE WHEN requested_date IS NOT NULL THEN session_id END) AS selected_slot,
        COUNT(DISTINCT CASE WHEN status = 'success' THEN session_id END) AS completed_booking
    FROM online_booking_attempts
    WHERE started_at >= CURRENT_DATE - INTERVAL '7 days'
)
SELECT
    started_booking,
    selected_service,
    ROUND(selected_service * 100.0 / started_booking, 2) AS pct_selected_service,
    selected_slot,
    ROUND(selected_slot * 100.0 / selected_service, 2) AS pct_selected_slot,
    completed_booking,
    ROUND(completed_booking * 100.0 / selected_slot, 2) AS pct_completed,
    ROUND(completed_booking * 100.0 / started_booking, 2) AS overall_conversion_rate
FROM funnel_stages;


-- Obra Social acceptance impact
SELECT
    os.name AS obra_social_name,
    COUNT(DISTINCT oba.id) AS booking_attempts,
    COUNT(DISTINCT CASE WHEN oba.failure_reason = 'obra_social_not_accepted' THEN oba.id END) AS rejected_bookings,
    COUNT(DISTINCT CASE WHEN oba.status = 'success' THEN oba.id END) AS successful_bookings,
    ROUND(
        COUNT(DISTINCT CASE WHEN oba.failure_reason = 'obra_social_not_accepted' THEN oba.id END) * 100.0 /
        NULLIF(COUNT(DISTINCT oba.id), 0),
        2
    ) AS rejection_rate

FROM online_booking_attempts oba
JOIN patients p ON p.id = oba.patient_id
JOIN obra_socials os ON os.id = p.primary_obra_social_id
WHERE oba.created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY os.name
HAVING COUNT(DISTINCT oba.id) >= 10
ORDER BY booking_attempts DESC;
```

---

## 9. Configuration & Settings

### 9.1 Admin Configuration Interface

```javascript
/**
 * Admin endpoints for configuring online booking rules
 */

/**
 * POST /api/admin/online-booking/configure
 *
 * Create or update online booking rules
 *
 * Request Body:
 * {
 *   "location_id": 1,
 *   "provider_id": 15,
 *   "appointment_type_id": 5,
 *   "enabled": true,
 *   "advance_booking_min_hours": 2,
 *   "advance_booking_max_days": 90,
 *   "show_available_slots_only": true,
 *   "slot_interval_minutes": 15,
 *   "max_slots_per_day": 20,
 *   "require_obra_social": true,
 *   "allowed_obra_socials": [1, 2, 5, 8],
 *   "require_existing_patient": false,
 *   "max_future_appointments": 3,
 *   "require_immediate_confirmation": true,
 *   "send_confirmation_email": true,
 *   "send_confirmation_sms": true,
 *   "allow_online_cancellation": true,
 *   "cancellation_deadline_hours": 24,
 *   "require_pre_visit_forms": true,
 *   "required_form_ids": [1, 3],
 *   "require_payment_upfront": false,
 *   "block_if_outstanding_balance": true,
 *   "min_days_between_same_type": 7
 * }
 */
router.post('/admin/online-booking/configure', authenticateAdmin, async (req, res) => {
  const rules = await db.OnlineBookingRule.upsert(req.body);
  res.json({ success: true, rules });
});


/**
 * GET /api/admin/online-booking/analytics
 *
 * Get comprehensive analytics dashboard
 */
router.get('/admin/online-booking/analytics', authenticateAdmin, async (req, res) => {
  const { start_date, end_date } = req.query;

  const analytics = await generateOnlineBookingAnalytics(start_date, end_date);

  res.json(analytics);
});
```

---

## 10. Argentina-Specific Compliance

### 10.1 Legal Requirements

```javascript
/**
 * ARGENTINA LEGAL COMPLIANCE FOR ONLINE BOOKING
 */

// 1. DNI Validation
function validateArgentinaDNI(dni) {
  // DNI format: 12.345.678 or 12345678 (7-8 digits)
  const dniRegex = /^[0-9]{7,8}$/;
  const cleanDNI = dni.replace(/\./g, '');

  if (!dniRegex.test(cleanDNI)) {
    throw new Error('DNI inválido. Debe contener 7 u 8 dígitos.');
  }

  return cleanDNI;
}

// 2. Obra Social Coverage Validation
async function validateObraSocialCoverage(patientId, providerId, appointmentTypeId) {
  const patient = await db.Patient.findByPk(patientId);
  const coverage = await db.ProviderObraSocial.findOne({
    where: {
      provider_id: providerId,
      obra_social_id: patient.primary_obra_social_id,
      is_active: true
    }
  });

  if (!coverage) {
    throw new Error(`El profesional no atiende ${patient.ObraSocial.name}`);
  }

  // Check if appointment type is covered
  if (coverage.excluded_appointment_types &&
      coverage.excluded_appointment_types.includes(appointmentTypeId)) {
    throw new Error('Este tipo de consulta no está cubierta por tu obra social');
  }

  return {
    covered: true,
    copay_amount: coverage.copay_amount,
    requires_authorization: coverage.requires_prior_authorization
  };
}

// 3. PAMI-Specific Rules
async function applyPAMIRules(patientId, appointmentDateTime) {
  const patient = await db.Patient.findByPk(patientId, {
    include: [{ model: db.ObraSocial, as: 'PrimaryObraSocial' }]
  });

  if (patient.PrimaryObraSocial.name.includes('PAMI')) {
    // PAMI restricts appointment frequency
    const recentAppointments = await db.Appointment.count({
      where: {
        patient_id: patientId,
        appointment_date: {
          [Op.gte]: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
        },
        status: { [Op.notIn]: ['cancelled', 'no_show'] }
      }
    });

    if (recentAppointments >= 2) {
      throw new Error('Los afiliados de PAMI pueden reservar máximo 2 turnos por mes');
    }
  }
}

// 4. Data Protection (Ley 25.326)
const ONLINE_BOOKING_PRIVACY_NOTICE = `
AVISO DE PRIVACIDAD - LEY 25.326

Al utilizar el sistema de turnos online, Ud. consiente que:

1. DATOS RECOPILADOS: Se almacenarán sus datos personales (nombre, DNI, obra social,
   teléfono, email) y datos de salud (motivo de consulta, síntomas, medicamentos).

2. FINALIDAD: Los datos se utilizan exclusivamente para gestionar su turno médico y
   brindar atención sanitaria.

3. SEGURIDAD: Sus datos se almacenan en servidores seguros con cifrado.

4. DERECHOS: Ud. tiene derecho a acceder, rectificar o suprimir sus datos contactando
   a privacidad@clinica.com.ar

5. CONFIDENCIALIDAD MÉDICA: El secreto médico se mantiene según Ley 17.132.

Al confirmar el turno, acepta estos términos.
`;
```

---

## 11. Testing & Quality Assurance

### 11.1 Test Scenarios

```javascript
/**
 * Test suite for online booking system
 */

describe('Online Booking System', () => {

  describe('Availability Calculation', () => {
    it('should return available slots within booking window', async () => {
      const slots = await calculateAvailableSlots({
        locationId: 1,
        providerId: 15,
        appointmentTypeId: 5,
        startDate: new Date(Date.now() + 2 * 60 * 60 * 1000), // 2 hours from now
        endDate: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
        patientId: 100,
        obraSocialId: 8
      });

      expect(slots.length).toBeGreaterThan(0);
      expect(slots[0].datetime).toBeInstanceOf(Date);
    });

    it('should exclude slots outside booking window', async () => {
      await expect(calculateAvailableSlots({
        startDate: new Date(Date.now() + 1 * 60 * 60 * 1000), // 1 hour (less than min 2h)
        endDate: new Date(Date.now() + 100 * 24 * 60 * 60 * 1000) // 100 days (more than max 90d)
      })).rejects.toThrow('booking window');
    });

    it('should exclude Argentina holidays', async () => {
      const christmasSlots = await calculateAvailableSlots({
        startDate: new Date('2025-12-25'),
        endDate: new Date('2025-12-25')
      });

      expect(christmasSlots.length).toBe(0);
    });

    it('should exclude slots when provider has time off', async () => {
      // Create time off record
      await db.ProviderTimeOff.create({
        provider_id: 15,
        start_datetime: '2025-11-20 00:00:00',
        end_datetime: '2025-11-22 23:59:59',
        reason: 'Vacation'
      });

      const slots = await calculateAvailableSlots({
        providerId: 15,
        startDate: new Date('2025-11-20'),
        endDate: new Date('2025-11-22')
      });

      expect(slots.length).toBe(0);
    });

    it('should exclude slots when obra social not accepted', async () => {
      await expect(calculateAvailableSlots({
        providerId: 15,
        obraSocialId: 999 // Non-existent obra social
      })).rejects.toThrow('not accepted');
    });
  });

  describe('Slot Reservation', () => {
    it('should reserve slot for 15 minutes', async () => {
      const slotDateTime = new Date('2025-11-16T10:00:00');
      const sessionId = 'test-session-123';

      await reserveTimeSlot(slotDateTime, 15, sessionId);

      const slot = await db.OnlineBookingSlotCache.findOne({
        where: { slot_datetime: slotDateTime }
      });

      expect(slot.is_available).toBe(false);
      expect(slot.reserved_by_session).toBe(sessionId);
      expect(slot.reserved_until).toBeInstanceOf(Date);
    });

    it('should release expired reservations', async () => {
      // Create expired reservation
      await db.OnlineBookingSlotCache.create({
        slot_datetime: new Date('2025-11-16T11:00:00'),
        is_available: false,
        reserved_until: new Date(Date.now() - 1000), // Expired 1 second ago
        reserved_by_session: 'expired-session'
      });

      await releaseExpiredReservations();

      const slot = await db.OnlineBookingSlotCache.findOne({
        where: { slot_datetime: new Date('2025-11-16T11:00:00') }
      });

      expect(slot.is_available).toBe(true);
      expect(slot.reserved_until).toBeNull();
    });
  });

  describe('Appointment Creation', () => {
    it('should create appointment with all required fields', async () => {
      const appointmentData = {
        slot_datetime: '2025-11-16T10:00:00',
        provider_id: 15,
        location_id: 1,
        appointment_type_id: 5,
        obra_social_id: 8,
        reason_for_visit: 'Control de presión arterial',
        pre_visit_form_responses: {
          reason: 'Control de presión arterial',
          symptoms: [],
          medications: 'Enalapril 10mg'
        }
      };

      const response = await request(app)
        .post('/api/online-booking/create-appointment')
        .send(appointmentData)
        .expect(200);

      expect(response.body.appointment_id).toBeDefined();
      expect(response.body.status).toBe('confirmed');
    });

    it('should prevent duplicate bookings (max future appointments)', async () => {
      // Create 3 future appointments
      for (let i = 0; i < 3; i++) {
        await db.Appointment.create({
          patient_id: 100,
          appointment_date: new Date(Date.now() + (i + 1) * 24 * 60 * 60 * 1000),
          status: 'confirmed'
        });
      }

      await expect(
        createAppointment({ patient_id: 100 })
      ).rejects.toThrow('Maximum 3 future appointments');
    });

    it('should block booking if outstanding balance exists', async () => {
      await db.Invoice.create({
        patient_id: 100,
        total_amount: 5000,
        amount_paid: 0,
        status: 'pending'
      });

      await expect(
        createAppointment({ patient_id: 100 })
      ).rejects.toThrow('outstanding balance');
    });
  });

  describe('Cancellation', () => {
    it('should allow cancellation within deadline', async () => {
      const appointment = await db.Appointment.create({
        patient_id: 100,
        appointment_date: new Date(Date.now() + 48 * 60 * 60 * 1000), // 48 hours
        status: 'confirmed'
      });

      const response = await request(app)
        .post('/api/online-booking/cancel-appointment')
        .send({ appointment_id: appointment.id })
        .expect(200);

      expect(response.body.cancelled).toBe(true);
    });

    it('should prevent cancellation past deadline', async () => {
      const appointment = await db.Appointment.create({
        patient_id: 100,
        appointment_date: new Date(Date.now() + 12 * 60 * 60 * 1000), // 12 hours
        status: 'confirmed'
      });

      await expect(
        cancelAppointment(appointment.id)
      ).rejects.toThrow('24 hours in advance');
    });
  });

  describe('Argentina-Specific Rules', () => {
    it('should validate DNI format', () => {
      expect(() => validateArgentinaDNI('12.345.678')).not.toThrow();
      expect(() => validateArgentinaDNI('12345678')).not.toThrow();
      expect(() => validateArgentinaDNI('123')).toThrow('DNI inválido');
      expect(() => validateArgentinaDNI('abc')).toThrow('DNI inválido');
    });

    it('should enforce PAMI appointment limits', async () => {
      const pamiPatient = await db.Patient.create({
        first_name: 'Test',
        last_name: 'PAMI',
        primary_obra_social_id: PAMI_OBRA_SOCIAL_ID
      });

      // Create 2 appointments this month
      for (let i = 0; i < 2; i++) {
        await db.Appointment.create({
          patient_id: pamiPatient.id,
          appointment_date: new Date(Date.now() + i * 24 * 60 * 60 * 1000),
          status: 'confirmed'
        });
      }

      await expect(
        applyPAMIRules(pamiPatient.id)
      ).rejects.toThrow('máximo 2 turnos por mes');
    });
  });
});
```

---

## 12. Performance Optimization

### 12.1 Caching Strategy

```javascript
/**
 * Slot cache refresh strategy
 *
 * - Run every 5 minutes via cron job
 * - Refresh next 30 days of availability
 * - Only for providers with online booking enabled
 */
async function refreshSlotCache() {
  const enabledRules = await db.OnlineBookingRule.findAll({
    where: { enabled: true }
  });

  for (const rule of enabledRules) {
    const startDate = new Date();
    const endDate = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);

    const slots = await calculateAvailableSlots({
      locationId: rule.location_id,
      providerId: rule.provider_id,
      appointmentTypeId: rule.appointment_type_id,
      startDate,
      endDate
    });

    // Upsert to cache
    for (const slot of slots) {
      await db.OnlineBookingSlotCache.upsert({
        location_id: rule.location_id,
        provider_id: rule.provider_id,
        appointment_type_id: rule.appointment_type_id,
        slot_date: slot.datetime.toISOString().split('T')[0],
        slot_time: slot.datetime.toTimeString().split(' ')[0].substring(0, 5),
        slot_datetime: slot.datetime,
        is_available: true,
        duration_minutes: slot.duration_minutes,
        cache_expires_at: new Date(Date.now() + 60 * 60 * 1000), // 1 hour
        last_refreshed_at: new Date()
      });
    }
  }
}

// Schedule: Every 5 minutes
// crontab: */5 * * * * /usr/bin/node refresh-slot-cache.js
```

---

## 13. Summary

The Online Patient Self-Booking System provides:

✅ **24/7 Self-Service**: Patients book appointments anytime without staff intervention
✅ **Real-Time Availability**: Dynamic calculation considering all constraints
✅ **Argentina Compliance**: DNI validation, Obra Social coverage, PAMI rules, holidays
✅ **Smart Scheduling**: Prevents double-booking, validates insurance, enforces limits
✅ **Multi-Channel Confirmations**: Email, SMS, WhatsApp notifications in Spanish
✅ **Calendar Integration**: Google Calendar and Outlook sync
✅ **Pre-Visit Forms**: Collect patient information before appointment
✅ **Flexible Cancellation**: Online cancellation/rescheduling within policy
✅ **Payment Integration**: Optional upfront payment via Mercado Pago
✅ **Analytics Dashboard**: Track conversion rates, popular times, failure reasons
✅ **Performance Optimized**: Slot caching, reservation system, 15-minute holds

**Next Steps:**
1. Implement frontend patient portal with booking interface
2. Set up cron jobs for slot cache refresh and reservation cleanup
3. Configure online booking rules per provider/location
4. Create pre-visit forms for common appointment types
5. Test Argentina holiday calendar accuracy
6. Train staff on monitoring online booking analytics
