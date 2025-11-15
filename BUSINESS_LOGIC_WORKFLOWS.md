# Clinical Management System - Business Logic & Workflows

## Overview

This document defines the business rules, validation requirements, state machines, and workflows for all major processes in the clinical management system.

---

## 1. PATIENT MANAGEMENT

### 1.1 Patient Registration Business Rules

#### Validation Rules
- **DNI:**
  - Must be 7-8 digits for Argentine DNI
  - Cannot be duplicate in system
  - Required for adult patients (>18)

- **Date of Birth:**
  - Cannot be in the future
  - Cannot be more than 120 years ago
  - Auto-calculate age from DOB

- **Phone:**
  - Must be valid Argentine phone format
  - At least one phone number required
  - Format: +54 9 11 XXXX-XXXX or +54 XXX XXX-XXXX

- **Email:**
  - Must be valid email format (RFC 5322)
  - Optional but recommended
  - Cannot be duplicate if provided

- **Patient Number:**
  - Auto-generated on creation
  - Format: P-XXXXX (sequential, zero-padded)
  - Unique across system

#### Business Logic

**Duplicate Patient Detection:**
```
When creating/editing a patient:
1. Check if DNI already exists
   - If exists: Show warning "Patient with DNI XXX already exists"
   - Offer to view existing patient
   - Allow override with confirmation (merge duplicate feature)

2. Fuzzy match on name + date of birth
   - If similar patient found (>85% match)
   - Show warning with potential duplicates
   - Allow to proceed or select existing patient
```

**Age Calculation:**
```
Age = Current Date - Date of Birth (in years)
Display format:
- If < 1 year: "X months"
- If < 1 month: "X days"
- Otherwise: "X years"
```

**Emergency Contact:**
```
If patient is minor (<18):
- Emergency contact is REQUIRED
- Must be parent or legal guardian
Else:
- Emergency contact is optional but recommended
```

### 1.2 Insurance Assignment

#### Business Rules

**Primary Insurance:**
```
- Patient can have multiple insurances
- Exactly ONE must be marked as primary
- When adding first insurance: automatically mark as primary
- When adding subsequent: ask if this should be primary
- If marking new insurance as primary: unmark current primary
```

**Coverage Validation:**
```
When using insurance for billing:
1. Check if insurance is active (valid_until >= current date)
2. Check if member number is present
3. If PAMI: validate age >= 60 or disability status
4. If obra social: verify coverage percentage and copay amount
```

**Insurance Coverage Percentage:**
```
- Value must be between 0 and 100
- If 100%: no patient responsibility
- If <100%: calculate patient copay = total * (1 - coverage%/100)
```

### 1.3 Medical History

#### Allergy Management

**Severity Levels:**
- **Critical:** Life-threatening (anaphylaxis)
- **High:** Severe reaction required hospitalization
- **Moderate:** Significant symptoms
- **Mild:** Minor reactions

**Allergy Types:**
- Medication
- Food
- Environmental
- Other

**Business Logic:**
```
When prescribing medication:
1. Check patient allergies
2. If medication or drug class matches allergy:
   - Show CRITICAL WARNING (red, prominent)
   - Require confirmation with reason to override
   - Log override in audit trail
3. Display all allergies prominently on prescription screen
```

---

## 2. APPOINTMENT SCHEDULING

### 2.1 Appointment State Machine

```
States:
┌──────────┐
│Scheduled │ ──► [Confirm] ──► ┌───────────┐
└──────────┘                   │ Confirmed │
     │                         └───────────┘
     │                              │
     ▼                              ▼
┌──────────┐                   ┌─────────┐
│Cancelled │                   │ Arrived │
└──────────┘                   └─────────┘
                                    │
                                    ▼
                              ┌─────────────┐
                              │ In Progress │
                              └─────────────┘
                                    │
                      ┌─────────────┼─────────────┐
                      ▼             ▼             ▼
                 ┌──────────┐ ┌──────────┐ ┌─────────┐
                 │Completed │ │Cancelled │ │ No Show │
                 └──────────┘ └──────────┘ └─────────┘
```

**State Transitions:**

