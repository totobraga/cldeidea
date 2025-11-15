# Clinical Management System - UI/UX Specifications

## Design Principles

### User Experience Goals
- **Efficiency:** Minimize clicks and time to complete common tasks
- **Clarity:** Clear information hierarchy and visual feedback
- **Consistency:** Uniform patterns across all modules
- **Accessibility:** WCAG 2.1 AA compliance
- **Responsiveness:** Works on desktop, tablet, and mobile
- **Speed:** Fast loading times, optimistic UI updates

### Visual Design Principles
- Clean, modern, professional medical aesthetic
- High contrast for readability
- Color-coded status indicators
- Generous whitespace
- Clear typography hierarchy
- Consistent iconography

---

## Global Navigation & Layout

### Main Navigation Structure
```
┌─────────────────────────────────────────────────────────┐
│  [Logo] Clinical Management System    [User] [Logout]   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────┬──────────────────────────────────────┐   │
│  │          │                                       │   │
│  │ Sidebar  │        Main Content Area             │   │
│  │  Nav     │                                       │   │
│  │          │                                       │   │
│  │  • Home  │                                       │   │
│  │  • Pax   │                                       │   │
│  │  • Citas │                                       │   │
│  │  • HC    │                                       │   │
│  │  • Rx    │                                       │   │
│  │  • Lab   │                                       │   │
│  │  • €€€   │                                       │   │
│  │          │                                       │   │
│  └──────────┴──────────────────────────────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Sidebar Navigation (Collapsible)
Primary navigation items:
- **Dashboard** (Inicio) - Icon: 📊
- **Patients** (Pacientes) - Icon: 👥
- **Appointments** (Turnos) - Icon: 📅
- **Clinical** (Historia Clínica) - Icon: 📋
- **Prescriptions** (Recetas) - Icon: 💊
- **Laboratory** (Laboratorio) - Icon: 🔬
- **Billing** (Facturación) - Icon: 💰
- **Inventory** (Inventario) - Icon: 📦
- **Reports** (Reportes) - Icon: 📈
- **Communications** (Comunicaciones) - Icon: 💬
- **Settings** (Configuración) - Icon: ⚙️

**Collapsed State:** Show only icons
**Expanded State:** Show icons + labels
**Active Item:** Highlighted with accent color and left border

### Top Bar (Header)
- **Left:** Logo and clinic name
- **Center:** Search bar (global patient/appointment search)
- **Right:**
  - Notifications bell with badge count
  - Quick actions menu (+ button)
  - User profile dropdown
  - Logout button

### Responsive Breakpoints
- **Desktop:** > 1200px (sidebar always visible)
- **Tablet:** 768px - 1199px (sidebar collapsible)
- **Mobile:** < 768px (hamburger menu, bottom nav for key items)

---

## Color System

### Primary Colors
- **Primary:** #2563eb (Blue) - Main actions, links
- **Secondary:** #10b981 (Green) - Success, confirmations
- **Accent:** #f59e0b (Amber) - Warnings, important info

### Status Colors
- **Success:** #10b981 (Green) - Completed, paid, active
- **Warning:** #f59e0b (Amber) - Pending, requires attention
- **Error:** #ef4444 (Red) - Cancelled, failed, critical
- **Info:** #3b82f6 (Blue) - Informational messages
- **Scheduled:** #8b5cf6 (Purple) - Scheduled appointments
- **In Progress:** #06b6d4 (Cyan) - Active consultations

### Neutral Colors
- **Background:** #f9fafb (Very light gray)
- **Surface:** #ffffff (White)
- **Border:** #e5e7eb (Light gray)
- **Text Primary:** #111827 (Almost black)
- **Text Secondary:** #6b7280 (Medium gray)
- **Text Disabled:** #9ca3af (Light gray)

### Appointment Status Colors
- Scheduled: #8b5cf6 (Purple)
- Confirmed: #3b82f6 (Blue)
- Arrived: #06b6d4 (Cyan)
- In Progress: #f59e0b (Amber)
- Completed: #10b981 (Green)
- Cancelled: #ef4444 (Red)
- No Show: #64748b (Slate)

---

## Typography

### Font Family
- **Primary:** Inter, system-ui, -apple-system, sans-serif
- **Monospace:** 'Fira Code', monospace (for codes, IDs)

### Type Scale
- **H1:** 2.5rem (40px) / Bold / Page titles
- **H2:** 2rem (32px) / Bold / Section headers
- **H3:** 1.5rem (24px) / Semibold / Card headers
- **H4:** 1.25rem (20px) / Semibold / Subsections
- **Body Large:** 1.125rem (18px) / Regular / Important text
- **Body:** 1rem (16px) / Regular / Default text
- **Body Small:** 0.875rem (14px) / Regular / Secondary text
- **Caption:** 0.75rem (12px) / Regular / Labels, hints

---

## Common Components

### Buttons

#### Primary Button
```
┌──────────────────┐
│  [Icon] Label    │  ← Filled with primary color
└──────────────────┘
States: Default, Hover, Active, Disabled
```

#### Secondary Button
```
┌──────────────────┐
│  [Icon] Label    │  ← Outline style
└──────────────────┘
```

#### Sizes
- **Large:** 48px height, 1rem (16px) text
- **Medium:** 40px height, 0.875rem (14px) text
- **Small:** 32px height, 0.875rem (14px) text

### Input Fields

#### Text Input
```
Label *
┌────────────────────────────────────┐
│ Placeholder text...                │
└────────────────────────────────────┘
Helper text or error message
```

- Required fields: Red asterisk (*)
- Error state: Red border, red helper text
- Success state: Green border (optional)
- Disabled: Gray background, no interaction

#### Select Dropdown
```
Label
┌────────────────────────────────┬─┐
│ Selected value                 │▼│
└────────────────────────────────┴─┘
```

- Searchable for > 10 options
- Multi-select with chips/badges

#### Date Picker
```
Label
┌─────────────────────┬──┬──┐
│ DD/MM/YYYY          │📅│x │
└─────────────────────┴──┴──┘
```
- Calendar popup on click
- Clear button (x)
- Format: DD/MM/YYYY (Argentina standard)

#### Time Picker
```
┌─────┬──┬─────┐
│ HH  │: │ MM  │
└─────┴──┴─────┘
```
- 24-hour format
- Dropdown or spinner

### Tables/Data Grids

```
┌─────────────────────────────────────────────────────────────┐
│ Table Title                    [Search] [Filter] [+ Action] │
├─────┬──────────┬────────┬──────────┬──────────┬────────────┤
│ ☑   │ Column 1 │ Col 2  │ Column 3 │ Status   │ Actions    │
├─────┼──────────┼────────┼──────────┼──────────┼────────────┤
│ ☑   │ Data     │ Data   │ Data     │ [Badge]  │ [•••]      │
│ ☐   │ Data     │ Data   │ Data     │ [Badge]  │ [•••]      │
│ ☐   │ Data     │ Data   │ Data     │ [Badge]  │ [•••]      │
└─────┴──────────┴────────┴──────────┴──────────┴────────────┘
       Showing 1-20 of 150       [← 1 2 3 ... 8 →]
