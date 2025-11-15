# Clinical Management System - Advanced Features Specification

## Overview

This document details advanced features including flexible scheduling with insurance-specific availability, comprehensive reporting system, and granular role management.

---

## 1. ADVANCED SCHEDULING SYSTEM

### 1.1 Multi-Dimensional Provider Availability

#### 1.1.1 Provider Schedule Structure

```
Provider Availability Dimensions:
1. Day of week
2. Time slots
3. Insurance/Obra Social
4. Location
5. Appointment type
6. Overbooking rules

Example:
Dr. Martinez works:
- Monday 9-13: OSDE patients only, max 2 overbook
- Monday 15-19: All insurances, max 1 overbook
- Tuesday 9-13: PAMI only, no overbooking
- Wednesday 9-17: Particular patients only, max 3 overbook
- Thursday: Not available
- Friday 9-12: Swiss Medical only, Location: Branch Office
```

#### 1.1.2 Database Schema Enhancement

```sql
-- Enhanced provider schedules table
CREATE TABLE provider_availability_rules (
    id                      BIGSERIAL PRIMARY KEY,
    provider_id             BIGINT NOT NULL REFERENCES users(id),

    -- Temporal rules
    day_of_week             INTEGER, -- 0-6, NULL = all days
    start_time              TIME NOT NULL,
    end_time                TIME NOT NULL,
    effective_from          DATE,
    effective_until         DATE,

    -- Location
    location_id             BIGINT REFERENCES locations(id),

    -- Insurance restrictions
    insurance_restriction   VARCHAR(50), -- 'allow_list', 'block_list', 'all'
    allowed_insurances      BIGINT[], -- Array of insurance IDs (if allow_list)
    blocked_insurances      BIGINT[], -- Array of insurance IDs (if block_list)

    -- Appointment type restrictions
    allowed_appointment_types BIGINT[], -- Array of appointment type IDs

    -- Capacity rules
    max_appointments        INTEGER, -- Max appointments in this slot
    slot_duration           INTEGER, -- Slot duration in minutes
    buffer_time             INTEGER, -- Minutes between appointments

    -- Overbooking rules
    allow_overbooking       BOOLEAN DEFAULT false,
    max_overbook_per_slot   INTEGER DEFAULT 0, -- Max overbook per time slot
    max_overbook_per_day    INTEGER DEFAULT 0, -- Max overbook for entire day
    overbook_only_for       BIGINT[], -- Insurance IDs that can overbook

    -- Priority
    priority                INTEGER DEFAULT 0, -- Higher priority rules override lower

    -- Status
    is_active               BOOLEAN DEFAULT true,
    notes                   TEXT,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_provider_avail_provider (provider_id),
    INDEX idx_provider_avail_day (day_of_week),
    INDEX idx_provider_avail_dates (effective_from, effective_until)
);

-- Overbooking rules per insurance
CREATE TABLE provider_overbook_rules (
    id                      BIGSERIAL PRIMARY KEY,
    provider_id             BIGINT NOT NULL REFERENCES users(id),
    insurance_id            BIGINT REFERENCES insurances(id),

    -- Rules can be day-specific
    day_of_week             INTEGER, -- NULL = applies to all days

    -- Overbooking limits
    max_overbook_per_hour   INTEGER DEFAULT 0,
    max_overbook_per_day    INTEGER DEFAULT 0,

    -- Time restrictions for overbooking
    overbook_allowed_from   TIME, -- e.g., only allow overbook after 17:00
    overbook_allowed_until  TIME,

    -- Emergency overbooking
    allow_emergency_overbook BOOLEAN DEFAULT false,

    effective_from          DATE,
    effective_until         DATE,
    is_active               BOOLEAN DEFAULT true,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_overbook_provider (provider_id),
    INDEX idx_overbook_insurance (insurance_id)
);

-- Locations table
CREATE TABLE locations (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    address                 TEXT,
    phone                   VARCHAR(50),
    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Provider working at multiple locations
CREATE TABLE provider_locations (
    id                      BIGSERIAL PRIMARY KEY,
    provider_id             BIGINT NOT NULL REFERENCES users(id),
    location_id             BIGINT NOT NULL REFERENCES locations(id),
    is_primary              BOOLEAN DEFAULT false,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(provider_id, location_id)
);
```

#### 1.1.3 Availability Calculation Algorithm

