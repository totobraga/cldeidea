# Clinical Management System - Workflow & Approval Engine

## Overview

This document specifies a comprehensive workflow and approval system for managing multi-step processes, task assignments, notifications, and approval chains.

---

## 1. WORKFLOW ENGINE ARCHITECTURE

### 1.1 Core Concepts

```
Workflow: A series of tasks/steps to complete a process
Task: A single action to be performed
Approval: A decision point requiring authorization
Trigger: An event that starts a workflow
Action: An automated operation
Condition: A rule that determines flow path
```

### 1.2 Database Schema

```sql
-- Workflow definitions (templates)
CREATE TABLE workflow_definitions (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,
    category                VARCHAR(100), -- 'clinical', 'administrative', 'billing'

    -- Configuration
    config                  JSONB NOT NULL,
    /* Example config:
    {
      "trigger": {
        "type": "manual" | "event" | "scheduled",
        "event": "patient_created" | "encounter_completed" | "invoice_pending"
      },
      "steps": [
        {
          "id": "step1",
          "type": "task" | "approval" | "automation",
          "name": "Review patient information",
          "assignee_type": "role" | "user" | "group",
          "assignee_id": 5,
          "due_hours": 24,
          "conditions": [],
          "actions": []
        }
      ]
    }
    */

    -- Triggers
    trigger_type            VARCHAR(50), -- 'manual', 'event', 'scheduled'
    trigger_event           VARCHAR(100), -- Event name if event-triggered

    -- Status
    is_active               BOOLEAN DEFAULT true,
    version                 INTEGER DEFAULT 1,

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_workflow_def_category (category),
    INDEX idx_workflow_def_active (is_active)
);

-- Workflow instances (actual running workflows)
CREATE TABLE workflow_instances (
    id                      BIGSERIAL PRIMARY KEY,
    workflow_definition_id  BIGINT NOT NULL REFERENCES workflow_definitions(id),

    -- Context
    entity_type             VARCHAR(50), -- 'patient', 'encounter', 'invoice'
    entity_id               BIGINT NOT NULL,

    -- Status
    status                  VARCHAR(50) DEFAULT 'pending',
                            -- 'pending', 'in_progress', 'completed', 'cancelled', 'failed'
    current_step            VARCHAR(100), -- Current step ID

    -- Tracking
    started_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at            TIMESTAMP,
    cancelled_at            TIMESTAMP,

    -- Data
    context_data            JSONB, -- Additional data passed through workflow

    started_by              BIGINT REFERENCES users(id),

    INDEX idx_workflow_inst_def (workflow_definition_id),
    INDEX idx_workflow_inst_entity (entity_type, entity_id),
    INDEX idx_workflow_inst_status (status)
);

-- Individual tasks within workflows
CREATE TABLE workflow_tasks (
    id                      BIGSERIAL PRIMARY KEY,
    workflow_instance_id    BIGINT NOT NULL REFERENCES workflow_instances(id) ON DELETE CASCADE,

    -- Task details
    step_id                 VARCHAR(100) NOT NULL, -- From workflow definition
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,
    task_type               VARCHAR(50) NOT NULL, -- 'task', 'approval', 'automation'

    -- Assignment
    assigned_to_user        BIGINT REFERENCES users(id),
    assigned_to_role        VARCHAR(50),
    assigned_to_group       BIGINT,

    -- Priority and timing
    priority                VARCHAR(20) DEFAULT 'normal', -- 'low', 'normal', 'high', 'urgent'
    due_at                  TIMESTAMP,

    -- Status
    status                  VARCHAR(50) DEFAULT 'pending',
                            -- 'pending', 'in_progress', 'completed', 'skipped', 'failed'

    -- Completion
    completed_at            TIMESTAMP,
    completed_by            BIGINT REFERENCES users(id),
    result                  VARCHAR(50), -- 'approved', 'rejected', 'completed'
    result_notes            TEXT,

    -- Data
    task_data               JSONB, -- Task-specific data

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_workflow_tasks_instance (workflow_instance_id),
    INDEX idx_workflow_tasks_assigned_user (assigned_to_user),
    INDEX idx_workflow_tasks_status (status),
    INDEX idx_workflow_tasks_due (due_at)
);

-- Task comments/notes
CREATE TABLE workflow_task_comments (
    id                      BIGSERIAL PRIMARY KEY,
    task_id                 BIGINT NOT NULL REFERENCES workflow_tasks(id) ON DELETE CASCADE,
    comment                 TEXT NOT NULL,
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_task_comments_task (task_id)
);

-- Workflow history/audit
CREATE TABLE workflow_history (
    id                      BIGSERIAL PRIMARY KEY,
    workflow_instance_id    BIGINT NOT NULL REFERENCES workflow_instances(id) ON DELETE CASCADE,
    task_id                 BIGINT REFERENCES workflow_tasks(id),

    event_type              VARCHAR(50) NOT NULL, -- 'started', 'task_assigned', 'task_completed', 'approved', 'rejected'
    event_data              JSONB,

    performed_by            BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_workflow_history_instance (workflow_instance_id),
    INDEX idx_workflow_history_task (task_id)
);
```