```

Features:
- Sortable columns (click header)
- Row selection (checkboxes)
- Bulk actions
- Inline actions (kebab menu •••)
- Pagination at bottom
- Empty state with illustration
- Loading skeleton

### Cards

```
┌──────────────────────────────────────┐
│ Card Header               [Action]   │
├──────────────────────────────────────┤
│                                      │
│  Card content area                   │
│  • Information                       │
│  • Data visualization                │
│  • Forms                             │
│                                      │
└──────────────────────────────────────┘
```

### Modal Dialogs

```
        ┌─────────────────────────────────┐
        │ Modal Title                  [X]│
        ├─────────────────────────────────┤
        │                                 │
        │  Modal content                  │
        │                                 │
        ├─────────────────────────────────┤
        │          [Cancel]  [Confirm]    │
        └─────────────────────────────────┘
```

Sizes:
- **Small:** 400px width
- **Medium:** 600px width
- **Large:** 800px width
- **Full:** 90% viewport

### Status Badges

```
[Scheduled]  [Confirmed]  [Completed]  [Cancelled]
```
- Rounded corners
- Uppercase text
- Color-coded by status
- Small size, inline with text

### Toast Notifications

```
┌────────────────────────────────────────┐
│ ✓ Success message here                 │ (Top-right corner)
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ ⚠ Warning message here                 │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ ✗ Error message here                   │
└────────────────────────────────────────┘
```

- Auto-dismiss after 5 seconds
- Manual close button
- Stack vertically
- Slide in animation

---

## Module-Specific UI Specifications

### 1. DASHBOARD

#### Layout
```
┌──────────────────────────────────────────────────────────┐
│ Dashboard                                  [Date Range ▼]│
├──────────────────────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│ │ Total    │ │ Today's  │ │ Pending  │ │ Revenue  │    │
│ │ Patients │ │ Appts    │ │ Claims   │ │ Today    │    │
│ │   1,250  │ │    24    │ │    15    │ │ $12,500  │    │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
├──────────────────────────────────────────────────────────┤
│ ┌─────────────────────────┐ ┌──────────────────────────┐│
│ │ Today's Schedule        │ │ Recent Activities        ││
│ │                         │ │                          ││
│ │ 09:00 - Juan P. (...)   │ │ • Patient check-in       ││
│ │ 09:30 - Maria G. (...)  │ │ • Lab result uploaded    ││
│ │ 10:00 - Carlos R. (...) │ │ • Invoice paid           ││
│ │                         │ │                          ││
│ └─────────────────────────┘ └──────────────────────────┘│
├──────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────┐ │
│ │ Revenue Chart (Last 30 Days)                        │ │
│ │ [Line/Bar Chart]                                    │ │
│ └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