```
SCHEDULED → CONFIRMED (Patient confirms via SMS/call)
SCHEDULED → CANCELLED (Patient/clinic cancels before appointment)
SCHEDULED → NO_SHOW (Appointment time passes without check-in)

CONFIRMED → ARRIVED (Patient checks in at reception)
CONFIRMED → CANCELLED (Patient cancels)
CONFIRMED → NO_SHOW (Appointment time passes without check-in)

ARRIVED → IN_PROGRESS (Doctor starts consultation)
ARRIVED → CANCELLED (Patient leaves before seeing doctor)

IN_PROGRESS → COMPLETED (Consultation finished)
IN_PROGRESS → CANCELLED (Emergency/interruption)
```

**Business Rules:**

```
- Cannot cancel appointment if status is COMPLETED
- Cannot reschedule NO_SHOW without marking as new appointment
- If CANCELLED and rescheduling: create new appointment, link to cancelled
- Auto-transition to NO_SHOW if 30 minutes past appointment time and status = SCHEDULED/CONFIRMED
```

### 2.2 Appointment Booking Rules

#### Time Slot Validation

```
When booking appointment:

1. Check provider schedule
   - Verify provider works on selected day
   - Verify selected time is within work hours

2. Check for conflicts
   - No overlapping appointments for same provider
   - Consider buffer time (5 min between appointments)

3. Check provider time-off
   - Verify provider not on vacation
   - Verify no blocked time slots

4. Validate appointment duration
   - Duration must match appointment type default
   - Can be overridden with reason

5. Check clinic hours
   - Appointment must end before clinic closes
```

#### Overbooking Rules

```
Overbooking Allowed:
- Emergency appointments (max 2 per hour)
- Only with provider permission
- Show warning on calendar

Overbooking Not Allowed:
- Regular appointments
- If provider already has 3+ appointments in same hour
```

#### Future Booking Limits

```
- Maximum booking: 6 months in advance
- Minimum booking: Same day allowed
- For same-day: verify reason if < 2 hours from current time
```

### 2.3 Appointment Reminders

#### Reminder Schedule

```
Reminder Timeline:
1. 24 hours before: Send SMS + Email
2. 2 hours before: Send SMS
3. (Optional) WhatsApp at 24 hours

Patient Preferences:
- Can opt-out of reminders
- Can choose SMS only, Email only, or both
- Can set custom reminder time
```

#### Reminder Logic

```
Daily at 8:00 AM, run reminder job:

For each appointment in next 24-28 hours:
  If reminder not sent AND patient opted-in:
    - Send reminder via configured channels
    - Mark reminder_sent_at timestamp
    - Include: Date, Time, Doctor, Location, Appointment type
    - Include confirmation link (optional)

For each appointment in next 2 hours:
  If within 2 hour window AND second reminder not sent:
    - Send SMS reminder
    - Include: "Your appointment is in 2 hours"
```

### 2.4 Waitlist Management

#### Waitlist Rules

```
When appointment is cancelled:
1. Check waitlist for matching:
   - Same provider (preferred)
   - Same approximate date/time
   - Same appointment type

2. If match found:
   - Notify waitlist patient (SMS/Email)
   - Mark waitlist entry as "Notified"
   - Give patient 24 hours to respond

3. If patient accepts:
   - Book appointment
   - Remove from waitlist

4. If patient declines or no response in 24h:
   - Move to next waitlist patient
```

---

## 3. CLINICAL ENCOUNTERS

### 3.1 Encounter Workflow

```
Workflow Steps:

1. START ENCOUNTER
   ├─ From appointment (link to appointment_id)
   └─ Walk-in (no appointment)

2. RECORD VITAL SIGNS
   ├─ Nurse/MA records vitals
   └─ Auto-attach to encounter

3. SOAP NOTES
   ├─ Doctor enters Subjective
   ├─ Doctor enters Objective
   ├─ Doctor enters Assessment
   └─ Doctor enters Plan

4. ADD DIAGNOSES (optional)
   └─ Link ICD-10 codes

5. CREATE ORDERS (optional)
   ├─ Prescriptions
   ├─ Lab orders
   └─ Imaging orders

6. SAVE AS DRAFT
   └─ Can return later to complete

7. FINALIZE ENCOUNTER
   ├─ Lock all fields (no further edits)
   ├─ Generate encounter summary
   └─ Trigger billing (create invoice)

8. END
```

