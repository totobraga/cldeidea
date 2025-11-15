# Clinical Management System - API Endpoints Documentation

## API Design Principles
- RESTful architecture
- JSON request/response format
- JWT authentication
- API versioning (v1, v2, etc.)
- Standard HTTP status codes
- Pagination for list endpoints
- Filtering, sorting, and search capabilities
- Rate limiting
- CORS support

---

## Base URL
```
Production: https://api.clinicmanagement.com/v1
Development: http://localhost:3000/api/v1
```

---

## Authentication

### POST /auth/login
Login to the system
```json
Request:
{
  "email": "doctor@clinic.com",
  "password": "securePassword123"
}

Response: 200 OK
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": 1,
      "email": "doctor@clinic.com",
      "firstName": "Juan",
      "lastName": "Martinez",
      "role": "doctor"
    }
  }
}
```

### POST /auth/logout
Logout from the system
```json
Request: (No body, token in header)

Response: 200 OK
{
  "success": true,
  "message": "Logged out successfully"
}
```

### POST /auth/refresh-token
Refresh access token
```json
Request:
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

Response: 200 OK
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### POST /auth/forgot-password
Request password reset
```json
Request:
{
  "email": "doctor@clinic.com"
}

Response: 200 OK
{
  "success": true,
  "message": "Password reset email sent"
}
```

### POST /auth/reset-password
Reset password with token
```json
Request:
{
  "token": "reset-token-from-email",
  "newPassword": "newSecurePassword123"
}

Response: 200 OK
{
  "success": true,
  "message": "Password reset successfully"
}
```

---

## Patients

### GET /patients
Get list of patients (paginated, searchable)
```
Query Parameters:
- page (default: 1)
- limit (default: 20)
- search (search by name, DNI, phone)
- insuranceId (filter by insurance)
- isActive (true/false)
- sortBy (firstName, lastName, createdAt)
- sortOrder (asc, desc)

Response: 200 OK
{
  "success": true,
  "data": {
    "patients": [...],
    "pagination": {
      "total": 150,
      "page": 1,
      "limit": 20,
      "totalPages": 8
    }
  }
}
```

### GET /patients/:id
Get patient details
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "id": 1,
    "patientNumber": "P-00001",
    "firstName": "Carlos",
    "lastName": "Rodriguez",
    "dni": "30123456",
    "dateOfBirth": "1980-05-15",
    "age": 45,
    "gender": "male",
    "bloodType": "O+",
    "email": "carlos@email.com",
    "phonePrimary": "+54 9 11 1234-5678",
    "address": {...},
    "emergencyContact": {...},
    "insurances": [...],
    "medicalHistory": {...},
    "createdAt": "2024-01-01T10:00:00Z"
  }
}
```

### POST /patients
Create new patient
```json
Request:
{
  "firstName": "Carlos",
  "lastName": "Rodriguez",
  "dni": "30123456",
  "dateOfBirth": "1980-05-15",
  "gender": "male",
  "phonePrimary": "+54 9 11 1234-5678",
  "email": "carlos@email.com",
  "addressStreet": "Av. Corrientes 1234",
  "addressCity": "Buenos Aires",
  "insurances": [
    {
      "insuranceId": 5,
      "memberNumber": "123456789",
      "isPrimary": true
    }
  ]
}

Response: 201 Created
{
  "success": true,
  "data": { /* patient object */ },
  "message": "Patient created successfully"
}
```

### PUT /patients/:id
Update patient information
```json
Request: (Same structure as POST, all fields optional)

Response: 200 OK
{
  "success": true,
  "data": { /* updated patient object */ },
  "message": "Patient updated successfully"
}
```

### DELETE /patients/:id
Soft delete patient (archive)
```json
Response: 200 OK
{
  "success": true,
  "message": "Patient archived successfully"
}
```

### GET /patients/:id/medical-history
Get patient medical history
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "allergies": [...],
    "chronicConditions": [...],
    "surgeries": [...],
    "familyHistory": [...],
    "immunizations": [...],
    "smokingStatus": "never",
    "alcoholConsumption": "occasional"
  }
}
```

### PUT /patients/:id/medical-history
Update medical history
```json
Request: (Same structure as GET response)