---

## 2. PRE-BUILT WORKFLOWS

### 2.1 Patient Onboarding Workflow

```yaml
Name: "New Patient Onboarding"
Trigger: Patient created
Steps:
  1. Task: "Verify patient information"
     Assigned to: Receptionist
     Due: 1 hour
     Required fields:
       - DNI verified
       - Insurance cards scanned
       - Emergency contact confirmed

  2. Task: "Complete medical history questionnaire"
     Assigned to: Patient (via portal) or Nurse
     Due: Before first appointment

  3. Approval: "Review and approve patient file"
     Assigned to: Doctor
     Due: 24 hours

  4. Automation: "Send welcome email with portal access"

  5. Automation: "Schedule orientation call (if new to clinic)"
```

### 2.2 Insurance Prior Authorization Workflow

```yaml
Name: "Insurance Prior Authorization"
Trigger: Service requires authorization
Steps:
  1. Task: "Gather required documentation"
     Assigned to: Billing Staff
     Due: 24 hours
     Documents needed:
       - Medical necessity letter
       - Patient clinical notes
       - Diagnostic results

  2. Task: "Submit authorization request to insurance"
     Assigned to: Billing Staff
     Due: 48 hours

  3. Task: "Follow up on authorization status"
     Assigned to: Billing Staff
     Due: 3 days after submission
     Repeats: Every 2 days until resolved

  4. Approval: "Review authorization response"
     Assigned to: Doctor or Billing Manager
     Possible outcomes:
       - Approved: Continue to scheduling
       - Denied: Start appeal process
       - More info needed: Go back to step 1

  5. Automation: "Notify patient of authorization status"

  6. If approved:
     Automation: "Schedule procedure/service"
```

### 2.3 Lab Results Review Workflow

```yaml
Name: "Lab Results Review and Release"
Trigger: Lab results entered into system
Steps:
  1. Automation: "Flag critical/abnormal results"

  2. Task: "Review lab results"
     Assigned to: Ordering physician
     Due: 24 hours (1 hour if critical)
     Priority: High if critical

  3. Approval: "Approve results for release"
     Assigned to: Ordering physician
     Options:
       - Approve and release
       - Approve with follow-up appointment
       - Request repeat test

  4. Automation: "Notify patient of results availability"
     Via: Patient portal notification + Email/SMS

  5. If abnormal:
     Task: "Schedule follow-up appointment"
     Assigned to: Receptionist or automated
```

### 2.4 Invoice Approval Workflow

```yaml
Name: "Large Invoice Approval"
Trigger: Invoice > $10,000 created
Steps:
  1. Approval: "Review invoice details"
     Assigned to: Billing Manager
     Due: 24 hours

  2. If approved:
     Approval: "Final authorization"
     Assigned to: Clinic Director
     Due: 48 hours

  3. If both approved:
     Automation: "Request CAE from AFIP"
     Automation: "Send invoice to patient"

  4. If rejected at any step:
     Task: "Revise invoice"
     Assigned to: Original creator
```

### 2.5 Prescription Refill Workflow

```yaml
Name: "Prescription Refill Request"
Trigger: Patient requests refill (via portal)
Steps:
  1. Automation: "Check refills remaining"
     If refills available:
       - Auto-approve
       - Generate prescription
       - Notify patient
     If no refills:
       - Continue to step 2

  2. Task: "Review refill request"
     Assigned to: Original prescribing doctor
     Due: 24 hours
     Show:
       - Medication history
       - Last prescription date
       - Patient compliance
       - Recent encounters

  3. Approval: "Approve or deny refill"
     Options:
       - Approve refill
       - Deny (requires appointment)
       - Approve with dosage change

  4. If approved:
     Automation: "Generate new prescription"
     Automation: "Notify patient"

  5. If denied:
     Automation: "Notify patient to schedule appointment"
     Task: "Schedule follow-up appointment"
```