### 3.2 Encounter Business Rules

#### Vital Signs

```
Valid Ranges (Adult):
- Systolic BP: 70-250 mmHg
- Diastolic BP: 40-150 mmHg
- Heart Rate: 40-200 bpm
- Temperature: 35-42°C
- Respiratory Rate: 8-40 breaths/min
- SpO2: 70-100%
- Weight: 20-300 kg
- Height: 50-250 cm

Warnings:
- If outside normal range: Show yellow warning
- If critical (e.g., BP > 180/120): Show red alert
- Suggest immediate action for critical values
```

#### BMI Calculation

```
BMI = Weight (kg) / (Height (m))²

Categories:
- < 18.5: Underweight
- 18.5 - 24.9: Normal
- 25 - 29.9: Overweight
- 30 - 34.9: Obese Class I
- 35 - 39.9: Obese Class II
- >= 40: Obese Class III

Display category with color coding
```

#### Encounter Finalization

```
Before finalizing:
1. Verify all required fields completed:
   - Chief complaint OR subjective
   - At least one vital sign
   - Assessment OR diagnosis

2. Check for incomplete orders:
   - Warn if prescription created but not printed
   - Warn if lab order created but not printed

3. Billing check:
   - If no invoice created: prompt to create
   - If insurance patient: verify coverage

4. Lock encounter:
   - Set status = "finalized"
   - Set finalized_at timestamp
   - No edits allowed after finalization
   - Create audit log entry
```

---

## 4. PRESCRIPTION MANAGEMENT

### 4.1 Prescription Creation Workflow

```
Workflow:

1. SELECT PATIENT
   └─ Must have active patient context

2. ADD MEDICATIONS
   For each medication:
   ├─ Search medication database
   ├─ Select dosage & form
   ├─ Specify frequency & duration
   ├─ Calculate quantity needed
   ├─ Add instructions
   └─ Run safety checks:
       ├─ Check allergies
       ├─ Check drug interactions
       └─ Check contraindications

3. REVIEW WARNINGS
   ├─ Address any critical warnings
   └─ Document override reason if proceeding

4. MARK AS CONTROLLED (if applicable)
   └─ Requires special prescription form

5. SAVE PRESCRIPTION
   ├─ Generate prescription number
   └─ Set status = "active"

6. PRINT/SEND
   ├─ Print on prescription pad
   ├─ Email to patient
   └─ Mark as dispensed (optional)

7. END
```

### 4.2 Drug Interaction Checking

#### Interaction Severity

```
Severity Levels:
1. CRITICAL (Red)
   - Contraindicated - do not use together
   - Example: Warfarin + Aspirin (bleeding risk)
   - Action: Block prescription, require override with documentation

2. MAJOR (Orange)
   - Serious interaction - monitor closely
   - Example: Metformin + Alcohol (lactic acidosis)
   - Action: Show warning, suggest alternatives, allow override

3. MODERATE (Yellow)
   - Moderate interaction - caution advised
   - Action: Show warning, allow prescription

4. MINOR (Blue)
   - Minimal interaction
   - Action: Informational note only
```

#### Checking Logic

```
When adding medication to prescription:

1. Check against patient's active medications:
   For each active medication:
     - Query interaction database
     - If interaction found:
       Display warning with:
       - Severity level
       - Mechanism of interaction
       - Clinical effects
       - Management recommendations

2. Check against medications in current prescription:
   - Same checks as above
   - Useful for multi-drug prescriptions

3. Check against patient allergies:
   - Drug name match
   - Drug class match
   - If match: CRITICAL warning

4. Check contraindications:
   - Check against patient diagnoses
   - Check against patient age
   - Check pregnancy status (if applicable)
```

### 4.3 Dosage Calculation

```
Pediatric Dosing (< 18 years):

If medication has pediatric dosing:
  Calculate based on:
  - Weight (mg/kg)
  - Body surface area (mg/m²)
  - Age-based dosing

  Display:
  - Recommended dose range
  - Maximum single dose
  - Maximum daily dose

  Warning if prescribed dose is:
  - < 50% of recommended: "Dose may be subtherapeutic"
  - > 110% of maximum: "Dose exceeds maximum - verify"
```