```javascript
/**
 * Check if provider is available for appointment
 * @param {number} providerId
 * @param {Date} appointmentDateTime
 * @param {number} duration - in minutes
 * @param {number} patientInsuranceId
 * @param {number} appointmentTypeId
 * @returns {Object} { available: boolean, reason: string, canOverbook: boolean }
 */
async function checkProviderAvailability(
  providerId,
  appointmentDateTime,
  duration,
  patientInsuranceId,
  appointmentTypeId
) {
  const dayOfWeek = appointmentDateTime.getDay();
  const timeSlot = appointmentDateTime.toTimeString().substring(0, 5); // HH:MM
  const date = appointmentDateTime.toISOString().split('T')[0];

  // 1. Get all availability rules for this provider
  const availabilityRules = await ProviderAvailabilityRule.findAll({
    where: {
      provider_id: providerId,
      is_active: true,
      [Op.or]: [
        { day_of_week: dayOfWeek },
        { day_of_week: null } // Rules that apply to all days
      ],
      [Op.or]: [
        { effective_from: null, effective_until: null },
        {
          effective_from: { [Op.lte]: date },
          effective_until: { [Op.gte]: date }
        }
      ]
    },
    order: [['priority', 'DESC']] // Higher priority first
  });

  if (availabilityRules.length === 0) {
    return {
      available: false,
      reason: 'Provider does not work on this day/time',
      canOverbook: false
    };
  }

  // 2. Find matching rule for this time slot
  let matchingRule = null;
  for (const rule of availabilityRules) {
    if (timeSlot >= rule.start_time && timeSlot < rule.end_time) {
      matchingRule = rule;
      break;
    }
  }

  if (!matchingRule) {
    return {
      available: false,
      reason: 'Provider not available at this time',
      canOverbook: false
    };
  }

  // 3. Check insurance restrictions
  if (matchingRule.insurance_restriction === 'allow_list') {
    if (!matchingRule.allowed_insurances.includes(patientInsuranceId)) {
      return {
        available: false,
        reason: 'Provider does not accept this insurance at this time',
        canOverbook: false
      };
    }
  } else if (matchingRule.insurance_restriction === 'block_list') {
    if (matchingRule.blocked_insurances.includes(patientInsuranceId)) {
      return {
        available: false,
        reason: 'Provider does not accept this insurance at this time',
        canOverbook: false
      };
    }
  }

  // 4. Check appointment type restrictions
  if (matchingRule.allowed_appointment_types &&
      matchingRule.allowed_appointment_types.length > 0) {
    if (!matchingRule.allowed_appointment_types.includes(appointmentTypeId)) {
      return {
        available: false,
        reason: 'This appointment type not available at this time',
        canOverbook: false
      };
    }
  }

  // 5. Check existing appointments in this slot
  const slotStart = new Date(appointmentDateTime);
  const slotEnd = new Date(appointmentDateTime.getTime() + duration * 60000);

  const existingAppointments = await Appointment.count({
    where: {
      provider_id: providerId,
      scheduled_at: {
        [Op.between]: [slotStart, slotEnd]
      },
      status: {
        [Op.notIn]: ['cancelled', 'no_show']
      }
    }
  });

  // 6. Check capacity
  if (existingAppointments < matchingRule.max_appointments) {
    return {
      available: true,
      reason: 'Slot available',
      canOverbook: false
    };
  }

  // 7. Check overbooking rules
  if (!matchingRule.allow_overbooking) {
    return {
      available: false,
      reason: 'Time slot is full and overbooking not allowed',
      canOverbook: false
    };
  }

  // 8. Get overbooking rules for this insurance
  const overbookRule = await ProviderOverbookRule.findOne({
    where: {
      provider_id: providerId,
      insurance_id: patientInsuranceId,
      is_active: true,
      [Op.or]: [
        { day_of_week: dayOfWeek },
        { day_of_week: null }
      ]
    }
  });

  if (!overbookRule) {
    return {
      available: false,
      reason: 'Overbooking not allowed for this insurance',
      canOverbook: false
    };
  }

  // 9. Check overbook limits for this slot
  const overbookedInSlot = existingAppointments - matchingRule.max_appointments;
  if (overbookedInSlot >= matchingRule.max_overbook_per_slot) {
    return {
      available: false,
      reason: 'Maximum overbooking reached for this time slot',
      canOverbook: false
    };
  }

  // 10. Check overbook limits for the day
  const dayStart = new Date(appointmentDateTime);
  dayStart.setHours(0, 0, 0, 0);
  const dayEnd = new Date(appointmentDateTime);
  dayEnd.setHours(23, 59, 59, 999);

  const appointmentsToday = await Appointment.count({
    where: {
      provider_id: providerId,
      scheduled_at: {
        [Op.between]: [dayStart, dayEnd]
      },
      status: {
        [Op.notIn]: ['cancelled', 'no_show']
      }
    }
  });

  // Calculate total capacity for the day
  const totalCapacityToday = calculateDailyCapacity(matchingRule, dayOfWeek);
  const overbookedToday = appointmentsToday - totalCapacityToday;

  if (overbookedToday >= matchingRule.max_overbook_per_day) {
    return {
      available: false,
      reason: 'Maximum daily overbooking reached',
      canOverbook: false
    };
  }

  // 11. Check time restrictions for overbooking
  if (overbookRule.overbook_allowed_from && overbookRule.overbook_allowed_until) {
    if (timeSlot < overbookRule.overbook_allowed_from ||
        timeSlot > overbookRule.overbook_allowed_until) {
      return {
        available: false,
        reason: 'Overbooking only allowed during specific hours',
        canOverbook: false
      };
    }
  }

  // Overbooking is allowed!
  return {
    available: true,
    reason: 'Available as overbooked slot',
    canOverbook: true,
    isOverbooked: true
  };
}
```

### 1.2 Provider Schedule Configuration UI

#### 1.2.1 Schedule Builder Interface