Response: 200 OK
{
  "success": true,
  "data": { /* updated medical history */ }
}
```

### GET /patients/:id/timeline
Get patient clinical timeline
```json
Response: 200 OK
{
  "success": true,
  "data": [
    {
      "type": "encounter",
      "id": 123,
      "date": "2025-11-10T14:00:00Z",
      "provider": "Dr. Martinez",
      "summary": "Follow-up consultation"
    },
    {
      "type": "prescription",
      "id": 45,
      "date": "2025-11-10T14:30:00Z",
      "medications": ["Ibuprofeno 400mg"]
    },
    {
      "type": "lab_order",
      "id": 78,
      "date": "2025-11-10T14:35:00Z",
      "tests": ["Hemograma completo"]
    }
  ]
}
```

---

## Appointments

### GET /appointments
Get list of appointments
```
Query Parameters:
- date (YYYY-MM-DD)
- startDate, endDate (date range)
- providerId
- patientId
- status (scheduled, confirmed, completed, etc.)
- page, limit

Response: 200 OK
{
  "success": true,
  "data": {
    "appointments": [
      {
        "id": 1,
        "patientId": 10,
        "patientName": "Carlos Rodriguez",
        "providerId": 2,
        "providerName": "Dr. Martinez",
        "scheduledAt": "2025-11-15T10:00:00Z",
        "duration": 30,
        "appointmentType": "Consulta nueva",
        "status": "scheduled",
        "chiefComplaint": "Control de presión"
      }
    ],
    "pagination": {...}
  }
}
```

### GET /appointments/:id
Get appointment details
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "id": 1,
    "patient": {...},
    "provider": {...},
    "scheduledAt": "2025-11-15T10:00:00Z",
    "duration": 30,
    "appointmentType": {...},
    "status": "scheduled",
    "chiefComplaint": "Control de presión",
    "notes": "",
    "checkedInAt": null,
    "createdAt": "2025-11-10T12:00:00Z"
  }
}
```

### POST /appointments
Create new appointment
```json
Request:
{
  "patientId": 10,
  "providerId": 2,
  "scheduledAt": "2025-11-15T10:00:00Z",
  "duration": 30,
  "appointmentTypeId": 1,
  "chiefComplaint": "Control de presión",
  "notes": ""
}

Response: 201 Created
{
  "success": true,
  "data": { /* appointment object */ },
  "message": "Appointment created successfully"
}
```

### PUT /appointments/:id
Update appointment
```json
Request: (Same structure as POST, fields optional)

Response: 200 OK
{
  "success": true,
  "data": { /* updated appointment */ }
}
```

### PATCH /appointments/:id/status
Update appointment status
```json
Request:
{
  "status": "confirmed",
  "notes": "Paciente confirmó por teléfono"
}

Response: 200 OK
{
  "success": true,
  "data": { /* updated appointment */ }
}
```

### POST /appointments/:id/check-in
Check in patient for appointment
```json
Response: 200 OK
{
  "success": true,
  "data": { /* appointment with checkedInAt timestamp */ }
}
```

### DELETE /appointments/:id
Cancel appointment
```json
Request:
{
  "reason": "Paciente canceló por motivos personales"
}

Response: 200 OK
{
  "success": true,
  "message": "Appointment cancelled"
}
```

### GET /appointments/availability
Check provider availability
```
Query Parameters:
- providerId (required)
- date (YYYY-MM-DD, required)
- duration (in minutes, default: 30)

Response: 200 OK
{
  "success": true,
  "data": {
    "date": "2025-11-15",
    "provider": {...},
    "availableSlots": [
      {
        "start": "2025-11-15T09:00:00Z",
        "end": "2025-11-15T09:30:00Z"
      },
      {
        "start": "2025-11-15T09:30:00Z",
        "end": "2025-11-15T10:00:00Z"
      }
    ]
  }
}
```

---

## Encounters (Clinical Consultations)

### GET /encounters
Get list of encounters
```
Query Parameters:
- patientId
- providerId
- startDate, endDate
- page, limit

Response: 200 OK
```

### GET /encounters/:id
Get encounter details
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "id": 1,
    "patient": {...},
    "provider": {...},
    "encounterDate": "2025-11-10T14:00:00Z",
    "encounterType": "outpatient",
    "subjective": "Paciente refiere dolor de cabeza...",
    "objective": "PA: 120/80, FC: 72 bpm, Temp: 36.5°C...",
    "assessment": "Cefalea tensional",
    "plan": "Analgésicos, reposo, seguimiento en 1 semana",
    "vitalSigns": {...},
    "diagnoses": [...],
    "status": "finalized",
    "finalizedAt": "2025-11-10T15:00:00Z"
  }
}
```

### POST /encounters
Create new encounter
```json
Request:
{
  "patientId": 10,
  "providerId": 2,
  "appointmentId": 5,
  "encounterDate": "2025-11-10T14:00:00Z",
  "subjective": "...",
  "objective": "...",
  "assessment": "...",
  "plan": "..."
}