### 4.4 Chronic Medication Management

```
Chronic Medications:

When marking medication as chronic:
1. Set is_chronic = true
2. Allow unlimited refills OR set high refill count
3. Add to patient's active medication list
4. Show on patient profile prominently
5. Include in medication reconciliation
6. Prompt for renewal at intervals (e.g., every 3 months)

Medication Reconciliation:
- At each encounter, review active chronic medications
- Ask patient: "Are you still taking [medication]?"
- Update adherence status
- Discontinue if no longer taking
```

---

## 5. LABORATORY MANAGEMENT

### 5.1 Lab Order Workflow

```
Workflow:

1. CREATE LAB ORDER
   ├─ Select patient
   └─ Link to encounter (optional)

2. SELECT TESTS
   ├─ Choose from common panels, OR
   └─ Select individual tests

3. SET PRIORITY
   ├─ Routine (default)
   ├─ Urgent (same day)
   └─ STAT (immediate)

4. ADD REQUIREMENTS
   ├─ Fasting required? (Yes/No)
   ├─ Special instructions
   └─ Clinical indication

5. SAVE & PRINT ORDER
   ├─ Generate order number
   ├─ Print for patient to take to lab
   └─ Or send electronically if integrated

6. AWAIT RESULTS
   └─ Status = "ordered"

7. RECEIVE RESULTS
   ├─ Manual entry, OR
   ├─ Electronic import (HL7)
   └─ Status = "completed"

8. REVIEW RESULTS
   ├─ Doctor reviews
   ├─ Flag abnormals
   └─ Add interpretation notes

9. NOTIFY PATIENT
   ├─ Via patient portal
   ├─ Via SMS/Email
   └─ Or call if critical

10. END
```

### 5.2 Result Flagging Logic

```
Automatic Flagging:

For each result:
1. Compare result value to reference range

2. If numeric result:
   - If < reference min: Flag as "LOW"
   - If > reference max: Flag as "HIGH"
   - If within range: Flag as "NORMAL"

3. If critical value:
   - If in critical range (defined per test): Flag as "CRITICAL"
   - Notify doctor immediately
   - Send SMS/Email alert

4. Color coding:
   - Green: Normal
   - Yellow: Borderline (within 10% of range boundary)
   - Orange: Abnormal (outside range)
   - Red: Critical

Examples:
- Hemoglobin: 14.5 g/dL (ref: 12-16) → NORMAL (Green)
- Glucose: 180 mg/dL (ref: 70-100) → HIGH (Orange)
- Glucose: 350 mg/dL (ref: 70-100) → CRITICAL (Red)
```

### 5.3 Critical Result Notification

```
Critical Result Workflow:

When critical result entered:
1. System auto-detects based on critical value rules
2. Set flag = "CRITICAL"
3. Create high-priority notification for ordering provider
4. Send immediate SMS/Email alert to provider
5. Log in critical results register
6. Track acknowledgment:
   - Provider must acknowledge within 1 hour
   - Escalate to supervisor if no acknowledgment
7. Require action documentation:
   - What action was taken?
   - Was patient notified?
   - Follow-up plan?
```

---

## 6. BILLING & INVOICING

### 6.1 Invoice Creation Workflow

```
Workflow:

1. TRIGGER
   ├─ From completed encounter, OR
   ├─ Manual invoice creation
   └─ Scheduled billing (monthly for subscriptions)

2. SELECT PATIENT & SERVICES
   ├─ Auto-populate from encounter, OR
   ├─ Manual service selection
   └─ Verify insurance coverage

3. DETERMINE INVOICE TYPE
   Decision tree:
   ├─ If patient has RUT/CUIT: Invoice Type A
   ├─ If consumer (no tax ID): Invoice Type B
   └─ If tax-exempt: Invoice Type C

4. CALCULATE AMOUNTS
   For each service:
   ├─ Base price (from service catalog or insurance rate)
   ├─ Apply insurance coverage %
   ├─ Calculate patient copay
   ├─ Calculate tax (if applicable)
   └─ Sum totals

5. REQUEST CAE FROM AFIP
   ├─ Build AFIP request with invoice data
   ├─ Send to AFIP webservice
   ├─ Receive CAE and expiration date
   ├─ If error: handle rejection and retry
   └─ Attach CAE to invoice

6. FINALIZE INVOICE
   ├─ Set status = "finalized"
   ├─ Generate QR code (AFIP requirement)
   ├─ Generate PDF
   └─ Store invoice

7. PROCESS PAYMENT (if immediate)
   └─ Record payment transaction

8. END
```