#### KPI Cards (4 across)
- Large number (H1)
- Label below (Body Small)
- Icon on left
- Color-coded border/background
- Trend indicator (↑ 12% from last week)
- Click to view details

#### Today's Schedule Widget
- Time slot list
- Patient name + appointment type
- Click to view details
- Color-coded by status
- "View All" link at bottom

#### Recent Activities Feed
- Chronological list
- Icon + description + timestamp
- Last 10 activities
- Real-time updates (websocket)

---

### 2. PATIENT MODULE

#### Patient List View
```
┌──────────────────────────────────────────────────────────┐
│ Patients                                  [+ New Patient]│
├──────────────────────────────────────────────────────────┤
│ [Search by name, DNI, phone...]          [Filters ▼]    │
├───┬────────┬─────────┬────────┬───────────┬──────────────┤
│ ☑ │ ID     │ Name    │ DNI    │ Insurance │ Actions      │
├───┼────────┼─────────┼────────┼───────────┼──────────────┤
│ ☐ │ P-0001 │ Juan P. │ 301... │ OSDE      │ View Edit •••│
│ ☐ │ P-0002 │ Maria G.│ 285... │ Swiss Med │ View Edit •••│
│ ☐ │ P-0003 │ Carlos  │ 412... │ PAMI      │ View Edit •••│
└───┴────────┴─────────┴────────┴───────────┴──────────────┘
```

**Features:**
- Global search (instant, as-you-type)
- Filters: Insurance, Age Range, Active/Inactive
- Sortable columns
- Quick actions: View, Edit, Schedule Appointment
- Bulk actions: Export, Print, Send Message