Response: 201 Created
```

### PUT /encounters/:id
Update encounter
```json
Response: 200 OK
```

### POST /encounters/:id/finalize
Finalize and lock encounter
```json
Response: 200 OK
{
  "success": true,
  "data": { /* encounter with status=finalized */ },
  "message": "Encounter finalized successfully"
}
```

### POST /encounters/:id/vital-signs
Add vital signs to encounter
```json
Request:
{
  "systolicBp": 120,
  "diastolicBp": 80,
  "heartRate": 72,
  "temperature": 36.5,
  "weight": 75,
  "height": 170
}

Response: 201 Created
```

---

## Prescriptions

### GET /prescriptions
Get list of prescriptions
```
Query Parameters:
- patientId
- providerId
- startDate, endDate
- status (active, dispensed, cancelled)

Response: 200 OK
```

### GET /prescriptions/:id
Get prescription details
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "id": 1,
    "prescriptionNumber": "RX-00001",
    "patient": {...},
    "provider": {...},
    "prescriptionDate": "2025-11-10",
    "status": "active",
    "items": [
      {
        "medicationName": "Ibuprofeno",
        "dosage": "400mg",
        "form": "comprimido",
        "frequency": "cada 8 horas",
        "duration": "7 días",
        "quantity": 21,
        "instructions": "Tomar con alimentos"
      }
    ]
  }
}
```

### POST /prescriptions
Create new prescription
```json
Request:
{
  "patientId": 10,
  "encounterId": 5,
  "items": [
    {
      "medicationName": "Ibuprofeno",
      "dosage": "400mg",
      "form": "comprimido",
      "frequency": "cada 8 horas",
      "duration": "7 días",
      "quantity": 21,
      "instructions": "Tomar con alimentos"
    }
  ]
}

Response: 201 Created
```

### GET /prescriptions/:id/pdf
Generate prescription PDF
```
Response: 200 OK (PDF file download)
```

### POST /prescriptions/:id/send-email
Email prescription to patient
```json
Request:
{
  "email": "patient@email.com"
}

Response: 200 OK
```

### GET /prescriptions/patient/:patientId/active
Get patient's active medications
```json
Response: 200 OK
{
  "success": true,
  "data": [
    {
      "medicationName": "Ibuprofeno",
      "dosage": "400mg",
      "frequency": "cada 8 horas",
      "startDate": "2025-11-10",
      "isChonic": false
    }
  ]
}
```

---

## Laboratory

### GET /lab-orders
Get list of lab orders
```
Query Parameters:
- patientId
- providerId
- status
- startDate, endDate

Response: 200 OK
```

### GET /lab-orders/:id
Get lab order details
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "id": 1,
    "orderNumber": "LAB-00001",
    "patient": {...},
    "provider": {...},
    "orderDate": "2025-11-10T14:00:00Z",
    "status": "completed",
    "tests": [
      {
        "testName": "Hemograma completo",
        "testCode": "CBC",
        "status": "completed"
      }
    ],
    "results": [...]
  }
}
```

### POST /lab-orders
Create new lab order
```json
Request:
{
  "patientId": 10,
  "encounterId": 5,
  "tests": [
    {
      "testName": "Hemograma completo",
      "testCode": "CBC"
    }
  ],
  "isUrgent": false,
  "fastingRequired": true,
  "specialInstructions": "Ayuno de 12 horas"
}

Response: 201 Created
```

### POST /lab-orders/:id/results
Add results to lab order
```json
Request:
{
  "results": [
    {
      "testName": "Hemoglobina",
      "resultValue": "14.5",
      "resultUnit": "g/dL",
      "referenceRange": "12-16",
      "isAbnormal": false
    }
  ]
}

Response: 201 Created
```

### GET /lab-orders/:id/pdf
Generate lab order PDF
```
Response: 200 OK (PDF file)
```

---

## Billing & Invoicing

### GET /invoices
Get list of invoices
```
Query Parameters:
- patientId
- status (draft, sent, paid, partially_paid)
- startDate, endDate
- insuranceId

Response: 200 OK
```

### GET /invoices/:id
Get invoice details
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "id": 1,
    "invoiceNumber": "0001-00000123",
    "invoiceType": "B",
    "patient": {...},
    "invoiceDate": "2025-11-10",
    "cae": "12345678901234",
    "caeExpiration": "2025-11-20",
    "items": [
      {
        "description": "Consulta médica",
        "quantity": 1,
        "unitPrice": 5000.00,
        "total": 5000.00
      }
    ],
    "subtotal": 5000.00,
    "taxAmount": 0.00,
    "total": 5000.00,
    "amountPaid": 5000.00,
    "balanceDue": 0.00,
    "status": "paid"
  }
}
```