### 2.6 Medical Record Request Workflow

```yaml
Name: "Medical Record Request"
Trigger: Patient or external entity requests records
Steps:
  1. Task: "Verify identity and authorization"
     Assigned to: Medical Records Staff
     Due: 2 business days
     Required:
       - Valid ID verification
       - Signed authorization form
       - Payment (if applicable)

  2. Approval: "Approve record release"
     Assigned to: Clinic Director or Doctor
     Due: 3 business days

  3. Task: "Prepare and redact records"
     Assigned to: Medical Records Staff
     Due: 5 business days
     Actions:
       - Compile requested records
       - Redact sensitive information (if needed)
       - Generate summary

  4. Approval: "Final review before release"
     Assigned to: Doctor or Compliance Officer

  5. Task: "Deliver records"
     Assigned to: Medical Records Staff
     Methods:
       - Secure email
       - Patient portal
       - Physical mail
       - In-person pickup

  6. Automation: "Log record release in audit trail"
```

### 2.7 Staff Onboarding Workflow

```yaml
Name: "New Staff Onboarding"
Trigger: New user created with staff role
Steps:
  1. Task: "Complete HR paperwork"
     Assigned to: HR Manager
     Due: 3 days

  2. Task: "System access setup"
     Assigned to: IT Admin
     Due: 2 days
     Creates:
       - User account
       - Email account
       - Assign roles and permissions

  3. Task: "Workstation setup"
     Assigned to: IT Admin
     Due: Before start date

  4. Task: "Complete orientation training"
     Assigned to: New employee
     Due: First week
     Modules:
       - System navigation
       - Privacy and security (HIPAA equivalent)
       - Clinic policies
       - Role-specific training

  5. Approval: "Training completion verification"
     Assigned to: Direct supervisor

  6. Task: "Schedule one-on-one check-in"
     Assigned to: Direct supervisor
     Due: End of first week
```

---

## 3. WORKFLOW BUILDER UI

### 3.1 Visual Workflow Designer

```
┌──────────────────────────────────────────────────────────────────┐
│ Workflow Builder                               [Save] [Test] [×] │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Workflow Name: [New Patient Onboarding]                         │
│ Category: [Clinical ▼]                                          │
│                                                                  │
│ Trigger: [When patient is created ▼]                            │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Workflow Steps:                                            │ │
│ │                                                            │ │
│ │ START                                                      │ │
│ │   ↓                                                        │ │
│ │ ┌────────────────────────────────────────────┐            │ │
│ │ │ [TASK] Verify patient information          │ [Edit] [×] │ │
│ │ │ Assigned to: Receptionist                  │            │ │
│ │ │ Due: 1 hour                                │            │ │
│ │ └────────────────────────────────────────────┘            │ │
│ │   ↓                                                        │ │
│ │ ┌────────────────────────────────────────────┐            │ │
│ │ │ [TASK] Complete medical history            │ [Edit] [×] │ │
│ │ │ Assigned to: Nurse                         │            │ │
│ │ │ Due: Before first appointment              │            │ │
│ │ └────────────────────────────────────────────┘            │ │
│ │   ↓                                                        │ │
│ │ ┌────────────────────────────────────────────┐            │ │
│ │ │ [APPROVAL] Review patient file             │ [Edit] [×] │ │
│ │ │ Assigned to: Doctor                        │            │ │
│ │ │ Due: 24 hours                              │            │ │
│ │ └────────────────────────────────────────────┘            │ │
│ │   ↓                                                        │ │
│ │ ┌────────────────────────────────────────────┐            │ │
│ │ │ [AUTO] Send welcome email                  │ [Edit] [×] │ │
│ │ └────────────────────────────────────────────┘            │ │
│ │   ↓                                                        │ │
│ │ END                                                        │ │
│ │                                                            │ │
│ │ [+ Add Step]                                               │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Available Step Types:                                            │
│ ┌──────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐            │
│ │ Task │ │ Approval │ │ Automation │ │ Condition│            │
│ └──────┘ └──────────┘ └────────────┘ └──────────┘            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 Step Configuration Modal

```
┌────────────────────────────────────────────┐
│ Configure Task Step                    [×] │
├────────────────────────────────────────────┤
│                                            │
│ Step Name:                                 │
│ [Verify patient information]               │
│                                            │
│ Description:                               │
│ [Confirm all patient details are accurate]│
│                                            │
│ Assign to:                                 │
│ ● Role  ○ User  ○ Group                    │
│ [Receptionist ▼]                           │
│                                            │
│ Due:                                       │
│ [1] [hours ▼] after workflow starts        │
│                                            │
│ Priority:                                  │
│ ○ Low  ● Normal  ○ High  ○ Urgent          │
│                                            │
│ Required Fields/Actions:                   │
│ ☑ DNI verified                             │
│ ☑ Insurance cards scanned                  │
│ ☑ Emergency contact confirmed              │
│ ☐ Address verified                         │
│                                            │
│ Notifications:                             │
│ ☑ Email assignee when task created         │
│ ☑ Reminder 2 hours before due              │
│ ☑ Escalate if overdue by 6 hours           │
│                                            │
│          [Cancel]  [Save Step]             │
└────────────────────────────────────────────┘
```

---

## 4. TASK MANAGEMENT

### 4.1 My Tasks Dashboard

```
┌──────────────────────────────────────────────────────────────────┐
│ My Tasks                                     [Filter ▼] [Sort ▼] │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Overdue (3)                                                      │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ 🔴 HIGH   Review lab results for Juan Perez               │ │
│ │          Workflow: Lab Results Review                      │ │
│ │          Due: 2 hours ago                        [View →] │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ 🟠 NORMAL Approve invoice #1234                            │ │
│ │          Workflow: Large Invoice Approval                  │ │
│ │          Due: 5 hours ago                        [View →] │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Due Today (5)                                                    │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ 🟡 URGENT Review patient file - Maria Garcia              │ │
│ │          Workflow: Patient Onboarding                      │ │
│ │          Due: in 3 hours                         [View →] │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ 🟢 NORMAL Verify insurance authorization                   │ │
│ │          Workflow: Prior Authorization                     │ │
│ │          Due: in 6 hours                         [View →] │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Upcoming (12)                                                    │
│ [Show more...]                                                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Task Detail View