#### Patient Detail View
```
┌──────────────────────────────────────────────────────────┐
│ ← Back to List                                [Edit] [•••]│
├──────────────────────────────────────────────────────────┤
│ [Photo]  Carlos Rodriguez, 45 años                       │
│          DNI: 30123456  |  O+ Blood Type                 │
│          📞 +54 9 11 1234-5678  |  📧 carlos@email.com  │
├──────────────────────────────────────────────────────────┤
│ [Demographics] [Insurance] [Medical History] [Timeline]  │ ← Tabs
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Active Tab Content                                      │
│                                                          │
│  Demographics Tab:                                       │
│  • Personal Information (editable inline)                │
│  • Contact Information                                   │
│  • Emergency Contact                                     │
│                                                          │
│  Insurance Tab:                                          │
│  • List of insurances (primary highlighted)              │
│  • Add insurance button                                  │
│                                                          │
│  Medical History Tab:                                    │
│  • Allergies (red badges)                                │
│  • Chronic conditions                                    │
│  • Previous surgeries                                    │
│  • Family history                                        │
│  • Immunizations                                         │
│                                                          │
│  Timeline Tab:                                           │
│  • Chronological list of encounters, prescriptions, etc. │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Right Sidebar (Sticky):**
```
┌──────────────────────────┐
│ Quick Actions            │
├──────────────────────────┤
│ [Schedule Appointment]   │
│ [New Encounter]          │
│ [New Prescription]       │
│ [Order Labs]             │
│ [Send Message]           │
├──────────────────────────┤
│ Upcoming Appointments    │
│ • Nov 15, 10:00 AM       │
│ • Dec 1, 2:00 PM         │
└──────────────────────────┘
```

#### Patient Registration Form
```
┌──────────────────────────────────────────────────────────┐
│ New Patient Registration                    [Save] [Cancel]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Step 1 of 3: Personal Information              ●○○      │
│                                                          │
│ First Name *          Last Name *                        │
│ [____________]        [____________]                     │
│                                                          │
│ DNI/Passport *        Date of Birth *      Gender *      │
│ [____________]        [DD/MM/YYYY]         [Select ▼]    │
│                                                          │
│ Blood Type            Marital Status                     │
│ [Select ▼]            [Select ▼]                         │
│                                                          │
│                              [Next: Contact Info →]      │
└──────────────────────────────────────────────────────────┘
```

**Multi-step wizard:**
1. Personal Information
2. Contact & Emergency
3. Insurance & Medical History

**Progress indicator** at top
**Validation** on each step before proceeding

---

### 3. APPOINTMENT MODULE

#### Calendar View
```
┌──────────────────────────────────────────────────────────┐
│ Appointments                              [+ New Appointment]│
├──────────────────────────────────────────────────────────┤
│ [Day] [Week] [Month]    ← Nov 15, 2025 →   [Dr. Martinez ▼]│
├──────────────────────────────────────────────────────────┤
│        Mon      Tue      Wed      Thu      Fri      Sat  │
│ 08:00  ─────────────────────────────────────────────────│
│ 09:00  [Juan P.][Maria G.]      [Carlos R.]             │
│        30 min   30 min           30 min                  │
│ 10:00  [Empty]  [Sofia L.]      [Empty]                 │
│ 11:00  ─────────────────────────────────────────────────│
│ 12:00           LUNCH BREAK                              │
│ 13:00  ─────────────────────────────────────────────────│
│ 14:00  [Pedro M.][Ana S.]       [Luis G.]               │
│ 15:00  ─────────────────────────────────────────────────│
└──────────────────────────────────────────────────────────┘
```

**Features:**
- Drag & drop to reschedule
- Click time slot to create appointment
- Click appointment to view/edit
- Color-coded by status
- Provider filter dropdown
- Show blocked time (lunch, meetings)
- Today button to jump to current date

#### Appointment Card (on click)
```
┌────────────────────────────────┐
│ Appointment Details        [X] │
├────────────────────────────────┤
│ Patient: Carlos Rodriguez      │
│ Date: Nov 15, 2025             │
│ Time: 09:00 - 09:30            │
│ Type: Consulta nueva           │
│ Status: [Scheduled ▼]          │
│                                │
│ Chief Complaint:               │
│ Control de presión arterial    │
│                                │
├────────────────────────────────┤
│ [Check In] [Reschedule] [•••]  │
└────────────────────────────────┘
```

#### New Appointment Form
```
┌──────────────────────────────────────────────────────────┐
│ New Appointment                           [Save] [Cancel]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Patient *                                                │
│ [Search patient by name or DNI...]                      │
│                                                          │
│ Provider *                    Date *        Time *       │
│ [Dr. Martinez ▼]              [DD/MM/YYYY]  [HH:MM]      │
│                                                          │
│ Appointment Type *            Duration                   │
│ [Consulta nueva ▼]            [30 minutes ▼]             │
│                                                          │
│ Chief Complaint                                          │
│ [_____________________________]                          │
│                                                          │
│ Notes                                                    │
│ [_____________________________]                          │
│ [_____________________________]                          │
│                                                          │
│ ☐ Send confirmation SMS/Email                            │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Auto-complete** for patient search
**Availability checking** when selecting date/time
**Conflict warning** if slot already booked

#### Waitlist View (Tab)
```
┌──────────────────────────────────────────────────────────┐
│ [Calendar] [List] [Waitlist]                             │
├──────────────────────────────────────────────────────────┤
│ Patients Waiting for Appointments                        │
├──────────────────────────────────────────────────────────┤
│ Patient Name    Preferred Date  Preferred Time  Actions  │
│ Juan Perez      Nov 15-20       Morning         [Schedule]│
│ Sofia Lopez     Nov 16          Afternoon       [Schedule]│
│ Maria Garcia    Flexible        Any             [Schedule]│
└──────────────────────────────────────────────────────────┘
```

---

### 4. CLINICAL / ENCOUNTERS MODULE

