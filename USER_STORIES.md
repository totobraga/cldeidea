# Clinical Management System - User Stories

## User Personas

1. **Dr. Martinez** - General Practitioner, clinic owner
2. **Nurse Sofia** - Registered nurse, assists with patient care
3. **Receptionist Ana** - Front desk, scheduling, billing
4. **Patient Carlos** - 45-year-old patient with obra social
5. **Admin Laura** - System administrator
6. **Billing Staff Maria** - Handles invoicing and insurance claims

---

## MODULE 1: AUTHENTICATION & USER MANAGEMENT

### Authentication
- **US-001:** As a **user**, I want to log in with my email and password so that I can access the system securely
- **US-002:** As a **user**, I want to reset my password if I forget it so that I can regain access
- **US-003:** As a **user**, I want to be automatically logged out after inactivity so that my session is secure
- **US-004:** As an **admin**, I want to enable two-factor authentication so that the system is more secure

### User Management
- **US-005:** As an **admin**, I want to create user accounts so that staff can access the system
- **US-006:** As an **admin**, I want to assign roles to users so that they have appropriate permissions
- **US-007:** As an **admin**, I want to deactivate user accounts so that former staff cannot access the system
- **US-008:** As an **admin**, I want to view audit logs of user actions so that I can track system usage
- **US-009:** As a **user**, I want to update my profile information so that my contact details are current

---

## MODULE 2: PATIENT MANAGEMENT

### Patient Registration
- **US-010:** As a **receptionist**, I want to register a new patient so that they can be scheduled for appointments
- **US-011:** As a **receptionist**, I want to capture patient DNI and obra social information so that billing is accurate
- **US-012:** As a **receptionist**, I want to record patient allergies so that the doctor is aware of contraindications
- **US-013:** As a **receptionist**, I want to upload a photo of the patient so that identification is easier
- **US-014:** As a **receptionist**, I want to record emergency contact information so that we can reach someone if needed

### Patient Search & Management
- **US-015:** As a **receptionist**, I want to search for patients by name, DNI, or phone so that I can quickly find their records
- **US-016:** As a **receptionist**, I want to view a patient's complete profile so that I have all their information in one place
- **US-017:** As a **receptionist**, I want to edit patient information so that records stay up to date
- **US-018:** As a **receptionist**, I want to merge duplicate patient records so that data is consolidated
- **US-019:** As an **admin**, I want to archive inactive patients so that the active patient list is manageable

### Medical Background
- **US-020:** As a **doctor**, I want to view a patient's allergy list so that I can prescribe safely
- **US-021:** As a **doctor**, I want to record chronic conditions so that I have a complete medical picture
- **US-022:** As a **doctor**, I want to see family medical history so that I can assess genetic risks
- **US-023:** As a **doctor**, I want to view immunization records so that I know what vaccines are due
- **US-024:** As a **nurse**, I want to update a patient's vital signs history so that trends are visible

---

## MODULE 3: APPOINTMENT SCHEDULING

### Appointment Booking
- **US-025:** As a **receptionist**, I want to view the doctor's calendar so that I can see available time slots
- **US-026:** As a **receptionist**, I want to book an appointment for a patient so that they can see the doctor
- **US-027:** As a **receptionist**, I want to set the appointment type so that the correct time is allocated
- **US-028:** As a **receptionist**, I want to add notes to an appointment so that special requirements are communicated
- **US-029:** As a **patient**, I want to book an appointment online so that I don't need to call

### Appointment Management
- **US-030:** As a **receptionist**, I want to cancel an appointment so that the slot becomes available
- **US-031:** As a **receptionist**, I want to reschedule an appointment so that I can accommodate patient requests
- **US-032:** As a **receptionist**, I want to mark a patient as "arrived" so that the doctor knows who's waiting
- **US-033:** As a **receptionist**, I want to see a waitlist so that I can fill cancelled appointments
- **US-034:** As a **doctor**, I want to block time on my calendar so that I can attend meetings