### POST /invoices
Create new invoice
```json
Request:
{
  "patientId": 10,
  "encounterId": 5,
  "invoiceType": "B",
  "items": [
    {
      "serviceId": 1,
      "description": "Consulta médica",
      "quantity": 1,
      "unitPrice": 5000.00
    }
  ]
}

Response: 201 Created
```

### POST /invoices/:id/request-cae
Request CAE from AFIP
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "cae": "12345678901234",
    "caeExpiration": "2025-11-20"
  }
}
```

### GET /invoices/:id/pdf
Generate invoice PDF with QR code
```
Response: 200 OK (PDF file)
```

### POST /invoices/:id/payments
Record payment for invoice
```json
Request:
{
  "amount": 5000.00,
  "paymentMethod": "cash",
  "paymentDate": "2025-11-10",
  "notes": ""
}

Response: 201 Created
```

### GET /payments
Get list of payments
```
Query Parameters:
- patientId
- startDate, endDate
- paymentMethod

Response: 200 OK
```

---

## Insurance & Claims

### GET /insurances
Get list of insurances (Obra Social/Prepaga)
```json
Response: 200 OK
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "OSDE",
      "type": "prepaga",
      "code": "OSDE",
      "isActive": true
    }
  ]
}
```

### GET /insurance-claims
Get list of insurance claims
```
Query Parameters:
- insuranceId
- patientId
- status (submitted, approved, denied, paid)

Response: 200 OK
```

### POST /insurance-claims
Submit insurance claim
```json
Request:
{
  "invoiceId": 10,
  "insuranceId": 5,
  "patientId": 10,
  "billedAmount": 5000.00,
  "authorizationNumber": "AUTH123456"
}

Response: 201 Created
```

### PATCH /insurance-claims/:id/status
Update claim status
```json
Request:
{
  "status": "approved",
  "approvedAmount": 4500.00,
  "notes": "Aprobado con ajuste"
}

Response: 200 OK
```

---

## Inventory

### GET /inventory
Get inventory items
```
Query Parameters:
- category
- lowStock (boolean - show items below reorder point)
- search

Response: 200 OK
```

### GET /inventory/:id
Get inventory item details
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "id": 1,
    "name": "Jeringas 5ml",
    "sku": "JER-005",
    "category": "consumable",
    "currentStock": 250,
    "minimumStock": 100,
    "reorderPoint": 150,
    "unitCost": 50.00,
    "batches": [...]
  }
}
```

### POST /inventory
Add new inventory item
```json
Request:
{
  "name": "Jeringas 5ml",
  "sku": "JER-005",
  "category": "consumable",
  "minimumStock": 100,
  "reorderPoint": 150,
  "unitCost": 50.00
}

Response: 201 Created
```

### POST /inventory/:id/transactions
Record inventory transaction
```json
Request:
{
  "transactionType": "in",
  "quantity": 100,
  "unitCost": 50.00,
  "notes": "Compra a proveedor X"
}

Response: 201 Created
```

### GET /inventory/alerts
Get low stock alerts
```json
Response: 200 OK
{
  "success": true,
  "data": [
    {
      "itemName": "Guantes L",
      "currentStock": 50,
      "reorderPoint": 100,
      "shortage": 50
    }
  ]
}
```

---

## Reports & Analytics

### GET /reports/revenue
Get revenue report
```
Query Parameters:
- startDate, endDate
- providerId
- insuranceId
- groupBy (day, week, month)

Response: 200 OK
{
  "success": true,
  "data": {
    "totalRevenue": 125000.00,
    "breakdown": [...]
  }
}
```

### GET /reports/appointments
Get appointment statistics
```
Query Parameters:
- startDate, endDate
- providerId

Response: 200 OK
{
  "success": true,
  "data": {
    "totalAppointments": 150,
    "completed": 120,
    "cancelled": 15,
    "noShow": 15,
    "noShowRate": 10.0
  }
}
```

### GET /reports/patients
Get patient statistics
```
Query Parameters:
- startDate, endDate

Response: 200 OK
{
  "success": true,
  "data": {
    "totalPatients": 500,
    "newPatients": 25,
    "demographics": {...}
  }
}
```