#### Encounter Form (During Consultation)
```
┌──────────────────────────────────────────────────────────┐
│ Clinical Encounter - Carlos Rodriguez, 45 años      [Save]│
├──────────────────────────────────────────────────────────┤
│ [SOAP Notes] [Vital Signs] [Diagnoses] [Orders]          │← Tabs
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Subjective (S)                                           │
│ ┌──────────────────────────────────────────────────────┐│
│ │ Chief Complaint:                                     ││
│ │ [Dolor de cabeza desde hace 3 días...]              ││
│ │                                                      ││
│ │ History of Present Illness:                          ││
│ │ [Describe symptoms...]                               ││
│ └──────────────────────────────────────────────────────┘│
│                                                          │
│ Objective (O)                                            │
│ ┌──────────────────────────────────────────────────────┐│
│ │ Vital Signs: BP 120/80, HR 72, Temp 36.5°C          ││
│ │ [Edit Vital Signs]                                   ││
│ │                                                      ││
│ │ Physical Examination:                                ││
│ │ [Findings from examination...]                       ││
│ └──────────────────────────────────────────────────────┘│
│                                                          │
│ Assessment (A)                                           │
│ ┌──────────────────────────────────────────────────────┐│
│ │ [Cefalea tensional...]                               ││
│ └──────────────────────────────────────────────────────┘│
│                                                          │
│ Plan (P)                                                 │
│ ┌──────────────────────────────────────────────────────┐│
│ │ [Treatment plan...]                                  ││
│ └──────────────────────────────────────────────────────┘│
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Auto-save** every 30 seconds
**Templates** dropdown for common encounters
**Voice-to-text** button (optional)
**Attach images/documents** button

#### Vital Signs Tab
```
┌──────────────────────────────────────────────────────────┐
│ Vital Signs                                     [Add New]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Blood Pressure    Heart Rate    Temperature    SpO2     │
│ [120]/[80] mmHg   [72] bpm      [36.5] °C      [98] %   │
│                                                          │
│ Weight           Height         BMI (calculated)         │
│ [75.0] kg        [170] cm       [25.95]                 │
│                                                          │
│ Pain Scale (0-10)                                        │
│ ○ ○ ○ ○ ○ ● ○ ○ ○ ○ ○                                   │
│ 0           5            10                              │
│                                                          │
│ ┌──────────────────────────────────────────────────────┐│
│ │ Vital Signs History (Graph)                          ││
│ │ [Line chart showing trends]                          ││
│ └──────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
```

**BMI** auto-calculated from height/weight
**Flagging** for abnormal values
**Historical trends** graph below

#### Diagnoses Tab
```
┌──────────────────────────────────────────────────────────┐
│ Diagnoses                                    [Add Diagnosis]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Active Diagnoses:                                        │
│ ┌──────────────────────────────────────────────────────┐│
│ │ • Hypertension (I10) - Since 2020-01-15  [Primary]  ││
│ │ • Type 2 Diabetes (E11.9) - Since 2019-06-01        ││
│ └──────────────────────────────────────────────────────┘│
│                                                          │
│ Add New Diagnosis:                                       │
│ ICD-10 Code / Description                                │
│ [Search ICD-10...]                                       │
│                                                          │
│ Type:  ○ Primary  ○ Secondary                            │
│ Status: ● Active  ○ Resolved  ○ Chronic                  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**ICD-10 search** with autocomplete
**Type selection** (primary/secondary)
**Status tracking** (active/resolved/chronic)

---

### 5. PRESCRIPTION MODULE