### 6.2 Invoice Type Selection Logic

```
Invoice Type Determination:

IF patient or responsible party has CUIT/RUT:
  └─ Invoice Type A
     ├─ Includes IVA breakdown
     ├─ Requires tax ID on invoice
     └─ Full tax documentation

ELSE IF regular consumer:
  └─ Invoice Type B
     ├─ IVA included in total (not broken out)
     ├─ No tax ID required
     └─ Simplified format

ELSE IF tax-exempt entity:
  └─ Invoice Type C
     └─ No IVA charged
```

### 6.3 Insurance Billing Logic

```
When billing with insurance:

1. VERIFY COVERAGE
   ├─ Check insurance is active
   ├─ Check member number is valid
   └─ Verify service is covered

2. CHECK AUTHORIZATION
   ├─ If service requires prior auth:
   │   ├─ Verify authorization number present
   │   └─ Check authorization is valid
   └─ If no auth: warn user, allow override

3. CALCULATE PAYMENT SPLIT
   Insurance Coverage:
   ├─ Service price: $5,000
   ├─ Insurance covers: 80% = $4,000
   └─ Patient copay: 20% = $1,000

4. CREATE INVOICE FOR PATIENT
   └─ Amount: $1,000 (patient responsibility)

5. CREATE CLAIM FOR INSURANCE
   └─ Amount: $4,000 (insurance responsibility)

6. TRACK SEPARATELY
   ├─ Patient payment in invoices table
   └─ Insurance payment in claims table
```

### 6.4 AFIP CAE Request Logic

```
CAE Request Process:

1. BUILD REQUEST
   Required data:
   ├─ Invoice type (A, B, C)
   ├─ Point of sale (punto de venta)
   ├─ Invoice number
   ├─ Invoice date
   ├─ Customer data (if Type A)
   ├─ Line items with amounts
   ├─ Tax breakdown
   └─ Total amount

2. AUTHENTICATE WITH AFIP
   ├─ Use X.509 certificate
   ├─ Generate authentication token
   └─ Token valid for 12 hours

3. SUBMIT CAE REQUEST
   ├─ Send SOAP request to AFIP webservice
   └─ Endpoint: wsfe (Factura Electrónica)

4. HANDLE RESPONSE
   Success:
   ├─ Receive CAE (14-digit number)
   ├─ Receive CAE expiration date
   ├─ Store CAE in invoice record
   └─ Generate QR code with invoice data

   Error:
   ├─ Parse error code and message
   ├─ Display user-friendly error
   ├─ Log error for troubleshooting
   ├─ Allow manual retry
   └─ If persistent: allow offline mode with later sync

5. GENERATE QR CODE
   QR contains:
   - URL: https://www.afip.gob.ar/fe/qr/?p=[base64_data]
   - Data: CUIT, invoice type, point of sale, number, amount, CAE
```

### 6.5 Payment Processing

#### Payment Workflow

```
Record Payment:

1. SELECT INVOICE
   └─ Load invoice with balance due

2. ENTER PAYMENT DETAILS
   ├─ Payment amount (can be partial)
   ├─ Payment method
   ├─ Payment date
   └─ Reference/Transaction ID

3. VALIDATE
   ├─ Amount > 0
   ├─ Amount <= balance due
   └─ Date not in future

4. PROCESS PAYMENT
   For cash:
   └─ Record immediately

   For card:
   ├─ Integrate with payment processor
   ├─ Await confirmation
   └─ Record with transaction ID

   For bank transfer:
   ├─ Record as pending
   └─ Update when confirmed

5. UPDATE INVOICE
   ├─ Add to amount_paid
   ├─ Recalculate balance_due
   ├─ If balance_due = 0: status = "paid"
   └─ If 0 < balance_due < total: status = "partially_paid"

6. GENERATE RECEIPT
   ├─ Receipt number (sequential)
   ├─ Payment details
   ├─ Remaining balance
   └─ Print or email to patient

7. END
```