### Calendar Views
- **US-035:** As a **doctor**, I want to view my daily schedule so that I know what appointments I have
- **US-036:** As a **doctor**, I want to view my weekly schedule so that I can plan ahead
- **US-037:** As a **receptionist**, I want to view multiple doctors' calendars so that I can distribute appointments
- **US-038:** As a **receptionist**, I want to filter appointments by status so that I can see who's confirmed
- **US-039:** As a **doctor**, I want to see appointment duration so that I can manage my time

### Reminders
- **US-040:** As a **patient**, I want to receive SMS reminders so that I don't miss my appointment
- **US-041:** As a **patient**, I want to receive email reminders so that I'm notified in advance
- **US-042:** As an **admin**, I want to configure reminder timing so that patients are notified appropriately
- **US-043:** As a **receptionist**, I want to manually send a reminder so that I can follow up with patients

---

## MODULE 4: ELECTRONIC HEALTH RECORDS (EHR)

### Clinical Consultation
- **US-044:** As a **doctor**, I want to document the chief complaint so that the reason for visit is recorded
- **US-045:** As a **doctor**, I want to record vital signs so that baseline health metrics are tracked
- **US-046:** As a **doctor**, I want to enter physical examination findings so that assessment is documented
- **US-047:** As a **doctor**, I want to record my clinical impression so that the diagnosis is clear
- **US-048:** As a **doctor**, I want to create a treatment plan so that next steps are defined

### Medical History
- **US-049:** As a **doctor**, I want to view a timeline of all consultations so that I can see the patient's history
- **US-050:** As a **doctor**, I want to see previous diagnoses so that I understand ongoing conditions
- **US-051:** As a **doctor**, I want to review previous treatments so that I can assess what worked
- **US-052:** As a **doctor**, I want to see hospitalization history so that major events are known
- **US-053:** As a **doctor**, I want to filter history by date range so that I can focus on recent visits

### Clinical Notes
- **US-054:** As a **doctor**, I want to use SOAP note templates so that documentation is consistent
- **US-055:** As a **doctor**, I want to attach images to notes so that visual findings are recorded
- **US-056:** As a **doctor**, I want to use rich text formatting so that notes are readable
- **US-057:** As a **doctor**, I want to save notes as draft so that I can complete them later
- **US-058:** As a **doctor**, I want to lock finalized notes so that they cannot be altered

### Diagnoses
- **US-059:** As a **doctor**, I want to search ICD-10 codes so that diagnoses are standardized
- **US-060:** As a **doctor**, I want to mark primary and secondary diagnoses so that hierarchy is clear
- **US-061:** As a **doctor**, I want to see active diagnoses so that I know current conditions
- **US-062:** As a **doctor**, I want to resolve diagnoses so that the status is updated

---

## MODULE 5: PRESCRIPTION MANAGEMENT

### Creating Prescriptions
- **US-063:** As a **doctor**, I want to search for medications so that I can prescribe accurately
- **US-064:** As a **doctor**, I want to specify dosage and frequency so that instructions are clear
- **US-065:** As a **doctor**, I want to see drug interactions so that I can avoid contraindications
- **US-066:** As a **doctor**, I want to check against allergies so that prescriptions are safe
- **US-067:** As a **doctor**, I want to use prescription templates so that common prescriptions are quick

### Prescription Output
- **US-068:** As a **doctor**, I want to print prescriptions so that patients can fill them
- **US-069:** As a **doctor**, I want to generate PDF prescriptions so that they can be emailed
- **US-070:** As a **doctor**, I want to electronically sign prescriptions so that they're valid
- **US-071:** As a **doctor**, I want to print on prescription pads so that they're official
- **US-072:** As a **patient**, I want to receive prescriptions via email so that I don't lose them