#### Prescription Creation
```
┌──────────────────────────────────────────────────────────┐
│ New Prescription - Carlos Rodriguez         [Save] [Print]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Medications:                                             │
│                                                          │
│ 1. ┌────────────────────────────────────────────────┐   │
│    │ Medication Name *                              │   │
│    │ [Search medications...]  → Ibuprofeno 400mg    │   │
│    │                                                │   │
│    │ Dosage        Form          Route              │   │
│    │ [400mg]       [Comprimido]  [Oral ▼]           │   │
│    │                                                │   │
│    │ Frequency           Duration       Quantity    │   │
│    │ [Cada 8 horas ▼]    [7 días]      [21]        │   │
│    │                                                │   │
│    │ Instructions:                                  │   │
│    │ [Tomar con alimentos]                          │   │
│    │                                                │   │
│    │ ⚠ Allergy Warning: None                        │   │
│    │ ℹ Interactions: None detected                  │   │
│    └────────────────────────────────────────────────┘   │
│                                                          │
│ [+ Add Another Medication]                               │
│                                                          │
│ ☐ Controlled Substance (Requires special prescription)   │
│ ☐ Chronic Medication (For long-term use)                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Drug search** with autocomplete from medication database
**Allergy checking** against patient allergies
**Interaction checking** between multiple medications
**Warnings** displayed prominently

#### Prescription Preview
```
┌──────────────────────────────────────────────────────────┐
│                    PRESCRIPTION                          │
│                                                          │
│ Dr. Juan Martinez                    Date: 15/11/2025   │
│ Matricula: MN 12345                  Rx #: RX-00123     │
│                                                          │
│ Patient: Carlos Rodriguez                                │
│ DNI: 30123456                        Age: 45 years      │
│                                                          │
│ ───────────────────────────────────────────────────────  │
│                                                          │
│ Rp/                                                      │
│                                                          │
│ 1. Ibuprofeno 400mg comprimidos                          │
│    Tomar 1 comprimido cada 8 horas por 7 días           │
│    Con alimentos                                         │
│    Cantidad: 21 comprimidos                              │
│                                                          │
│ ───────────────────────────────────────────────────────  │
│                                                          │
│                                    _____________________  │
│                                    Firma del Médico      │
│                                                          │
│        [Print]  [Email to Patient]  [Download PDF]       │
└──────────────────────────────────────────────────────────┘
```

---

### 6. LABORATORY MODULE

#### Lab Order Form
```
┌──────────────────────────────────────────────────────────┐
│ New Lab Order - Carlos Rodriguez            [Save] [Print]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Select Tests:                                            │
│                                                          │
│ [Search tests...] or select from common panels:          │
│                                                          │
│ Quick Panels:                                            │
│ [Hemograma Completo] [Perfil Lipídico] [Hepatograma]    │
│ [Glucemia] [Función Renal] [Perfil Tiroideo]            │
│                                                          │
│ Selected Tests:                                          │
│ ☑ Hemoglobina                                            │
│ ☑ Hematocrito                                            │
│ ☑ Glóbulos blancos                                       │
│ ☑ Plaquetas                                              │
│                                                          │
│ ☐ Urgent/STAT                                            │
│ ☑ Fasting Required (12 hours)                            │
│                                                          │
│ Special Instructions:                                    │
│ [_____________________________]                          │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Common panels** as quick-select buttons
**Individual tests** with checkboxes
**Requirements** clearly indicated

#### Lab Results Entry
```
┌──────────────────────────────────────────────────────────┐
│ Lab Results - Order #LAB-00123                      [Save]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Test Name         Result  Unit    Reference   Flag      │
│ ───────────────────────────────────────────────────────  │
│ Hemoglobina       14.5    g/dL    12-16       ✓ Normal  │
│ Hematocrito       42      %       37-47       ✓ Normal  │
│ Glóbulos blancos  12.5    10³/μL  4-11        ⚠ High    │
│ Plaquetas         250     10³/μL  150-400     ✓ Normal  │
│                                                          │
│ Notes:                                                   │
│ [_____________________________]                          │
│                                                          │
│ ☑ Notify patient of results                              │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Automatic flagging** of abnormal values
**Color coding:** Green (normal), Yellow (borderline), Red (critical)
**Reference ranges** displayed for context

---

### 7. BILLING MODULE

#### Invoice List
```
┌──────────────────────────────────────────────────────────┐
│ Invoices                                  [+ New Invoice] │
├──────────────────────────────────────────────────────────┤
│ [Search...]  Status:[All ▼]  Date Range:[Last 30 days ▼]│
├─────┬────────────┬──────────┬──────────┬─────────┬───────┤
│ Inv │ Date       │ Patient  │ Amount   │ Status  │ Action│
├─────┼────────────┼──────────┼──────────┼─────────┼───────┤
│ 0001│ 15/11/2025 │ Carlos R.│ $5,000   │ [Paid]  │ View  │
│ 0002│ 15/11/2025 │ Maria G. │ $3,500   │ [Pending]│ View │
│ 0003│ 14/11/2025 │ Juan P.  │ $4,200   │ [Partial]│ View │
└─────┴────────────┴──────────┴──────────┴─────────┴───────┘
```

Status badges:
- **Paid** (Green)
- **Pending** (Yellow)
- **Partial** (Amber)
- **Overdue** (Red)

#### Invoice Detail / Creation
```
┌──────────────────────────────────────────────────────────┐
│ New Invoice                        [Save Draft] [Finalize]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Patient *                    Invoice Type *              │
│ [Carlos Rodriguez ▼]         ○ A  ● B  ○ C              │
│                                                          │
│ Insurance                    Member Number               │
│ [OSDE ▼]                     [123456789]                 │
│                                                          │
│ ───────────────────────────────────────────────────────  │
│                                                          │
│ Line Items:                                              │
│                                                          │
│ Service          Qty  Unit Price  Tax    Total          │
│ Consulta médica   1   $5,000     $0     $5,000          │
│ [+ Add Item]                                             │
│                                                          │
│ ───────────────────────────────────────────────────────  │
│                                              Subtotal: $5,000│
│                                              IVA (21%): $0   │
│                                              TOTAL:     $5,000│
│                                                          │
│ Notes:                                                   │
│ [_____________________________]                          │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Service autocomplete** from services catalog
**Tax calculation** automatic based on invoice type
**Insurance selection** pre-fills coverage details