### GET /reports/diagnoses
Get most common diagnoses
```
Query Parameters:
- startDate, endDate
- limit (default: 10)

Response: 200 OK
{
  "success": true,
  "data": [
    {
      "icd10Code": "I10",
      "diagnosisName": "Hipertensión arterial",
      "count": 45
    }
  ]
}
```

---

## Communications

### POST /communications/send-sms
Send SMS to patient
```json
Request:
{
  "patientId": 10,
  "message": "Recordatorio: Tiene turno mañana a las 10:00",
  "referenceType": "appointment",
  "referenceId": 123
}

Response: 200 OK
{
  "success": true,
  "data": {
    "messageId": "msg_123",
    "status": "sent"
  }
}
```

### POST /communications/send-email
Send email to patient
```json
Request:
{
  "patientId": 10,
  "subject": "Resultados de laboratorio disponibles",
  "message": "...",
  "referenceType": "lab_order",
  "referenceId": 45
}

Response: 200 OK
```

### GET /communications/logs
Get communication logs
```
Query Parameters:
- patientId
- type (sms, email, whatsapp)
- status
- startDate, endDate

Response: 200 OK
```

---

## Users & Administration

### GET /users
Get list of users
```
Query Parameters:
- role
- isActive

Response: 200 OK
```

### POST /users
Create new user
```json
Request:
{
  "email": "doctor@clinic.com",
  "password": "securePassword123",
  "firstName": "Juan",
  "lastName": "Martinez",
  "role": "doctor",
  "medicalLicense": "MN 12345",
  "specialization": "Clínica Médica"
}

Response: 201 Created
```

### PUT /users/:id
Update user
```json
Response: 200 OK
```

### DELETE /users/:id
Deactivate user
```json
Response: 200 OK
```

### GET /system-settings
Get system settings
```json
Response: 200 OK
{
  "success": true,
  "data": {
    "clinicName": "Clínica Central",
    "clinicLogo": "https://...",
    "appointmentDuration": 30,
    "workingHours": {...}
  }
}
```

### PUT /system-settings
Update system settings
```json
Request:
{
  "clinicName": "Clínica Central",
  "appointmentDuration": 30
}

Response: 200 OK
```

---

## Error Responses

### Standard Error Format
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  }
}
```

### HTTP Status Codes
- `200 OK` - Success
- `201 Created` - Resource created
- `400 Bad Request` - Validation error
- `401 Unauthorized` - Authentication required
- `403 Forbidden` - Insufficient permissions
- `404 Not Found` - Resource not found
- `409 Conflict` - Resource conflict (e.g., duplicate)
- `422 Unprocessable Entity` - Business logic error
- `429 Too Many Requests` - Rate limit exceeded
- `500 Internal Server Error` - Server error

---

## Pagination Format

All list endpoints support pagination:
```
GET /patients?page=2&limit=20

Response:
{
  "success": true,
  "data": {
    "items": [...],
    "pagination": {
      "total": 150,
      "page": 2,
      "limit": 20,
      "totalPages": 8,
      "hasNext": true,
      "hasPrev": true
    }
  }
}
```

---

## Filtering & Sorting

Most list endpoints support:
```
GET /appointments?status=scheduled&sortBy=scheduledAt&sortOrder=asc&providerId=2

Supported operators (when applicable):
- eq (equal)
- ne (not equal)
- gt (greater than)
- lt (less than)
- gte (greater than or equal)
- lte (less than or equal)
- in (in array)
- contains (string contains)

Example:
GET /patients?age[gte]=18&age[lt]=65
```

---

## Rate Limiting

```
Rate Limit: 100 requests per minute per IP
Rate Limit Headers:
- X-RateLimit-Limit: 100
- X-RateLimit-Remaining: 95
- X-RateLimit-Reset: 1636545600 (Unix timestamp)

Response when exceeded:
HTTP 429 Too Many Requests
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests, please try again later"
  }
}
```

---

## Webhooks (Optional)

Webhook events for integration:
- `patient.created`
- `appointment.scheduled`
- `appointment.cancelled`
- `encounter.finalized`
- `invoice.paid`
- `lab_order.completed`

```
Webhook POST request format:
{
  "event": "appointment.scheduled",
  "timestamp": "2025-11-10T14:00:00Z",
  "data": {
    "appointmentId": 123,
    "patientId": 10,
    "providerId": 2,
    "scheduledAt": "2025-11-15T10:00:00Z"
  }
}
```

---

*API Documentation Version: 1.0*
*Last Updated: 2025-11-15*
*Base Version: v1*