#### Partial Payment Logic

```
Partial Payments Allowed:
- Patient can pay any amount >= minimum ($1)
- Multiple payments can be applied to single invoice
- Track all payments in payments table
- Show payment history on invoice

Payment Allocation:
- If patient has multiple outstanding invoices:
  ├─ By default: apply to oldest invoice first
  ├─ Allow user to specify which invoice
  └─ Can split payment across multiple invoices
```

---

## 7. INVENTORY MANAGEMENT

### 7.1 Stock Level Management

```
Stock Status:

For each inventory item:

1. CALCULATE STATUS
   IF current_stock <= 0:
     └─ Status: OUT_OF_STOCK (Red)
   ELSE IF current_stock <= minimum_stock:
     └─ Status: LOW_STOCK (Yellow)
   ELSE IF current_stock <= reorder_point:
     └─ Status: REORDER_NEEDED (Amber)
   ELSE:
     └─ Status: ADEQUATE (Green)

2. GENERATE ALERTS
   Daily check at 9:00 AM:
   ├─ Find items with status OUT_OF_STOCK or LOW_STOCK
   ├─ Create notification for inventory manager
   └─ Send email with low stock report

3. AUTO-REORDER (Optional)
   IF auto_reorder_enabled AND current_stock <= reorder_point:
   ├─ Generate purchase order
   ├─ Quantity = (maximum_stock - current_stock)
   ├─ Send to preferred supplier
   └─ Mark as "pending order"
```

### 7.2 Inventory Transaction Rules

```
Stock IN (Receiving):
1. Create transaction with type = "in"
2. Add quantity to current_stock
3. Create or update batch record
4. Record unit cost for cost tracking
5. Update last_received_date

Stock OUT (Usage):
1. Create transaction with type = "out"
2. Verify current_stock >= quantity (prevent negative stock)
3. Subtract quantity from current_stock
4. If batch tracking enabled: use FIFO (First In, First Out)
5. Link to patient encounter if used in patient care

Stock ADJUSTMENT:
1. Create transaction with type = "adjustment"
2. Calculate difference: new_stock - current_stock
3. Set current_stock = new_stock
4. Require reason for adjustment
5. Audit log for accountability
```

### 7.3 Expiration Date Tracking

```
Expiration Checking:

Daily job at 8:00 AM:

1. FIND EXPIRING ITEMS
   Criteria:
   ├─ Expiration date within next 30 days: EXPIRING_SOON
   ├─ Expiration date within next 7 days: EXPIRING_URGENT
   └─ Expiration date in past: EXPIRED

2. GENERATE ALERTS
   ├─ EXPIRED items: Critical alert, prevent usage
   ├─ EXPIRING_URGENT: High priority notification
   └─ EXPIRING_SOON: Warning notification

3. AUTOMATIC ACTIONS
   ├─ Mark expired items as unavailable
   ├─ Suggest waste/disposal transaction
   └─ Exclude from available stock count

4. FIFO ENFORCEMENT
   When dispensing:
   ├─ Use batch with earliest expiration date first
   └─ System suggests which batch to use
```

---

## 8. REPORTING BUSINESS LOGIC

### 8.1 Revenue Calculation

```
Revenue Metrics:

Total Revenue:
- Sum of all invoice.total_amount where status IN ('paid', 'partially_paid')
- For date range: filter by invoice_date

Collected Revenue:
- Sum of all payments.amount
- For date range: filter by payment_date

Outstanding Revenue:
- Sum of all invoice.balance_due where balance_due > 0
- Age invoices: 0-30 days, 31-60 days, 61-90 days, >90 days

Revenue by Provider:
- Join encounters to invoices
- Group by provider_id
- Sum revenue per provider

Revenue by Service:
- Join invoice_items to services
- Group by service_id
- Sum revenue per service

Revenue by Insurance:
- Join invoices to insurances
- Group by insurance_id
- Separate patient payments vs insurance payments
```