### Medication Management
- **US-073:** As a **doctor**, I want to view active medications so that I know what the patient is taking
- **US-074:** As a **doctor**, I want to discontinue medications so that the list is current
- **US-075:** As a **doctor**, I want to see medication history so that I can track adherence
- **US-076:** As a **doctor**, I want to mark medications as chronic so that refills are tracked
- **US-077:** As a **pharmacist**, I want to view prescription history so that I can verify refills

---

## MODULE 6: LABORATORY & DIAGNOSTIC IMAGING

### Lab Orders
- **US-078:** As a **doctor**, I want to order lab tests so that diagnostics are performed
- **US-079:** As a **doctor**, I want to use order templates so that common panels are quick
- **US-080:** As a **doctor**, I want to mark orders as urgent so that priority is clear
- **US-081:** As a **doctor**, I want to add special instructions so that the lab knows requirements
- **US-082:** As a **doctor**, I want to print lab orders so that patients can take them to the lab

### Lab Results
- **US-083:** As a **lab tech**, I want to enter lab results so that doctors can review them
- **US-084:** As a **doctor**, I want to see flagged abnormal results so that critical values are highlighted
- **US-085:** As a **doctor**, I want to view result trends so that I can see changes over time
- **US-086:** As a **doctor**, I want to add comments to results so that interpretation is noted
- **US-087:** As a **patient**, I want to be notified when results are ready so that I can follow up

### Imaging
- **US-088:** As a **doctor**, I want to order imaging studies so that diagnostics are complete
- **US-089:** As a **doctor**, I want to view uploaded images so that I can review findings
- **US-090:** As a **doctor**, I want to attach radiology reports so that findings are documented
- **US-091:** As a **radiologist**, I want to upload DICOM images so that they're in the system

---

## MODULE 7: BILLING & INVOICING

### Service Charges
- **US-092:** As a **receptionist**, I want to add charges to a visit so that billing is captured
- **US-093:** As a **receptionist**, I want to see obra social coverage so that I know what's covered
- **US-094:** As a **billing staff**, I want to calculate coseguro so that patient payment is correct
- **US-095:** As a **billing staff**, I want to apply discounts so that special pricing is honored
- **US-096:** As a **admin**, I want to configure price lists so that rates are current

### Payment Processing
- **US-097:** As a **receptionist**, I want to record cash payments so that transactions are logged
- **US-098:** As a **receptionist**, I want to process card payments so that patients have options
- **US-099:** As a **receptionist**, I want to issue receipts so that patients have proof of payment
- **US-100:** As a **receptionist**, I want to record partial payments so that balances are tracked
- **US-101:** As a **billing staff**, I want to see outstanding balances so that I can follow up

### Obra Social Billing
- **US-102:** As a **billing staff**, I want to verify coverage so that claims will be paid
- **US-103:** As a **billing staff**, I want to submit claims so that reimbursement is requested
- **US-104:** As a **billing staff**, I want to track claim status so that I know what's pending
- **US-105:** As a **billing staff**, I want to handle rejections so that claims can be corrected
- **US-106:** As a **billing staff**, I want to see reimbursement so that payment is reconciled

### AFIP Invoicing
- **US-107:** As a **billing staff**, I want to generate Factura B so that consumers are invoiced
- **US-108:** As a **billing staff**, I want to generate Factura A so that businesses are invoiced
- **US-109:** As a **billing staff**, I want to request CAE so that invoices are authorized
- **US-110:** As a **billing staff**, I want to print invoices with QR codes so that they're compliant
- **US-111:** As a **billing staff**, I want to generate credit notes so that adjustments are made

### Financial Reports
- **US-112:** As an **admin**, I want to see daily revenue so that cash flow is monitored
- **US-113:** As an **admin**, I want to see revenue by doctor so that productivity is tracked
- **US-114:** As an **admin**, I want to see revenue by obra social so that payer mix is understood
- **US-115:** As an **admin**, I want to export financial data so that accounting is simplified