```
┌──────────────────────────────────────────────────────────────────┐
│ ← Back to Tasks        Review Lab Results                   [×] │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Task Details:                                                    │
│ Workflow: Lab Results Review and Release                        │
│ Patient: Juan Perez (DNI: 12345678)                            │
│ Priority: HIGH                                                   │
│ Created: Nov 15, 2025 08:00                                     │
│ Due: Nov 15, 2025 10:00 (2 hours overdue)                      │
│                                                                  │
│ Description:                                                     │
│ Review lab results and approve for release to patient.          │
│ Critical values detected - requires immediate attention.         │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Lab Results:                                               │ │
│ │                                                            │ │
│ │ Test          Result    Reference   Flag                  │ │
│ │ Glucose       350 mg/dL 70-100     🔴 CRITICAL HIGH       │ │
│ │ HbA1c         10.2 %    < 5.7      🔴 HIGH                │ │
│ │ Cholesterol   240 mg/dL < 200      🟠 HIGH                │ │
│ │                                                            │ │
│ │ [View Full Results]                                        │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Actions:                                                         │
│ ☑ Approve and release results                                   │
│ ☑ Schedule follow-up appointment                                │
│ ☐ Request repeat test                                           │
│                                                                  │
│ Notes:                                                           │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Critical glucose level. Patient needs immediate follow-up. │ │
│ │ Schedule within 24 hours.                                  │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Comments (2):                                                    │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Nurse Sofia - Nov 15, 09:00                                │ │
│ │ Patient called, experiencing symptoms. Urgent.             │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ Lab Tech Ana - Nov 15, 08:30                               │ │
│ │ Results confirmed. Sample quality good.                    │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ [Add Comment...]                                                 │
│                                                                  │
│             [Reject]  [Request More Info]  [Approve]            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. APPROVAL CHAINS

### 5.1 Multi-Level Approvals

```sql
CREATE TABLE approval_chains (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Configuration
    chain_config            JSONB NOT NULL,
    /* Example:
    {
      "levels": [
        {
          "level": 1,
          "name": "Manager Approval",
          "approvers": {
            "type": "role",
            "value": "manager"
          },
          "required_approvals": 1,
          "can_reject": true
        },
        {
          "level": 2,
          "name": "Director Approval",
          "approvers": {
            "type": "user",
            "value": [15, 16]
          },
          "required_approvals": 1,
          "can_reject": true
        }
      ],
      "parallel": false  // Sequential or parallel approvals
    }
    */

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE approval_requests (
    id                      BIGSERIAL PRIMARY KEY,
    approval_chain_id       BIGINT REFERENCES approval_chains(id),
    workflow_task_id        BIGINT REFERENCES workflow_tasks(id),

    -- Context
    entity_type             VARCHAR(50),
    entity_id               BIGINT,

    -- Status
    status                  VARCHAR(50) DEFAULT 'pending',
                            -- 'pending', 'approved', 'rejected', 'cancelled'
    current_level           INTEGER DEFAULT 1,

    -- Data
    request_data            JSONB,

    requested_by            BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at            TIMESTAMP
);

CREATE TABLE approval_responses (
    id                      BIGSERIAL PRIMARY KEY,
    approval_request_id     BIGINT NOT NULL REFERENCES approval_requests(id),

    level                   INTEGER NOT NULL,
    approver_id             BIGINT NOT NULL REFERENCES users(id),

    response                VARCHAR(20) NOT NULL, -- 'approved', 'rejected'
    comments                TEXT,

    responded_at            TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_approval_resp_request (approval_request_id),
    INDEX idx_approval_resp_approver (approver_id)
);
```

### 5.2 Conditional Approvals

```javascript
// Example: Amount-based approval routing
function determineApprovalChain(invoiceAmount) {
  if (invoiceAmount < 5000) {
    return null; // No approval needed
  } else if (invoiceAmount < 10000) {
    return 'manager_approval'; // Single level
  } else if (invoiceAmount < 50000) {
    return 'manager_director_approval'; // Two levels
  } else {
    return 'full_approval_chain'; // All levels including CEO
  }
}
```

---

## 6. NOTIFICATIONS & ESCALATIONS

### 6.1 Notification Rules

```sql
CREATE TABLE notification_rules (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,

    -- Trigger conditions
    trigger_type            VARCHAR(50) NOT NULL,
                            -- 'task_assigned', 'task_due', 'task_overdue', 'approval_pending'

    -- Timing
    timing_offset           INTEGER, -- Minutes before/after trigger
    timing_type             VARCHAR(20), -- 'before', 'after', 'at'

    -- Recipients
    recipient_type          VARCHAR(50), -- 'assignee', 'role', 'user', 'creator'
    recipient_value         TEXT,

    -- Channels
    send_email              BOOLEAN DEFAULT true,
    send_sms                BOOLEAN DEFAULT false,
    send_push               BOOLEAN DEFAULT true,
    send_in_app             BOOLEAN DEFAULT true,

    -- Template
    email_template_id       BIGINT,
    sms_template_id         BIGINT,

    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 6.2 Escalation Rules

```sql
CREATE TABLE escalation_rules (
    id                      BIGSERIAL PRIMARY KEY,
    workflow_definition_id  BIGINT REFERENCES workflow_definitions(id),

    -- Conditions
    trigger_condition       VARCHAR(100), -- 'task_overdue', 'approval_pending'
    overdue_threshold       INTEGER, -- Minutes overdue

    -- Escalation actions
    escalate_to_type        VARCHAR(50), -- 'supervisor', 'role', 'user'
    escalate_to_value       TEXT,

    -- Notifications
    notify_original_assignee BOOLEAN DEFAULT true,
    notify_escalation_target BOOLEAN DEFAULT true,
    notification_message    TEXT,

    is_active               BOOLEAN DEFAULT true,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 6.3 Example Escalation Flow

```
Task assigned to Receptionist:
- Due: 1 hour from creation
- Overdue by 30 min: Send reminder to Receptionist
- Overdue by 1 hour: Escalate to Reception Supervisor
- Overdue by 2 hours: Escalate to Clinic Manager
- Overdue by 4 hours: Escalate to Clinic Director + Alert on dashboard
```

---

## 7. WORKFLOW ANALYTICS

### 7.1 Workflow Performance Metrics

```sql
CREATE VIEW workflow_performance AS
SELECT
    wd.name AS workflow_name,
    wi.status,
    COUNT(*) AS instance_count,
    AVG(EXTRACT(EPOCH FROM (wi.completed_at - wi.started_at))/3600) AS avg_duration_hours,
    MIN(wi.started_at) AS first_instance,
    MAX(wi.started_at) AS last_instance
FROM workflow_instances wi
JOIN workflow_definitions wd ON wi.workflow_definition_id = wd.id
GROUP BY wd.id, wd.name, wi.status;
```

### 7.2 Workflow Reports

**Workflow Efficiency Report:**
- Average completion time per workflow
- Bottleneck identification (slowest steps)
- Completion rate (completed vs abandoned)
- On-time completion rate

**Task Performance Report:**
- Tasks by assignee
- Average completion time
- Overdue rate
- Most common delays

**Approval Analytics:**
- Approval time by approver
- Approval vs rejection rate
- Approval chain efficiency

---

## 8. INTEGRATION WITH EXISTING MODULES

### 8.1 Triggering Workflows from Events

```javascript
// Example: Trigger workflow when patient created
async function createPatient(patientData) {
  // Create patient
  const patient = await Patient.create(patientData);

  // Trigger workflow
  await workflowEngine.trigger('patient_created', {
    entity_type: 'patient',
    entity_id: patient.id,
    data: patient
  });

  return patient;
}

// Example: Trigger workflow when invoice exceeds threshold
async function createInvoice(invoiceData) {
  const invoice = await Invoice.create(invoiceData);

  if (invoice.total_amount > 10000) {
    await workflowEngine.trigger('large_invoice_created', {
      entity_type: 'invoice',
      entity_id: invoice.id,
      data: invoice
    });
  }

  return invoice;
}
```

### 8.2 Workflow Actions

```javascript
// Automated actions workflows can perform
const workflowActions = {
  // Send notifications
  sendEmail: async (to, template, data) => {
    await emailService.send({ to, template, data });
  },

  sendSMS: async (phone, message) => {
    await smsService.send({ phone, message });
  },

  // Create records
  createAppointment: async (patientId, providerId, dateTime) => {
    await Appointment.create({ patientId, providerId, scheduledAt: dateTime });
  },

  // Update records
  updatePatientStatus: async (patientId, status) => {
    await Patient.update({ status }, { where: { id: patientId } });
  },

  // Generate documents
  generatePrescription: async (encounterId, medications) => {
    await prescriptionService.generate(encounterId, medications);
  },

  // Call external APIs
  submitInsuranceClaim: async (claimData) => {
    await insuranceAPI.submitClaim(claimData);
  }
};
```

---

## 9. WORKFLOW TEMPLATES LIBRARY

### 9.1 Clinical Workflows

1. **New Patient Onboarding** (defined above)
2. **Lab Results Review** (defined above)
3. **Prescription Refill** (defined above)
4. **Referral Management**
5. **Test Results Follow-up**
6. **Immunization Tracking**
7. **Chronic Disease Management Protocol**

### 9.2 Administrative Workflows

1. **Insurance Prior Authorization** (defined above)
2. **Medical Record Request** (defined above)
3. **Staff Onboarding** (defined above)
4. **Equipment Maintenance**
5. **Inventory Restock**
6. **Vendor Approval**

### 9.3 Billing Workflows

1. **Large Invoice Approval** (defined above)
2. **Payment Plan Setup**
3. **Collection Process**
4. **Insurance Claim Appeal**
5. **Refund Request**

---

## 10. API ENDPOINTS

### 10.1 Workflow Management

```
POST   /api/workflows/definitions              Create workflow template
GET    /api/workflows/definitions              List workflow templates
GET    /api/workflows/definitions/:id          Get workflow template
PUT    /api/workflows/definitions/:id          Update workflow template
DELETE /api/workflows/definitions/:id          Delete workflow template

POST   /api/workflows/instances                Start workflow instance
GET    /api/workflows/instances                List workflow instances
GET    /api/workflows/instances/:id            Get workflow instance details
PUT    /api/workflows/instances/:id/cancel     Cancel workflow instance
```

### 10.2 Task Management

```
GET    /api/tasks                              Get my tasks
GET    /api/tasks/:id                          Get task details
PUT    /api/tasks/:id/claim                    Claim task (if role-assigned)
PUT    /api/tasks/:id/complete                 Complete task
PUT    /api/tasks/:id/reject                   Reject task
POST   /api/tasks/:id/comments                 Add comment to task
GET    /api/tasks/:id/history                  Get task history
```

### 10.3 Approvals

```
GET    /api/approvals/pending                  Get pending approvals
POST   /api/approvals/:id/approve              Approve request
POST   /api/approvals/:id/reject               Reject request
GET    /api/approvals/:id/history              Get approval history
```

---

*Workflow & Approval Engine Specification Version: 1.0*
*Last Updated: 2025-11-15*
*Comprehensive workflow automation system*