### 8.2 Appointment Analytics

```
Appointment Metrics:

Total Appointments:
- Count of appointments in date range

Completed Rate:
- (Count of status='completed') / (Total) * 100

No-Show Rate:
- (Count of status='no_show') / (Total scheduled) * 100
- Exclude 'cancelled' from denominator

Cancellation Rate:
- (Count of status='cancelled') / (Total) * 100

Average Wait Time:
- Avg(arrived_at - scheduled_at) for completed appointments

Provider Utilization:
- (Actual appointment hours) / (Available hours) * 100
- Show per provider
```

### 8.3 Patient Analytics

```
Patient Metrics:

Total Patients:
- Count of all active patients

New Patients:
- Count of patients where created_at within date range

Patient Retention:
- Patients with encounter in last 12 months / Total patients * 100

Average Visits per Patient:
- Total encounters / Total patients

Demographics:
- Group by age ranges: 0-17, 18-30, 31-50, 51-70, 70+
- Group by gender
- Group by insurance type
- Visualize with charts
```

---

## 9. COMMUNICATION LOGIC

### 9.1 SMS Sending Logic

```
SMS Sending Workflow:

1. TRIGGER EVENT
   Examples:
   ├─ Appointment reminder
   ├─ Lab results ready
   ├─ Payment reminder
   └─ Manual message from staff

2. CHECK PREREQUISITES
   ├─ Patient has phone number
   ├─ Patient opted-in for SMS (not opted-out)
   └─ SMS credits available (if using paid service)

3. FORMAT MESSAGE
   ├─ Use template for event type
   ├─ Replace placeholders with actual data
   ├─ Ensure message <= 160 chars (or split into multiple)
   └─ Add opt-out instruction (required): "Reply STOP to opt out"

4. SEND VIA PROVIDER
   ├─ Call SMS API (Twilio, etc.)
   ├─ Handle rate limiting
   └─ Await delivery status

5. LOG COMMUNICATION
   ├─ Store in communication_logs table
   ├─ Record: patient_id, type, recipient, message, status
   ├─ Update with delivery status when received
   └─ Track failures for troubleshooting

6. HANDLE ERRORS
   IF error:
   ├─ Log error message
   ├─ Retry up to 3 times with exponential backoff
   ├─ If still fails: mark as failed
   └─ Notify admin of failure
```

### 9.2 Email Sending Logic

```
Email Sending Workflow:

Similar to SMS with differences:

1. FORMAT EMAIL
   ├─ Subject line
   ├─ HTML body (with fallback plain text)
   ├─ Include clinic branding/logo
   ├─ Attachments (if needed): PDF prescriptions, invoices
   └─ Unsubscribe link (required)

2. SEND VIA EMAIL SERVICE
   ├─ SendGrid, Amazon SES, etc.
   ├─ Set from_email, reply_to
   └─ Track opens and clicks (optional)

3. HANDLE BOUNCES
   ├─ Soft bounce: Retry later
   ├─ Hard bounce: Mark email as invalid
   └─ Update patient record if invalid
```

### 9.3 Opt-Out Management

```
Opt-Out Handling:

When patient opts out:
1. Receive opt-out signal:
   ├─ SMS reply "STOP"
   ├─ Email unsubscribe click
   └─ Manual opt-out in patient record

2. Update patient record:
   ├─ Set sms_opt_out = true OR email_opt_out = true
   └─ Record opt_out_date

3. Respect opt-out:
   ├─ Do NOT send marketing messages
   ├─ Still send critical messages:
   │   ├─ Appointment confirmations (if requested)
   │   ├─ Critical lab results
   │   └─ Urgent medical updates
   ├─ Include message: "You requested not to receive messages. This is a critical notification."

4. Allow opt-in again:
   └─ Provide link/method to opt back in
```

---

## 10. USER PERMISSIONS & ACCESS CONTROL

### 10.1 Role-Based Permissions Matrix