---

## MODULE 8: INVENTORY MANAGEMENT

### Inventory Tracking
- **US-116:** As an **admin**, I want to add items to inventory so that supplies are tracked
- **US-117:** As a **nurse**, I want to see current stock levels so that I know what's available
- **US-118:** As an **admin**, I want to set minimum stock alerts so that reordering is timely
- **US-119:** As a **nurse**, I want to record stock usage so that inventory is accurate
- **US-120:** As an **admin**, I want to track expiration dates so that expired items are removed

### Purchasing
- **US-121:** As an **admin**, I want to create purchase orders so that supplies are ordered
- **US-122:** As an **admin**, I want to receive stock so that inventory is updated
- **US-123:** As an **admin**, I want to manage suppliers so that vendor info is available
- **US-124:** As an **admin**, I want to track costs so that expenses are monitored

### Reports
- **US-125:** As an **admin**, I want to see inventory valuation so that asset value is known
- **US-126:** As an **admin**, I want to see expiring items so that waste is minimized
- **US-127:** As an **admin**, I want to see usage trends so that ordering is optimized

---

## MODULE 9: REPORTING & ANALYTICS

### Clinical Reports
- **US-128:** As an **admin**, I want to see new patient counts so that growth is tracked
- **US-129:** As a **doctor**, I want to see my consultation volume so that workload is understood
- **US-130:** As an **admin**, I want to see common diagnoses so that clinical trends are visible
- **US-131:** As an **admin**, I want to see no-show rates so that scheduling can be improved

### Financial Reports
- **US-132:** As an **admin**, I want to see monthly revenue so that financial health is monitored
- **US-133:** As an **admin**, I want to see collection rates so that billing efficiency is tracked
- **US-134:** As an **admin**, I want to see accounts receivable aging so that collections are prioritized
- **US-135:** As an **admin**, I want to see provider productivity so that compensation is calculated

### Operational Reports
- **US-136:** As an **admin**, I want to see appointment utilization so that capacity is optimized
- **US-137:** As an **admin**, I want to see average wait times so that patient experience is improved
- **US-138:** As an **admin**, I want to export reports to Excel so that further analysis is possible
- **US-139:** As an **admin**, I want to schedule automated reports so that stakeholders are informed

---

## MODULE 10: COMMUNICATIONS

### Patient Communications
- **US-140:** As a **patient**, I want to receive appointment reminders so that I don't forget
- **US-141:** As a **patient**, I want to be notified when results are ready so that I can follow up
- **US-142:** As a **patient**, I want to receive birthday greetings so that I feel valued
- **US-143:** As a **patient**, I want to receive health tips so that I stay informed

### Internal Communications
- **US-144:** As a **doctor**, I want to send messages to staff so that coordination is easy
- **US-145:** As a **receptionist**, I want to see announcements so that I'm informed of changes
- **US-146:** As a **nurse**, I want to assign tasks so that work is distributed
- **US-147:** As a **doctor**, I want to leave handoff notes so that coverage is seamless

### WhatsApp Integration
- **US-148:** As a **patient**, I want to receive WhatsApp reminders so that I see notifications
- **US-149:** As an **admin**, I want to send WhatsApp confirmations so that patients acknowledge
- **US-150:** As a **receptionist**, I want to use WhatsApp templates so that messaging is professional

---

## MODULE 11: PATIENT PORTAL

### Portal Access
- **US-151:** As a **patient**, I want to register for the portal so that I can access my records
- **US-152:** As a **patient**, I want to log in securely so that my data is protected
- **US-153:** As a **patient**, I want to reset my password so that I can regain access