#### CAE Request (AFIP)
```
┌────────────────────────────────┐
│ Request CAE from AFIP          │
├────────────────────────────────┤
│                                │
│ Invoice Type: B                │
│ Point of Sale: 0001            │
│ Invoice Number: 00000123       │
│ Amount: $5,000.00              │
│                                │
│ ⏳ Requesting CAE from AFIP... │
│                                │
│ ✓ CAE Received!                │
│ CAE: 12345678901234            │
│ Expiration: 25/11/2025         │
│                                │
│          [Print Invoice]       │
└────────────────────────────────┘
```

**Real-time status** during AFIP request
**Error handling** with retry option
**Success confirmation** with CAE details

#### Payment Recording
```
┌──────────────────────────────────────────────────────────┐
│ Record Payment - Invoice #0001-00000123                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Invoice Total:    $5,000.00                              │
│ Amount Paid:      $0.00                                  │
│ Balance Due:      $5,000.00                              │
│                                                          │
│ Payment Amount *                                         │
│ [$5,000.00]                                              │
│                                                          │
│ Payment Method *                                         │
│ ● Cash  ○ Card  ○ Bank Transfer  ○ Mercado Pago         │
│                                                          │
│ Payment Date                                             │
│ [15/11/2025]                                             │
│                                                          │
│ Reference/Transaction ID                                 │
│ [____________]                                           │
│                                                          │
│ Notes                                                    │
│ [_____________________________]                          │
│                                                          │
│ ☑ Print receipt                                          │
│                                                          │
│                    [Cancel]  [Record Payment]            │
└──────────────────────────────────────────────────────────┘
```

**Partial payment** support
**Receipt generation** automatic
**Payment method** specific fields (card last 4, transaction ID)

---

### 8. REPORTS MODULE

#### Reports Dashboard
```
┌──────────────────────────────────────────────────────────┐
│ Reports & Analytics                                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Select Report Type:                                      │
│                                                          │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │
│ │  Financial   │ │  Clinical    │ │ Operational  │     │
│ │  Reports     │ │  Reports     │ │  Reports     │     │
│ └──────────────┘ └──────────────┘ └──────────────┘     │
│                                                          │
│ ┌──────────────────────────────────────────────────────┐│
│ │ Revenue Report                                       ││
│ │                                                      ││
│ │ Date Range: [01/11/2025] to [15/11/2025]  [Generate]││
│ │                                                      ││
│ │ Group By: ○ Day  ● Week  ○ Month                     ││
│ │ Filter:   [All Providers ▼] [All Insurances ▼]      ││
│ │                                                      ││
│ │ ┌──────────────────────────────────────────────────┐││
│ │ │ Total Revenue: $125,000                          │││
│ │ │ [Bar Chart by Week]                              │││
│ │ └──────────────────────────────────────────────────┘││
│ │                                                      ││
│ │ [Export to Excel] [Export to PDF] [Print]           ││
│ └──────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────┘
```

**Report categories** as cards
**Filter options** for date, provider, insurance, etc.
**Visualization** with charts/graphs
**Export options** (Excel, PDF, print)

#### Custom Report Builder
```
┌──────────────────────────────────────────────────────────┐
│ Custom Report Builder                              [Save]│
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Report Name                                              │
│ [My Custom Report]                                       │
│                                                          │
│ Data Source                                              │
│ [Patients ▼]                                             │
│                                                          │
│ Columns to Include:                                      │
│ ☑ Name                                                   │
│ ☑ Age                                                    │
│ ☑ Insurance                                              │
│ ☑ Last Visit Date                                        │
│ ☐ Diagnoses                                              │
│                                                          │
│ Filters:                                                 │
│ [+ Add Filter]                                           │
│ • Age >= 18                                              │
│ • Insurance = "OSDE"                                     │
│                                                          │
│ Sort By: [Last Visit Date ▼]  Order: [Desc ▼]           │
│                                                          │
│                    [Preview]  [Generate Report]          │
└──────────────────────────────────────────────────────────┘
```