```
| Feature                  | Admin | Doctor | Nurse | Reception | Billing |
|--------------------------|-------|--------|-------|-----------|---------|
| View all patients        |   ✓   |   ✓    |   ✓   |     ✓     |    ✓    |
| Create patient           |   ✓   |   ✓    |   ✓   |     ✓     |    -    |
| Edit patient             |   ✓   |   ✓    |   ✓   |     ✓     |    -    |
| Delete patient           |   ✓   |   -    |   -   |     -     |    -    |
| View appointments        |   ✓   |   ✓*   |   ✓*  |     ✓     |    -    |
| Create appointment       |   ✓   |   ✓    |   ✓   |     ✓     |    -    |
| Cancel appointment       |   ✓   |   ✓*   |   -   |     ✓     |    -    |
| View encounters          |   ✓   |   ✓*   |   ✓*  |     -     |    -    |
| Create encounter         |   ✓   |   ✓    |   -   |     -     |    -    |
| Finalize encounter       |   ✓   |   ✓*   |   -   |     -     |    -    |
| Prescribe medications    |   -   |   ✓    |   -   |     -     |    -    |
| Order labs               |   -   |   ✓    |   -   |     -     |    -    |
| Enter lab results        |   ✓   |   ✓    |   ✓   |     -     |    -    |
| View invoices            |   ✓   |   ✓*   |   -   |     ✓     |    ✓    |
| Create invoice           |   ✓   |   -    |   -   |     ✓     |    ✓    |
| Process payment          |   ✓   |   -    |   -   |     ✓     |    ✓    |
| View reports             |   ✓   |   ✓^   |   -   |     -     |    ✓^   |
| Manage inventory         |   ✓   |   -    |   ✓   |     -     |    -    |
| Manage users             |   ✓   |   -    |   -   |     -     |    -    |
| System settings          |   ✓   |   -    |   -   |     -     |    -    |

* = Only for own patients/appointments
^ = Limited reports only
```

### 10.2 Data Access Rules

```
Data Filtering by Role:

DOCTOR:
- Can view/edit own patients and encounters
- Can view other doctors' patients if:
  ├─ Patient explicitly assigned to them
  ├─ Covering for another doctor
  └─ Emergency access (with audit log)

NURSE:
- Can view patients assigned to doctors they support
- Can enter vital signs for any patient
- Cannot finalize encounters

RECEPTIONIST:
- Can view all patients (for scheduling)
- Can view demographic info only
- Cannot view clinical notes or medical history

BILLING:
- Can view billing-related data for all patients
- Cannot view clinical notes
- Can view insurance and payment information

ADMIN:
- Full access to all data
- Super user privileges
- All actions logged in audit trail
```

---

## 11. DATA VALIDATION RULES

### 11.1 Common Validation Rules

```
Email:
- Format: RFC 5322 compliant
- Example: user@domain.com
- Regex: ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$

Phone (Argentina):
- Mobile: +54 9 11 XXXX-XXXX or +54 9 XXX XXX-XXXX
- Landline: +54 11 XXXX-XXXX or +54 XXX XXX-XXXX
- Strip formatting for storage, format for display

DNI:
- 7-8 digits
- Numeric only
- Regex: ^[0-9]{7,8}$

Date:
- Format: DD/MM/YYYY (display) or YYYY-MM-DD (storage)
- Must be valid date
- Min: 01/01/1900
- Max: Current date + 1 year (for appointments)

Currency:
- Positive numbers only (except adjustments)
- Max 2 decimal places
- Format: $X,XXX.XX
- Validation: >= 0, max 999,999,999.99
```

### 11.2 Business-Specific Validations

```
Patient Age:
- If < 18: Require parent/guardian info
- If > 100: Warning (possible data entry error)

Insurance Coverage %:
- Min: 0
- Max: 100
- Default: 100 for certain obra sociales

Appointment Duration:
- Min: 5 minutes
- Max: 480 minutes (8 hours)
- Must be multiple of 5

Prescription Quantity:
- Min: 1
- Max: 9999
- Must be integer
- If > 100: Warning (possible error)

Lab Result Values:
- Must match expected data type (numeric vs text)
- If numeric: check against reference range
- Flag if far outside normal (> 200% or < 50% of range)
```

---

*Business Logic & Workflows Version: 1.0*
*Last Updated: 2025-11-15*
*Comprehensive specification for implementation*