### Portal Features
- **US-154:** As a **patient**, I want to view my appointment history so that I have a record
- **US-155:** As a **patient**, I want to request appointments so that I don't need to call
- **US-156:** As a **patient**, I want to view my lab results so that I'm informed
- **US-157:** As a **patient**, I want to download prescriptions so that I have copies
- **US-158:** As a **patient**, I want to pay invoices online so that payment is convenient
- **US-159:** As a **patient**, I want to update my contact info so that records are current
- **US-160:** As a **patient**, I want to message my doctor so that I can ask questions

---

## MODULE 12: ADMINISTRATION

### System Configuration
- **US-161:** As an **admin**, I want to configure appointment types so that scheduling is accurate
- **US-162:** As an **admin**, I want to manage obra sociales so that billing is correct
- **US-163:** As an **admin**, I want to set clinic hours so that availability is defined
- **US-164:** As an **admin**, I want to configure email templates so that communications are branded
- **US-165:** As an **admin**, I want to upload the clinic logo so that documents are professional

### Data Management
- **US-166:** As an **admin**, I want to backup data so that information is protected
- **US-167:** As an **admin**, I want to restore from backup so that disaster recovery is possible
- **US-168:** As an **admin**, I want to view audit logs so that system activity is tracked
- **US-169:** As an **admin**, I want to export data so that migration is possible

### Security
- **US-170:** As an **admin**, I want to enforce password policies so that accounts are secure
- **US-171:** As an **admin**, I want to see login attempts so that suspicious activity is detected
- **US-172:** As an **admin**, I want to configure session timeout so that idle sessions are closed

---

## ARGENTINA-SPECIFIC USER STORIES

### AFIP Integration
- **US-173:** As a **billing staff**, I want to authenticate with AFIP so that invoicing is authorized
- **US-174:** As a **billing staff**, I want to request CAE automatically so that manual steps are avoided
- **US-175:** As a **billing staff**, I want to handle AFIP errors so that failed invoices are retried

### Obra Social Integration
- **US-176:** As a **billing staff**, I want to verify PAMI coverage so that claims are valid
- **US-177:** As a **billing staff**, I want to use nomenclator codes so that billing is standardized
- **US-178:** As a **billing staff**, I want to track prior authorizations so that services are approved

### Compliance
- **US-179:** As an **admin**, I want to obtain patient consent so that data privacy laws are followed
- **US-180:** As an **admin**, I want to log data access so that professional secrecy is maintained
- **US-181:** As an **admin**, I want to handle data deletion requests so that patient rights are respected

---

## ACCEPTANCE CRITERIA EXAMPLES

### Example for US-010 (Patient Registration)
**Given** I am a receptionist
**When** I fill in the patient registration form with valid data
**And** I click "Save Patient"
**Then** the patient is created in the system
**And** I see a success message
**And** the patient appears in the patient list
**And** a unique patient ID is generated

### Example for US-025 (View Calendar)
**Given** I am a receptionist
**When** I navigate to the appointments calendar
**Then** I see the doctor's schedule for today
**And** available time slots are highlighted in green
**And** booked appointments show patient names
**And** I can navigate to other dates using the date picker

### Example for US-107 (Generate Factura B)
**Given** I am billing staff
**When** I complete a patient visit with cash payment
**And** I click "Generate Invoice"
**Then** a Factura B is created
**And** a CAE is requested from AFIP
**And** the invoice displays the CAE number
**And** the invoice can be printed with QR code
**And** the invoice is stored in the system

---

## STORY MAPPING

### MVP (Phase 1) Stories
Minimum viable product to launch: US-001 to US-061, US-092 to US-100, US-161 to US-165

### Phase 2 Stories
Enhanced features: US-063 to US-091, US-102 to US-115, US-173 to US-178

### Phase 3 Stories
Patient engagement: US-140 to US-160

### Phase 4 Stories
Advanced features: US-116 to US-139

### Future/Optional Stories
Nice-to-have features: US-179 to US-181, portal enhancements

---

*User Stories Version: 1.0*
*Last Updated: 2025-11-15*
*Total Stories: 181*