```
┌──────────────────────────────────────────────────────────────────┐
│ Provider Schedule - Dr. Martinez                    [Save] [Cancel]│
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Base Schedule:                                                   │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Monday    [09:00] - [13:00]  Location: [Main Clinic ▼]    │ │
│ │           Insurance: [All ▼]  Max: [8] Overbook: [2]       │ │
│ │                                                            │ │
│ │           [15:00] - [19:00]  Location: [Main Clinic ▼]    │ │
│ │           Insurance: [OSDE only ▼]  Max: [8] Overbook: [1]│ │
│ │           [+ Add Time Slot]                                │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ Tuesday   [09:00] - [13:00]  Location: [Main Clinic ▼]    │ │
│ │           Insurance: [PAMI only ▼]  Max: [6] Overbook: [0]│ │
│ │           [+ Add Time Slot]                                │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ Wednesday [09:00] - [17:00]  Location: [Branch Office ▼]  │ │
│ │           Insurance: [All except PAMI ▼]  Max: [10]       │ │
│ │           [+ Add Time Slot]                                │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ Thursday  [Not Working]                    [+ Add Hours]   │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ Friday    [09:00] - [12:00]  Location: [Main Clinic ▼]    │ │
│ │           Insurance: [Swiss Medical ▼]  Max: [4]          │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Advanced Overbooking Rules by Insurance:                         │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Insurance       Max/Hour  Max/Day  Time Window  Emergency  │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ OSDE            2         4        Any           ✓         │ │
│ │ Swiss Medical   1         2        15:00-19:00   ✓         │ │
│ │ PAMI            0         0        N/A           ✗         │ │
│ │ Particular      3         6        Any           ✓         │ │
│ │ [+ Add Insurance Rule]                                     │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Special Availability:                                            │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ • Dec 1-31: Extended hours Mon-Fri 9-20 (Holiday season)   │ │
│ │ • Jan 15-Feb 15: Reduced hours (Vacation coverage)         │ │
│ │ [+ Add Special Period]                                     │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Blocked Times:                                                   │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ • Nov 20, 14:00-16:00 - Conference                         │ │
│ │ • Every Monday 13:00-15:00 - Lunch & Admin                 │ │
│ │ [+ Add Blocked Time]                                       │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

#### 1.2.2 Insurance-Specific Schedule Configuration

```sql
-- Template for complex schedule rule
INSERT INTO provider_availability_rules (
    provider_id,
    day_of_week,
    start_time,
    end_time,
    location_id,
    insurance_restriction,
    allowed_insurances,
    max_appointments,
    slot_duration,
    buffer_time,
    allow_overbooking,
    max_overbook_per_slot,
    max_overbook_per_day,
    priority
) VALUES (
    5, -- Dr. Martinez
    1, -- Monday
    '09:00',
    '13:00',
    1, -- Main clinic
    'allow_list',
    ARRAY[1, 2, 3], -- OSDE, Swiss Medical, Galeno
    8, -- Max 8 appointments in this 4-hour block
    30, -- 30-minute slots
    5, -- 5 minutes buffer
    true, -- Allow overbooking
    2, -- Max 2 overbook per slot
    4, -- Max 4 overbook per day
    10 -- Priority 10
);
```

### 1.3 Appointment Type Flexibility

#### 1.3.1 Enhanced Appointment Types

```sql
CREATE TABLE appointment_types (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(100) NOT NULL,
    description             TEXT,

    -- Duration and timing
    default_duration        INTEGER NOT NULL DEFAULT 30, -- minutes
    min_duration            INTEGER, -- Minimum allowed duration
    max_duration            INTEGER, -- Maximum allowed duration
    allow_duration_override BOOLEAN DEFAULT false,

    -- Scheduling rules
    buffer_before           INTEGER DEFAULT 0, -- Minutes before
    buffer_after            INTEGER DEFAULT 0, -- Minutes after
    advance_booking_min     INTEGER, -- Min days in advance
    advance_booking_max     INTEGER, -- Max days in advance

    -- Availability
    available_days          INTEGER[], -- Array: [1,2,3] for Mon,Tue,Wed
    available_hours_start   TIME, -- e.g., only bookable after 14:00
    available_hours_end     TIME,

    -- Insurance restrictions
    allowed_insurances      BIGINT[], -- NULL = all insurances
    requires_authorization  BOOLEAN DEFAULT false,

    -- Capacity
    max_per_day_per_provider INTEGER, -- e.g., max 2 "New Patient" per day

    -- Cost and billing
    default_price           DECIMAL(10,2),
    insurance_prices        JSONB, -- Different prices per insurance

    -- Display
    color                   VARCHAR(20), -- For calendar
    icon                    VARCHAR(50),
    display_order           INTEGER,

    -- Requirements
    requires_forms          BOOLEAN DEFAULT false, -- Pre-appointment forms
    required_forms          BIGINT[], -- Array of form IDs

    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Link appointment types to providers (some doctors don't offer all types)
CREATE TABLE provider_appointment_types (
    id                      BIGSERIAL PRIMARY KEY,
    provider_id             BIGINT NOT NULL REFERENCES users(id),
    appointment_type_id     BIGINT NOT NULL REFERENCES appointment_types(id),

    -- Provider-specific overrides
    duration_override       INTEGER, -- This doctor takes longer/shorter
    price_override          DECIMAL(10,2),
    max_per_day            INTEGER,

    is_active               BOOLEAN DEFAULT true,

    UNIQUE(provider_id, appointment_type_id)
);
```

### 1.4 Resource Booking (Rooms, Equipment)

```sql
CREATE TABLE resources (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    type                    VARCHAR(50) NOT NULL, -- 'room', 'equipment', 'bed'
    location_id             BIGINT REFERENCES locations(id),
    description             TEXT,
    capacity                INTEGER, -- e.g., room can hold 2 simultaneous appointments
    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Link appointments to resources
CREATE TABLE appointment_resources (
    id                      BIGSERIAL PRIMARY KEY,
    appointment_id          BIGINT NOT NULL REFERENCES appointments(id),
    resource_id             BIGINT NOT NULL REFERENCES resources(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(appointment_id, resource_id)
);

-- Resource availability rules
CREATE TABLE resource_availability (
    id                      BIGSERIAL PRIMARY KEY,
    resource_id             BIGINT NOT NULL REFERENCES resources(id),
    day_of_week             INTEGER,
    start_time              TIME NOT NULL,
    end_time                TIME NOT NULL,
    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 2. COMPREHENSIVE REPORTING SYSTEM

### 2.1 Report Categories

#### 2.1.1 Clinical Reports

**1. Patient Demographics Report**
```
Filters:
- Date range (registration date)
- Age range
- Gender
- Insurance type
- Active/Inactive
- Location

Columns:
- Patient count
- Age distribution (0-17, 18-30, 31-50, 51-70, 70+)
- Gender breakdown
- Insurance distribution
- Average age
- New patients vs returning

Grouping:
- By insurance
- By age group
- By month
- By location

Visualization:
- Pie chart: Gender distribution
- Bar chart: Age groups
- Line chart: New patients trend
```

**2. Diagnosis Frequency Report**
```
Filters:
- Date range
- Provider
- ICD-10 code range
- Patient age range
- Insurance type

Columns:
- ICD-10 code
- Diagnosis name
- Count
- Percentage of total
- Average patient age
- Gender breakdown
- Trend (vs previous period)

Top diagnoses shown by default

Visualization:
- Bar chart: Top 20 diagnoses
- Trend line: Selected diagnosis over time
```

**3. Provider Productivity Report**
```
Filters:
- Date range
- Provider(s)
- Location

Metrics per provider:
- Total encounters
- Encounters by type (new patient, follow-up)
- Average encounters per day
- Total work hours
- Utilization rate (appointments / available slots)
- Average encounter duration
- No-show rate
- Patient satisfaction (if collected)

Grouping:
- By provider
- By day/week/month
- By location

Visualization:
- Bar chart: Encounters per provider
- Line chart: Productivity trend
- Heatmap: Work hours distribution
```

**4. Appointment Analytics Report**
```
Metrics:
- Total appointments scheduled
- Appointments by status (completed, cancelled, no-show)
- Completion rate %
- Cancellation rate %
- No-show rate %
- Average wait time (arrived to in-progress)
- Average consultation time
- Appointments by day of week
- Appointments by hour of day
- Appointments by insurance

Filters:
- Date range
- Provider
- Appointment type
- Insurance
- Location

Visualization:
- Donut chart: Status breakdown
- Heatmap: Appointments by day/hour
- Bar chart: Appointments by insurance
```

**5. Prescription Analysis Report**
```
Metrics:
- Total prescriptions issued
- Unique medications prescribed
- Most prescribed medications
- Prescriptions by drug class
- Average medications per prescription
- Chronic vs acute prescriptions
- Controlled substances prescribed

Filters:
- Date range
- Provider
- Medication class
- Patient age range

Grouping:
- By medication
- By drug class
- By provider

Visualization:
- Bar chart: Top 20 medications
- Pie chart: Prescription types
```

**6. Lab Results Report**
```
Metrics:
- Total lab orders
- Total results received
- Average turnaround time
- Abnormal results rate
- Critical results count
- Most ordered tests
- Orders by test type

Filters:
- Date range
- Provider
- Test type
- Patient
- Status

Visualization:
- Bar chart: Test frequency
- Line chart: Abnormal rate trend
```

#### 2.1.2 Financial Reports

**1. Revenue Summary Report**
```
Metrics:
- Total revenue
- Revenue by payment method
- Revenue by insurance
- Revenue by service type
- Revenue by provider
- Revenue by location
- Average transaction value
- Number of transactions

Filters:
- Date range
- Provider
- Insurance
- Payment method
- Location

Grouping:
- By day/week/month/quarter/year
- By insurance
- By provider
- By service

Visualization:
- Line chart: Revenue trend
- Pie chart: Revenue by source
- Bar chart: Revenue by provider
- Stacked bar: Revenue by insurance over time
```

**2. Accounts Receivable Aging Report**
```
Columns:
- Patient name
- Invoice number
- Invoice date
- Total amount
- Amount paid
- Balance due
- Days outstanding
- Aging bucket (0-30, 31-60, 61-90, 90+)
- Insurance
- Last payment date

Filters:
- Aging bucket
- Insurance
- Minimum balance

Summary:
- Total AR
- AR by aging bucket
- Collection rate %

Visualization:
- Bar chart: AR by aging bucket
- Table with drill-down
```

**3. Insurance Claims Report**
```
Metrics:
- Total claims submitted
- Total claims approved
- Total claims denied
- Total claims pending
- Approval rate %
- Denial rate %
- Average claim amount
- Average approved amount
- Average denial reason

Filters:
- Date range
- Insurance
- Status
- Provider

Grouping:
- By insurance
- By status
- By month

Visualization:
- Funnel chart: Claim lifecycle
- Bar chart: Claims by insurance
- Line chart: Approval rate trend
```

**4. Payment Collection Report**
```
Metrics:
- Total payments collected
- Payments by method (cash, card, transfer)
- Average payment amount
- Collection rate (payments / invoices)
- Outstanding balance
- Payments by insurance

Filters:
- Date range
- Payment method
- Insurance
- Provider

Visualization:
- Pie chart: Payment methods
- Line chart: Collections trend
- Bar chart: Collections by day of week
```

**5. Service Profitability Report**
```
Metrics per service:
- Times performed
- Total revenue
- Average price
- Revenue by insurance
- Provider performing most
- Trend over time

Filters:
- Date range
- Service category
- Provider

Visualization:
- Bar chart: Revenue per service
- Table with trends
```

**6. AFIP Invoicing Report**
```
Metrics:
- Invoices by type (A, B, C)
- Total invoices issued
- Total amount invoiced
- CAE success rate
- Failed CAE requests
- Invoice cancellations (credit notes)
- Tax collected (IVA)

Filters:
- Date range
- Invoice type
- Point of sale

Export:
- AFIP-compliant format
- Excel export for accounting
```

#### 2.1.3 Operational Reports

**1. Wait Time Analysis**
```
Metrics:
- Average wait time (check-in to appointment start)
- Median wait time
- 90th percentile wait time
- Wait time by provider
- Wait time by appointment type
- Wait time by day of week
- Wait time by hour of day

Filters:
- Date range
- Provider
- Appointment type

Visualization:
- Box plot: Wait time distribution
- Heatmap: Wait times by day/hour
- Line chart: Wait time trend
```

**2. Capacity Utilization Report**
```
Metrics:
- Total available slots
- Total booked slots
- Utilization rate %
- Overbooking rate
- Cancellation rate
- No-show rate
- Slots by provider
- Peak times
- Underutilized times

Filters:
- Date range
- Provider
- Location

Visualization:
- Gauge: Overall utilization
- Heatmap: Utilization by day/hour
- Bar chart: Utilization by provider
```

**3. Inventory Turnover Report**
```
Metrics:
- Items low in stock
- Items expiring soon
- Items expired
- Inventory value
- Inventory by category
- Usage rate
- Reorder recommendations

Filters:
- Category
- Expiration date range
- Stock level

Visualization:
- Table with alerts
- Bar chart: Inventory value by category
```

**4. Staff Performance Report**
```
Metrics per staff member:
- Patients processed (receptionist)
- Encounters completed (doctor/nurse)
- Revenue generated
- Average service time
- Patient satisfaction scores
- Attendance/punctuality

Filters:
- Date range
- Department
- Role

Visualization:
- Leaderboard table
- Bar chart: Performance metrics
```

### 2.2 Custom Report Builder

#### 2.2.1 Report Builder Interface

```
┌──────────────────────────────────────────────────────────────────┐
│ Custom Report Builder                          [Save] [Run] [Export]│
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Report Name: [My Custom Patient Report]                         │
│                                                                  │
│ 1. SELECT DATA SOURCE                                            │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ ● Patients                                                 │ │
│ │ ○ Appointments                                             │ │
│ │ ○ Encounters                                               │ │
│ │ ○ Invoices                                                 │ │
│ │ ○ Payments                                                 │ │
│ │ ○ Lab Orders                                               │ │
│ │ ○ Prescriptions                                            │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ 2. SELECT COLUMNS                                                │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Available Columns          Selected Columns               │ │
│ │ ┌──────────────────┐      ┌──────────────────┐           │ │
│ │ │ □ Patient Number │      │ ☑ Full Name      │ [↑][↓][×]│ │
│ │ │ □ DNI            │      │ ☑ Age            │ [↑][↓][×]│ │
│ │ │ □ Email          │      │ ☑ Insurance      │ [↑][↓][×]│ │
│ │ │ □ Phone          │      │ ☑ Last Visit     │ [↑][↓][×]│ │
│ │ │ □ Address        │      │ ☑ Total Visits   │ [↑][↓][×]│ │
│ │ │ ...              │      │                  │           │ │
│ │ └──────────────────┘      └──────────────────┘           │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ 3. ADD FILTERS                                                   │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ • Age >= [18]                               [Edit] [Remove]│ │
│ │ • Insurance = [OSDE]                        [Edit] [Remove]│ │
│ │ • Last Visit Date between [01/01/2025] and [31/12/2025]   │ │
│ │                                             [Edit] [Remove]│ │
│ │ [+ Add Filter]                                             │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ 4. GROUPING & AGGREGATION (Optional)                             │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Group By: [Insurance ▼]                                    │ │
│ │                                                            │ │
│ │ Aggregations:                                              │ │
│ │ • Count of patients                                        │ │
│ │ • Average age                                              │ │
│ │ • Sum of total visits                                      │ │
│ │ [+ Add Aggregation]                                        │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ 5. SORTING                                                       │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Sort By: [Last Visit Date ▼]  Order: [Descending ▼]       │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ 6. VISUALIZATION (Optional)                                      │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Chart Type: [Bar Chart ▼]                                  │ │
│ │ X-Axis: [Insurance]      Y-Axis: [Patient Count]          │ │
│ │ [Preview Chart]                                            │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ 7. SCHEDULING (Optional)                                         │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ ☑ Schedule this report                                     │ │
│ │ Frequency: [Weekly ▼]  Day: [Monday ▼]  Time: [09:00]     │ │
│ │ Email to: [admin@clinic.com]                               │ │
│ │ Format: [PDF ▼]                                            │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

#### 2.2.2 Saved Reports and Templates

```sql
CREATE TABLE saved_reports (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Report definition
    data_source             VARCHAR(50) NOT NULL, -- 'patients', 'appointments', etc.
    columns                 JSONB NOT NULL, -- Array of column definitions
    filters                 JSONB, -- Filter conditions
    grouping                JSONB, -- Grouping and aggregation rules
    sorting                 JSONB, -- Sort rules
    visualization           JSONB, -- Chart configuration

    -- Scheduling
    is_scheduled            BOOLEAN DEFAULT false,
    schedule_frequency      VARCHAR(50), -- 'daily', 'weekly', 'monthly'
    schedule_day            INTEGER, -- Day of week (1-7) or day of month (1-31)
    schedule_time           TIME,
    email_recipients        TEXT[], -- Array of email addresses
    export_format           VARCHAR(20), -- 'pdf', 'excel', 'csv'

    -- Access control
    created_by              BIGINT REFERENCES users(id),
    is_public               BOOLEAN DEFAULT false, -- Visible to all users
    allowed_roles           VARCHAR(50)[], -- Roles that can view this report

    -- Usage tracking
    last_run_at             TIMESTAMP,
    run_count               INTEGER DEFAULT 0,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_saved_reports_creator (created_by),
    INDEX idx_saved_reports_scheduled (is_scheduled, schedule_frequency)
);
```

### 2.3 Dashboard Configuration

#### 2.3.1 Customizable Dashboards

```sql
CREATE TABLE dashboards (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Layout configuration
    layout                  JSONB NOT NULL, -- Grid layout definition
    /* Example layout:
    {
      "grid": [
        { "widget_id": 1, "x": 0, "y": 0, "w": 6, "h": 4 },
        { "widget_id": 2, "x": 6, "y": 0, "w": 6, "h": 4 },
        { "widget_id": 3, "x": 0, "y": 4, "w": 12, "h": 6 }
      ]
    }
    */

    -- Access control
    created_by              BIGINT REFERENCES users(id),
    is_default              BOOLEAN DEFAULT false, -- Default dashboard for role
    default_for_role        VARCHAR(50), -- If default, which role
    is_public               BOOLEAN DEFAULT false,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE dashboard_widgets (
    id                      BIGSERIAL PRIMARY KEY,
    dashboard_id            BIGINT REFERENCES dashboards(id) ON DELETE CASCADE,

    -- Widget configuration
    widget_type             VARCHAR(50) NOT NULL, -- 'kpi', 'chart', 'table', 'calendar'
    title                   VARCHAR(200),

    -- Data configuration
    data_source             VARCHAR(50), -- Report or metric source
    saved_report_id         BIGINT REFERENCES saved_reports(id),
    configuration           JSONB NOT NULL,
    /* Example for KPI widget:
    {
      "metric": "total_patients",
      "comparison_period": "last_month",
      "show_trend": true,
      "color": "#10b981"
    }
    */

    -- Refresh settings
    auto_refresh            BOOLEAN DEFAULT false,
    refresh_interval        INTEGER, -- Seconds

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_dashboard_widgets_dashboard (dashboard_id)
);
```

#### 2.3.2 Dashboard Examples

**Admin Dashboard**
```
┌─────────────────────────────────────────────────────────────────┐
│ Admin Dashboard                                    [Customize]   │
├─────────────────────────────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│ │ Total    │ │ Today's  │ │ Pending  │ │ Revenue  │          │
│ │ Patients │ │ Appts    │ │ Claims   │ │ MTD      │          │
│ │  1,250   │ │   24     │ │   15     │ │ $125,000 │          │
│ │ ↑ 5.2%   │ │ ↓ 8.3%   │ │ → 0%     │ │ ↑ 12.5%  │          │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘          │
├─────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────┐ ┌────────────────────────────────┐ │
│ │ Revenue Trend (30d)    │ │ Top Diagnoses                  │ │
│ │ [Line Chart]           │ │ [Bar Chart]                    │ │
│ └────────────────────────┘ └────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Provider Utilization                                       │ │
│ │ [Table with sparklines]                                    │ │
│ │ Dr. Martinez    ████████░░  85%                            │ │
│ │ Dr. Garcia      ██████████  92%                            │ │
│ │ Dr. Lopez       █████░░░░░  55%                            │ │
│ └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Doctor Dashboard**
```
┌─────────────────────────────────────────────────────────────────┐
│ My Dashboard - Dr. Martinez                                      │
├─────────────────────────────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│ │ Today's  │ │ Pending  │ │ My       │                        │
│ │ Patients │ │ Results  │ │ Patients │                        │
│ │    8     │ │    3     │ │   156    │                        │
│ └──────────┘ └──────────┘ └──────────┘                        │
├─────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Today's Schedule                                           │ │
│ │ 09:00 - Juan Perez (OSDE) - Follow-up                      │ │
│ │ 09:30 - Maria Garcia (PAMI) - New patient                  │ │
│ │ 10:00 - Carlos Rodriguez (Particular) - Control            │ │
│ │ ...                                                        │ │
│ └────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────┐ ┌──────────────────────────────────┐ │
│ │ My Top Diagnoses     │ │ Pending Lab Results              │ │
│ │ [Pie Chart]          │ │ [Table]                          │ │
│ └──────────────────────┘ └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. ADVANCED ROLE & PERMISSION MANAGEMENT

### 3.1 Granular Permission System

#### 3.1.1 Permission Structure

```sql
CREATE TABLE permissions (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(100) UNIQUE NOT NULL, -- e.g., 'patient:create'
    description             TEXT,
    category                VARCHAR(50), -- 'patient', 'appointment', 'clinical', etc.
    is_system               BOOLEAN DEFAULT false, -- Cannot be deleted
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE roles (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(100) UNIQUE NOT NULL,
    description             TEXT,
    is_system               BOOLEAN DEFAULT false, -- System roles cannot be deleted
    priority                INTEGER DEFAULT 0, -- For role hierarchy
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE role_permissions (
    id                      BIGSERIAL PRIMARY KEY,
    role_id                 BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id           BIGINT NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(role_id, permission_id)
);

-- User can have multiple roles
CREATE TABLE user_roles (
    id                      BIGSERIAL PRIMARY KEY,
    user_id                 BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id                 BIGINT NOT NULL REFERENCES roles(id) ON DELETE CASCADE,

    -- Temporal roles (e.g., temporary admin access)
    valid_from              DATE,
    valid_until             DATE,

    granted_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, role_id)
);
```

#### 3.1.2 Comprehensive Permission List

```sql
-- Patient permissions
INSERT INTO permissions (name, description, category) VALUES
('patient:create', 'Create new patients', 'patient'),
('patient:read', 'View patient information', 'patient'),
('patient:read:all', 'View all patients (not just own)', 'patient'),
('patient:read:demographics', 'View patient demographics only', 'patient'),
('patient:read:clinical', 'View patient clinical data', 'patient'),
('patient:update', 'Edit patient information', 'patient'),
('patient:delete', 'Delete/archive patients', 'patient'),
('patient:merge', 'Merge duplicate patient records', 'patient'),
('patient:export', 'Export patient data', 'patient'),

-- Appointment permissions
('appointment:create', 'Create appointments', 'appointment'),
('appointment:read', 'View appointments', 'appointment'),
('appointment:read:all', 'View all appointments (not just own)', 'appointment'),
('appointment:update', 'Edit appointments', 'appointment'),
('appointment:cancel', 'Cancel appointments', 'appointment'),
('appointment:reschedule', 'Reschedule appointments', 'appointment'),
('appointment:check_in', 'Check in patients', 'appointment'),
('appointment:overbook', 'Create overbooked appointments', 'appointment'),

-- Clinical permissions
('encounter:create', 'Create clinical encounters', 'clinical'),
('encounter:read', 'View clinical encounters', 'clinical'),
('encounter:read:all', 'View all encounters (not just own)', 'clinical'),
('encounter:update', 'Edit clinical encounters', 'clinical'),
('encounter:finalize', 'Finalize and lock encounters', 'clinical'),
('encounter:delete', 'Delete draft encounters', 'clinical'),

('prescription:create', 'Create prescriptions', 'clinical'),
('prescription:read', 'View prescriptions', 'clinical'),
('prescription:update', 'Edit prescriptions', 'clinical'),
('prescription:cancel', 'Cancel prescriptions', 'clinical'),
('prescription:controlled', 'Prescribe controlled substances', 'clinical'),

('lab_order:create', 'Create lab orders', 'clinical'),
('lab_order:read', 'View lab orders', 'clinical'),
('lab_order:update', 'Edit lab orders', 'clinical'),
('lab_result:enter', 'Enter lab results', 'clinical'),
('lab_result:approve', 'Approve and release lab results', 'clinical'),

('vital_signs:create', 'Record vital signs', 'clinical'),
('vital_signs:read', 'View vital signs', 'clinical'),
('vital_signs:update', 'Edit vital signs', 'clinical'),

('diagnosis:create', 'Add diagnoses', 'clinical'),
('diagnosis:read', 'View diagnoses', 'clinical'),
('diagnosis:update', 'Edit diagnoses', 'clinical'),

-- Billing permissions
('invoice:create', 'Create invoices', 'billing'),
('invoice:read', 'View invoices', 'billing'),
('invoice:read:all', 'View all invoices', 'billing'),
('invoice:update', 'Edit invoices', 'billing'),
('invoice:delete', 'Delete draft invoices', 'billing'),
('invoice:request_cae', 'Request CAE from AFIP', 'billing'),
('invoice:cancel', 'Cancel invoices (credit notes)', 'billing'),

('payment:create', 'Record payments', 'billing'),
('payment:read', 'View payments', 'billing'),
('payment:update', 'Edit payments', 'billing'),
('payment:delete', 'Delete payments', 'billing'),
('payment:refund', 'Process refunds', 'billing'),

('insurance_claim:create', 'Create insurance claims', 'billing'),
('insurance_claim:read', 'View insurance claims', 'billing'),
('insurance_claim:update', 'Update claim status', 'billing'),

-- Inventory permissions
('inventory:read', 'View inventory', 'inventory'),
('inventory:create', 'Add inventory items', 'inventory'),
('inventory:update', 'Edit inventory items', 'inventory'),
('inventory:delete', 'Delete inventory items', 'inventory'),
('inventory:transact', 'Record inventory transactions', 'inventory'),
('inventory:adjust', 'Adjust stock levels', 'inventory'),

-- Report permissions
('report:view:clinical', 'View clinical reports', 'report'),
('report:view:financial', 'View financial reports', 'report'),
('report:view:operational', 'View operational reports', 'report'),
('report:view:all', 'View all reports', 'report'),
('report:create', 'Create custom reports', 'report'),
('report:export', 'Export reports', 'report'),
('report:schedule', 'Schedule automated reports', 'report'),

-- User management permissions
('user:create', 'Create users', 'admin'),
('user:read', 'View users', 'admin'),
('user:update', 'Edit users', 'admin'),
('user:delete', 'Delete users', 'admin'),
('user:assign_role', 'Assign roles to users', 'admin'),
('user:reset_password', 'Reset user passwords', 'admin'),

-- Role management permissions
('role:create', 'Create custom roles', 'admin'),
('role:read', 'View roles', 'admin'),
('role:update', 'Edit roles', 'admin'),
('role:delete', 'Delete custom roles', 'admin'),
('role:assign_permission', 'Assign permissions to roles', 'admin'),

-- System permissions
('settings:read', 'View system settings', 'admin'),
('settings:update', 'Edit system settings', 'admin'),
('settings:backup', 'Perform system backup', 'admin'),
('settings:restore', 'Restore from backup', 'admin'),

('audit:read', 'View audit logs', 'admin'),
('audit:export', 'Export audit logs', 'admin'),

-- Communication permissions
('communication:send_sms', 'Send SMS messages', 'communication'),
('communication:send_email', 'Send email messages', 'communication'),
('communication:send_whatsapp', 'Send WhatsApp messages', 'communication'),
('communication:view_logs', 'View communication logs', 'communication');
```

#### 3.1.3 Default Role Configurations

```sql
-- SUPER_ADMIN: All permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 1, id FROM permissions; -- Role ID 1 = SUPER_ADMIN

-- ADMIN: Most permissions except super admin functions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 2, id FROM permissions
WHERE name NOT IN ('user:delete', 'role:delete', 'settings:restore');

-- DOCTOR: Clinical and patient permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 3, id FROM permissions
WHERE name IN (
    'patient:create', 'patient:read', 'patient:update', 'patient:read:clinical',
    'appointment:create', 'appointment:read', 'appointment:update', 'appointment:cancel',
    'encounter:create', 'encounter:read', 'encounter:update', 'encounter:finalize',
    'prescription:create', 'prescription:read', 'prescription:controlled',
    'lab_order:create', 'lab_order:read',
    'vital_signs:create', 'vital_signs:read',
    'diagnosis:create', 'diagnosis:read', 'diagnosis:update',
    'report:view:clinical', 'report:export'
);

-- NURSE: Limited clinical permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 4, id FROM permissions
WHERE name IN (
    'patient:read', 'patient:read:demographics',
    'appointment:read', 'appointment:check_in',
    'encounter:read',
    'vital_signs:create', 'vital_signs:read', 'vital_signs:update',
    'lab_result:enter',
    'inventory:read', 'inventory:transact'
);

-- RECEPTIONIST: Patient and appointment management
INSERT INTO role_permissions (role_id, permission_id)
SELECT 5, id FROM permissions
WHERE name IN (
    'patient:create', 'patient:read', 'patient:read:demographics', 'patient:update',
    'appointment:create', 'appointment:read', 'appointment:read:all',
    'appointment:update', 'appointment:cancel', 'appointment:reschedule',
    'appointment:check_in',
    'communication:send_sms', 'communication:send_email'
);

-- BILLING: Financial permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 6, id FROM permissions
WHERE name IN (
    'patient:read', 'patient:read:demographics',
    'invoice:create', 'invoice:read', 'invoice:read:all', 'invoice:update',
    'invoice:request_cae', 'invoice:cancel',
    'payment:create', 'payment:read', 'payment:update', 'payment:refund',
    'insurance_claim:create', 'insurance_claim:read', 'insurance_claim:update',
    'report:view:financial', 'report:export'
);

-- LAB_TECH: Laboratory permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 7, id FROM permissions
WHERE name IN (
    'patient:read', 'patient:read:demographics',
    'lab_order:read',
    'lab_result:enter', 'lab_result:approve'
);
```

### 3.2 Custom Role Builder UI

```
┌──────────────────────────────────────────────────────────────────┐
│ Custom Role Builder                            [Save] [Cancel]   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Role Name: [Medical Assistant]                                  │
│ Description: [Assists doctors with patient care and admin tasks]│
│                                                                  │
│ Base Role (Optional): [Nurse ▼] [Copy Permissions]              │
│                                                                  │
│ Permissions:                                                     │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ [▼] Patient Management                                     │ │
│ │     ☑ Create patients                                      │ │
│ │     ☑ View patient demographics                            │ │
│ │     ☑ View patient clinical data                           │ │
│ │     ☑ Edit patient information                             │ │
│ │     ☐ Delete patients                                      │ │
│ │     ☐ Merge patient records                                │ │
│ │                                                            │ │
│ │ [▼] Appointments                                           │ │
│ │     ☑ Create appointments                                  │ │
│ │     ☑ View appointments                                    │ │
│ │     ☑ Edit appointments                                    │ │
│ │     ☑ Cancel appointments                                  │ │
│ │     ☑ Check in patients                                    │ │
│ │     ☐ Allow overbooking                                    │ │
│ │                                                            │ │
│ │ [▼] Clinical                                               │ │
│ │     ☑ View encounters                                      │ │
│ │     ☐ Create encounters                                    │ │
│ │     ☑ Record vital signs                                   │ │
│ │     ☐ Create prescriptions                                 │ │
│ │     ☑ Create lab orders                                    │ │
│ │     ☑ Enter lab results                                    │ │
│ │                                                            │ │
│ │ [▼] Billing                                                │ │
│ │     ☐ Create invoices                                      │ │
│ │     ☑ View invoices                                        │ │
│ │     ☐ Process payments                                     │ │
│ │                                                            │ │
│ │ [▼] Inventory                                              │ │
│ │     ☑ View inventory                                       │ │
│ │     ☑ Record usage                                         │ │
│ │     ☐ Adjust stock levels                                  │ │
│ │                                                            │ │
│ │ [▼] Reports                                                │ │
│ │     ☑ View clinical reports                                │ │
│ │     ☐ View financial reports                               │ │
│ │     ☑ Export reports                                       │ │
│ │                                                            │ │
│ │ [▼] Communication                                          │ │
│ │     ☑ Send SMS                                             │ │
│ │     ☑ Send email                                           │ │
│ │     ☐ Send WhatsApp                                        │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Summary: 18 permissions selected                                 │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 3.3 Multi-Role Users

```sql
-- Example: User with multiple roles
INSERT INTO user_roles (user_id, role_id) VALUES
(10, 3), -- Doctor role
(10, 5); -- Also has receptionist role (covers front desk when needed)

-- Temporary role assignment
INSERT INTO user_roles (user_id, role_id, valid_from, valid_until, granted_by) VALUES
(15, 2, '2025-12-01', '2025-12-31', 1); -- Temporary admin access for December
```

**Permission Resolution:**
```javascript
// Get effective permissions for user (union of all roles)
async function getUserPermissions(userId) {
  const permissions = await db.query(`
    SELECT DISTINCT p.name
    FROM user_roles ur
    JOIN role_permissions rp ON ur.role_id = rp.role_id
    JOIN permissions p ON rp.permission_id = p.id
    WHERE ur.user_id = $1
      AND (ur.valid_from IS NULL OR ur.valid_from <= CURRENT_DATE)
      AND (ur.valid_until IS NULL OR ur.valid_until >= CURRENT_DATE)
  `, [userId]);

  return permissions.map(p => p.name);
}
```

### 3.4 Permission Delegation

```sql
CREATE TABLE permission_delegations (
    id                      BIGSERIAL PRIMARY KEY,
    from_user_id            BIGINT NOT NULL REFERENCES users(id),
    to_user_id              BIGINT NOT NULL REFERENCES users(id),
    permission_id           BIGINT NOT NULL REFERENCES permissions(id),

    -- Temporal delegation
    valid_from              TIMESTAMP NOT NULL,
    valid_until             TIMESTAMP NOT NULL,

    reason                  TEXT,
    is_active               BOOLEAN DEFAULT true,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_delegation_to (to_user_id),
    INDEX idx_delegation_dates (valid_from, valid_until)
);
```

**Use Case:**
Dr. Martinez delegates their `encounter:finalize` permission to Dr. Garcia while on vacation.

---

## 4. ADDITIONAL ADVANCED FEATURES

### 4.1 Appointment Templates

```sql
CREATE TABLE appointment_templates (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Appointment series configuration
    appointment_type_id     BIGINT REFERENCES appointment_types(id),
    duration                INTEGER,

    -- Recurrence pattern
    recurrence_type         VARCHAR(50), -- 'weekly', 'biweekly', 'monthly'
    recurrence_days         INTEGER[], -- Days of week [1,3,5] for Mon, Wed, Fri
    recurrence_times        TIME[], -- Specific times

    -- Series length
    occurrence_count        INTEGER, -- Number of appointments, OR
    end_date                DATE, -- End date

    -- Default values
    default_notes           TEXT,

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Use Case:**
- Physical therapy: 3 times per week for 6 weeks
- Chemotherapy: Every 2 weeks for 6 months
- Follow-up series: Monthly for 1 year

### 4.2 Waitlist with Preferences

```sql
CREATE TABLE appointment_waitlist (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT NOT NULL REFERENCES patients(id),
    provider_id             BIGINT REFERENCES users(id),
    appointment_type_id     BIGINT REFERENCES appointment_types(id),

    -- Preferences
    preferred_days          INTEGER[], -- [1,2,3] for Mon, Tue, Wed
    preferred_time_start    TIME,
    preferred_time_end      TIME,
    preferred_locations     BIGINT[], -- Array of location IDs

    -- Flexibility
    accept_any_provider     BOOLEAN DEFAULT false,
    accept_any_time         BOOLEAN DEFAULT false,

    -- Urgency
    priority                INTEGER DEFAULT 0, -- Higher = more urgent
    requested_by_date       DATE, -- Needs appointment by this date

    -- Status
    status                  VARCHAR(50) DEFAULT 'active', -- active, notified, converted, expired
    notified_at             TIMESTAMP,
    expires_at              TIMESTAMP,

    notes                   TEXT,

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_waitlist_patient (patient_id),
    INDEX idx_waitlist_provider (provider_id),
    INDEX idx_waitlist_status (status),
    INDEX idx_waitlist_priority (priority DESC)
);
```

### 4.3 Provider Groups / Care Teams

```sql
CREATE TABLE provider_groups (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,
    group_type              VARCHAR(50), -- 'care_team', 'specialty', 'location'

    -- Scheduling
    allow_cross_booking     BOOLEAN DEFAULT false, -- Can book with any member
    share_patients          BOOLEAN DEFAULT false, -- All members see all patients

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE provider_group_members (
    id                      BIGSERIAL PRIMARY KEY,
    group_id                BIGINT NOT NULL REFERENCES provider_groups(id),
    provider_id             BIGINT NOT NULL REFERENCES users(id),
    role                    VARCHAR(50), -- 'lead', 'member', 'backup'

    joined_at               TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(group_id, provider_id)
);
```

**Use Case:**
- Cardiology team shares patients
- Backup doctors can cover each other's appointments
- Specialty groups for referrals

---

*Advanced Features Specification Version: 1.0*
*Last Updated: 2025-11-15*
*Comprehensive advanced features for maximum flexibility*