---

### 9. SETTINGS MODULE

#### Settings Navigation
```
┌──────────────────────────────────────────────────────────┐
│ Settings                                                 │
├──────────────────────────────────────────────────────────┤
│ ┌────────────┬───────────────────────────────────────┐  │
│ │ General    │ Clinic Information                    │  │
│ │ Users      │                                       │  │
│ │ Insurances │ Clinic Name                           │  │
│ │ Services   │ [Clínica Central]                     │  │
│ │ Templates  │                                       │  │
│ │ Billing    │ Address                               │  │
│ │ Integration│ [Av. Corrientes 1234...]              │  │
│ │ Security   │                                       │  │
│ │ Backup     │ Phone                                 │  │
│ └────────────┤ [+54 11 1234-5678]                    │  │
│              │                                       │  │
│              │ Upload Logo                           │  │
│              │ [Choose File]  [Upload]               │  │
│              │                                       │  │
│              │ [Save Changes]                        │  │
│              └───────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**Left sidebar** for settings categories
**Right panel** for selected category
**Form-based** configuration

---

## Mobile Responsive Design

### Mobile Navigation (< 768px)
```
┌─────────────────────────┐
│ ☰  Clinical System  🔔  │  ← Top bar with hamburger menu
├─────────────────────────┤
│                         │
│   Mobile Content Area   │
│                         │
│                         │
│                         │
│                         │
├─────────────────────────┤
│ [👥][📅][📋][💰][⚙️]   │  ← Bottom navigation (key items)
└─────────────────────────┘
```

### Mobile Adaptations
- **Navigation:** Hamburger menu + bottom nav for key items
- **Tables:** Horizontal scroll or card view
- **Forms:** Stack fields vertically
- **Modals:** Full-screen on mobile
- **Calendar:** Day view default, swipe to change days
- **Search:** Expandable search bar

---

## Accessibility (WCAG 2.1 AA)

### Requirements
- **Keyboard Navigation:** All interactive elements accessible via keyboard
- **Screen Reader:** Proper ARIA labels and semantic HTML
- **Color Contrast:** Minimum 4.5:1 for normal text, 3:1 for large text
- **Focus Indicators:** Clear visible focus states
- **Alt Text:** All images have descriptive alt text
- **Form Labels:** All form inputs properly labeled
- **Error Messages:** Clear, descriptive error messages
- **Skip Links:** "Skip to main content" link

### Keyboard Shortcuts (Optional)
- `Ctrl/Cmd + K` - Global search
- `N` - New (patient, appointment, etc. based on context)
- `S` - Save
- `Esc` - Close modal/dialog
- `?` - Show keyboard shortcuts help

---

## Loading States & Feedback

### Loading Spinner
```
     ⏳
  Loading...
```
- Show during data fetching
- Skeleton screens for tables/lists

### Empty States
```
┌──────────────────────────┐
│     [Illustration]       │
│                          │
│  No patients found       │
│  Add your first patient  │
│                          │
│   [+ Add Patient]        │
└──────────────────────────┘
```
- Friendly message
- Call-to-action button
- Helpful illustration

### Error States
```
┌──────────────────────────┐
│     ⚠️                    │
│                          │
│  Something went wrong    │
│  Please try again        │
│                          │
│   [Retry]  [Cancel]      │
└──────────────────────────┘
```
- Clear error message
- Suggested action
- Retry option

---

## Performance Considerations

### Optimization Strategies
- **Lazy Loading:** Load modules on demand
- **Pagination:** Limit initial data load
- **Debouncing:** Search inputs debounced (300ms)
- **Caching:** Cache frequently accessed data
- **Image Optimization:** Compress and lazy-load images
- **Code Splitting:** Split bundles by route
- **Virtual Scrolling:** For long lists (> 100 items)

### Target Metrics
- **First Contentful Paint:** < 1.5s
- **Time to Interactive:** < 3s
- **Largest Contentful Paint:** < 2.5s
- **Page Load:** < 2s on 4G connection

---

## Print Layouts

### Printable Documents
- Prescriptions
- Lab orders
- Invoices (with QR code)
- Patient summaries
- Reports

### Print CSS
- Remove navigation/sidebars
- Optimize for A4/Letter paper
- Include clinic header/footer
- Page breaks where appropriate
- QR codes for AFIP invoices

---

*UI/UX Specifications Version: 1.0*
*Last Updated: 2025-11-15*
*Ready for Implementation*
