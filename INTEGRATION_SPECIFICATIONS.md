# Clinical Management System - Integration Specifications

## Overview

This document specifies all external integrations required for the clinical management system, including authentication methods, API endpoints, data formats, error handling, and implementation guidelines.

---

## 1. AFIP INTEGRATION (Argentine Tax Authority)

### 1.1 Overview

**Purpose:** Electronic invoicing (Factura Electrónica) compliance
**Service:** AFIP Web Services Facturación Electrónica (WSFE)
**Environment:** Homologation (testing) and Production

### 1.2 Prerequisites

#### Certificates Required
1. **X.509 Certificate** from AFIP
   - Request via AFIP portal
   - Valid for 2 years
   - Separate certificates for testing and production

2. **Private Key** (.key file)
   - Generated locally
   - Keep secure, never share

3. **CUIT** (tax ID) of the clinic
   - Used for all AFIP transactions

#### AFIP Portal Setup
1. Register in AFIP portal
2. Enable "Factura Electrónica" service
3. Assign points of sale (Puntos de Venta)
4. Generate and download certificate

### 1.3 Authentication Flow

```
Authentication Workflow:

1. LOGIN TICKET REQUEST
   ├─ Build XML request with:
   │  ├─ Service: wsfe
   │  ├─ CUIT
   │  └─ Timestamp
   ├─ Sign with private key
   ├─ Send to WSAA (Web Service de Autenticación y Autorización)
   └─ Receive Login Ticket (valid 12 hours)

2. LOGIN TICKET CONTAINS
├─ Token (authentication token)
├─ Sign (signature)
└─ Expiration time

3. USE TOKEN FOR ALL SUBSEQUENT REQUESTS
└─ Include token and sign in SOAP headers
```

#### Authentication Endpoint

**Testing:**
```
URL: https://wsaahomo.afip.gov.ar/ws/services/LoginCms
WSDL: https://wsaahomo.afip.gov.ar/ws/services/LoginCms?wsdl
```

**Production:**
```
URL: https://wsaa.afip.gov.ar/ws/services/LoginCms
WSDL: https://wsaa.afip.gov.ar/ws/services/LoginCms?wsdl
```

#### Sample Authentication Request

```xml
<?xml version="1.0" encoding="UTF-8"?>
<loginTicketRequest version="1.0">
  <header>
    <uniqueId>1234567890</uniqueId>
    <generationTime>2025-11-15T10:00:00-03:00</generationTime>
    <expirationTime>2025-11-15T22:00:00-03:00</expirationTime>
  </header>
  <service>wsfe</service>
</loginTicketRequest>
```

### 1.4 Invoice Submission (CAE Request)

#### WSFE Endpoints

**Testing:**
```
URL: https://wswhomo.afip.gov.ar/wsfev1/service.asmx
WSDL: https://wswhomo.afip.gov.ar/wsfev1/service.asmx?WSDL
```

**Production:**
```
URL: https://servicios1.afip.gov.ar/wsfev1/service.asmx
WSDL: https://servicios1.afip.gov.ar/wsfev1/service.asmx?WSDL
```

#### Key Methods

**FECAESolicitar** - Request CAE for invoice
```xml
<FECAESolicitar>
  <Auth>
    <Token>...</Token>
    <Sign>...</Sign>
    <Cuit>20123456789</Cuit>
  </Auth>
  <FeCAEReq>
    <FeCabReq>
      <CantReg>1</CantReg>
      <PtoVta>1</PtoVta>
      <CbteTipo>6</CbteTipo> <!-- 6=Factura B -->
    </FeCabReq>
    <FeDetReq>
      <FECAEDetRequest>
        <Concepto>1</Concepto> <!-- 1=Products, 2=Services, 3=Both -->
        <DocTipo>96</DocTipo> <!-- 96=DNI, 80=CUIT -->
        <DocNro>30123456</DocNro>
        <CbteDesde>123</CbteDesde>
        <CbteHasta>123</CbteHasta>
        <CbteFch>20251115</CbteFch> <!-- YYYYMMDD -->
        <ImpTotal>5000.00</ImpTotal>
        <ImpTotConc>0.00</ImpTotConc> <!-- Non-taxable amount -->
        <ImpNeto>5000.00</ImpNeto>
        <ImpOpEx>0.00</ImpOpEx> <!-- Exempt amount -->
        <ImpTrib>0.00</ImpTrib> <!-- Other taxes -->
        <ImpIVA>0.00</ImpIVA>
        <FchServDesde></FchServDesde> <!-- If service -->
        <FchServHasta></FchServHasta>
        <FchVtoPago></FchVtoPago>
        <MonId>PES</MonId> <!-- PES=Pesos -->
        <MonCotiz>1</MonCotiz>
      </FECAEDetRequest>
    </FeDetReq>
  </FeCAEReq>
</FECAESolicitar>
```

**Response:**
```xml
<FECAESolicitarResponse>
  <FECAESolicitarResult>
    <FeCabResp>
      <Resultado>A</Resultado> <!-- A=Approved, R=Rejected -->
      <Reproceso>N</Reproceso>
    </FeCabResp>
    <FeDetResp>
      <FECAEDetResponse>
        <Concepto>1</Concepto>
        <DocTipo>96</DocTipo>
        <DocNro>30123456</DocNro>
        <CbteDesde>123</CbteDesde>
        <CbteHasta>123</CbteHasta>
        <CbteFch>20251115</CbteFch>
        <Resultado>A</Resultado>
        <CAE>12345678901234</CAE> <!-- 14 digits -->
        <CAEFchVto>20251125</CAEFchVto> <!-- CAE expiration -->
      </FECAEDetResponse>
    </FeDetResp>
  </FECAESolicitarResult>
</FECAESolicitarResponse>
```

**Other Useful Methods:**
- `FECompUltimoAutorizado` - Get last authorized invoice number
- `FEParamGetTiposCbte` - Get invoice types
- `FEParamGetTiposDoc` - Get document types
- `FECompConsultar` - Query invoice by CAE

### 1.5 QR Code Generation

**Format:** QR code must include invoice verification URL

```
QR Code URL Format:
https://www.afip.gob.ar/fe/qr/?p=<base64_encoded_data>

Data to encode (JSON):
{
  "ver": 1,
  "fecha": "2025-11-15",
  "cuit": 20123456789,
  "ptoVta": 1,
  "tipoCmp": 6,
  "nroCmp": 123,
  "importe": 5000.00,
  "moneda": "PES",
  "ctz": 1,
  "tipoDocRec": 96,
  "nroDocRec": 30123456,
  "tipoCodAut": "E",
  "codAut": 12345678901234
}

Steps:
1. Build JSON object
2. Convert to JSON string
3. Base64 encode
4. Append to URL
5. Generate QR code image
```

### 1.6 Error Handling

**Common AFIP Errors:**

| Error Code | Description | Action |
|------------|-------------|--------|
| 10000 | Authentication failed | Re-authenticate, check certificate |
| 10016 | Invoice number already exists | Get last number, increment |
| 602 | Invalid date format | Use YYYYMMDD format |
| 1502 | Point of sale not authorized | Verify punto de venta in AFIP |
| 10048 | Duplicate CAE request | Query existing CAE |

**Retry Logic:**
```
On error:
1. Check if error is recoverable
   - 10000, 10001 (auth): Re-authenticate and retry
   - 10048 (duplicate): Query existing CAE
   - 10016 (number conflict): Fetch correct number and retry

2. If recoverable: Retry up to 3 times with 5-second delay

3. If not recoverable: Log error, notify user

4. Fallback: Allow offline mode
   - Save invoice draft
   - Manually request CAE later
   - Sync when AFIP available
```

### 1.7 Testing Strategy

**Homologation Environment:**
```
1. Use testing CUIT: Get from AFIP
2. Use testing certificate
3. Test all invoice types: A, B, C
4. Test credit/debit notes
5. Verify CAE generation
6. Verify QR code generation
7. Test error scenarios
```

**Production Cutover:**
```
1. Obtain production certificate
2. Update configuration with production URLs
3. Test with real invoices (internal first)
4. Monitor for errors
5. Keep homologation access for testing
```

---

## 2. SMS INTEGRATION (Twilio or similar)

### 2.1 Overview

**Provider:** Twilio (recommended) or local Argentine SMS gateway
**Use Cases:** Appointment reminders, lab results notifications, payment reminders

### 2.2 Twilio Integration

#### Setup
```
1. Create Twilio account: https://www.twilio.com
2. Obtain credentials:
   - Account SID
   - Auth Token
   - Twilio Phone Number (Argentine number recommended)
3. Purchase phone number with SMS capability
4. Fund account (pay-as-you-go)
```

#### API Endpoint
```
Base URL: https://api.twilio.com/2010-04-01
Method: POST /Accounts/{AccountSid}/Messages.json
Authentication: Basic Auth (Account SID + Auth Token)
```

#### Send SMS Request

```http
POST https://api.twilio.com/2010-04-01/Accounts/ACXXXXXXXX/Messages.json
Authorization: Basic base64(AccountSid:AuthToken)
Content-Type: application/x-www-form-urlencoded

Body:
To=+5491112345678
From=+5491187654321
Body=Recordatorio: Tiene turno mañana 15/11 a las 10:00 con Dr. Martinez. Clínica Central.
StatusCallback=https://yourdomain.com/sms/status
```

#### Response
```json
{
  "sid": "SMXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "date_created": "Wed, 15 Nov 2025 10:00:00 +0000",
  "date_updated": "Wed, 15 Nov 2025 10:00:00 +0000",
  "date_sent": null,
  "account_sid": "ACXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "to": "+5491112345678",
  "from": "+5491187654321",
  "messaging_service_sid": null,
  "body": "Recordatorio: ...",
  "status": "queued",
  "num_segments": "1",
  "num_media": "0",
  "direction": "outbound-api",
  "api_version": "2010-04-01",
  "price": null,
  "price_unit": "USD",
  "error_code": null,
  "error_message": null,
  "uri": "/2010-04-01/Accounts/AC.../Messages/SM...",
  "subresource_uris": {
    "media": "/2010-04-01/Accounts/AC.../Messages/SM.../Media"
  }
}
```

#### Status Callback

Twilio sends webhook updates to `StatusCallback` URL:

```http
POST https://yourdomain.com/sms/status
Content-Type: application/x-www-form-urlencoded

MessageSid=SMXXXXXXXX
MessageStatus=delivered
To=+5491112345678
From=+5491187654321
```

**Status values:**
- `queued` - Message queued for sending
- `sent` - Message sent to carrier
- `delivered` - Message delivered to recipient
- `undelivered` - Message failed to deliver
- `failed` - Message failed

#### Error Handling

```
Common Errors:
- 21211: Invalid 'To' phone number
- 21408: Permission to send denied (e.g., landline)
- 21610: Message blocked (unsubscribed)
- 30007: Message filtered as spam

Handle:
1. Validate phone number before sending
2. Track unsubscribe status
3. Retry on temporary failures (rate limit)
4. Log and notify on permanent failures
```

### 2.3 Argentine SMS Alternatives

**Local Providers:**
- **SMS Masivos:** https://www.smsmasivos.com.ar
- **Nexmo/Vonage:** Better rates for Argentina
- **Infobip:** Enterprise solution

**Advantages of local providers:**
- Better rates for Argentine numbers
- Local support
- Better deliverability

---

## 3. EMAIL INTEGRATION (SendGrid or similar)

### 3.1 Overview

**Provider:** SendGrid (recommended), Amazon SES, or Mailgun
**Use Cases:** Appointment confirmations, lab results, invoices, prescriptions

### 3.2 SendGrid Integration

#### Setup
```
1. Create SendGrid account: https://sendgrid.com
2. Verify sender email/domain
3. Obtain API Key
4. Configure SPF and DKIM records (for deliverability)
```

#### API Endpoint
```
Base URL: https://api.sendgrid.com/v3
Method: POST /mail/send
Authentication: Bearer Token (API Key)
```

#### Send Email Request

```http
POST https://api.sendgrid.com/v3/mail/send
Authorization: Bearer SG.XXXXXXXXXXXXXXXXXXXX
Content-Type: application/json

{
  "personalizations": [
    {
      "to": [
        {
          "email": "patient@example.com",
          "name": "Carlos Rodriguez"
        }
      ],
      "subject": "Confirmación de Turno - Clínica Central"
    }
  ],
  "from": {
    "email": "noreply@clinicacentral.com",
    "name": "Clínica Central"
  },
  "reply_to": {
    "email": "info@clinicacentral.com",
    "name": "Clínica Central"
  },
  "content": [
    {
      "type": "text/plain",
      "value": "Su turno ha sido confirmado para el 15/11/2025 a las 10:00."
    },
    {
      "type": "text/html",
      "value": "<html><body><h1>Turno Confirmado</h1><p>Su turno ha sido confirmado para el <strong>15/11/2025 a las 10:00</strong> con Dr. Martinez.</p></body></html>"
    }
  ],
  "attachments": [
    {
      "content": "BASE64_ENCODED_PDF",
      "type": "application/pdf",
      "filename": "receta.pdf",
      "disposition": "attachment"
    }
  ],
  "categories": ["appointment_confirmation"],
  "custom_args": {
    "appointment_id": "123",
    "patient_id": "456"
  }
}
```

#### Response
```json
{
  "message": "success"
}
```

**Status Code: 202** (Accepted)

#### Webhooks (Event Tracking)

Configure webhook to receive delivery events:

```http
POST https://yourdomain.com/email/webhook
Content-Type: application/json

[
  {
    "email": "patient@example.com",
    "timestamp": 1636977600,
    "event": "delivered",
    "category": ["appointment_confirmation"],
    "sg_event_id": "XXXXXXXX",
    "sg_message_id": "XXXXXXXX.filter0001.XXXXX.XXXX.0"
  }
]
```

**Event Types:**
- `processed` - Email processed by SendGrid
- `delivered` - Email delivered to recipient
- `open` - Email opened by recipient
- `click` - Link in email clicked
- `bounce` - Email bounced
- `dropped` - Email dropped (invalid, unsubscribed)
- `spam` - Marked as spam
- `unsubscribe` - Recipient unsubscribed

#### Unsubscribe Management

```
SendGrid manages unsubscribes automatically:

1. Add unsubscribe link in email (required)
2. SendGrid adds to suppression list
3. Future emails to unsubscribed address are blocked

Manual check before sending:
GET https://api.sendgrid.com/v3/suppression/unsubscribes/{email}
```

---

## 4. WHATSAPP INTEGRATION (WhatsApp Business API)

### 4.1 Overview

**Provider:** Twilio WhatsApp Business API, 360Dialog, or Meta WhatsApp Business API
**Use Cases:** Appointment reminders, two-way messaging (optional)

### 4.2 Twilio WhatsApp Integration

#### Setup
```
1. Twilio account (same as SMS)
2. Request WhatsApp Business Profile approval
3. Submit message templates for approval (24-48 hours)
4. Obtain WhatsApp-enabled phone number
```

#### Message Templates

**Templates must be pre-approved by WhatsApp**

Example template:
```
Name: appointment_reminder
Language: es
Category: appointment_update

Template:
Hola {{1}}, le recordamos su turno el {{2}} a las {{3}} con {{4}}. Clínica Central.
```

After approval, use template in API calls.

#### Send WhatsApp Message

```http
POST https://api.twilio.com/2010-04-01/Accounts/ACXXXXXXXX/Messages.json
Authorization: Basic base64(AccountSid:AuthToken)
Content-Type: application/x-www-form-urlencoded

To=whatsapp:+5491112345678
From=whatsapp:+5491187654321
Body=Hola Carlos, le recordamos su turno el 15/11/2025 a las 10:00 con Dr. Martinez. Clínica Central.
```

For template-based messages:
```
ContentSid=HXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
ContentVariables={"1":"Carlos","2":"15/11/2025","3":"10:00","4":"Dr. Martinez"}
```

#### Two-Way Messaging

**Receiving Messages:**

Set webhook URL to receive incoming messages:

```http
POST https://yourdomain.com/whatsapp/incoming
Content-Type: application/x-www-form-urlencoded

From=whatsapp:+5491112345678
To=whatsapp:+5491187654321
Body=Confirmo
MessageSid=SMXXXXXXXX
```

**Reply:**
```
Parse incoming message
If confirmation keywords: Mark appointment as confirmed
Reply with confirmation message
```

**24-hour window rule:**
- Can send messages freely within 24 hours of last patient message
- After 24 hours: Must use pre-approved templates only

---

## 5. PAYMENT GATEWAY INTEGRATION (Mercado Pago)

### 5.1 Overview

**Provider:** Mercado Pago (most popular in Argentina)
**Use Cases:** Online payments, card processing, QR code payments

### 5.2 Mercado Pago Integration

#### Setup
```
1. Create Mercado Pago account: https://www.mercadopago.com.ar
2. Register as business
3. Obtain credentials:
   - Public Key (for frontend)
   - Access Token (for backend)
4. Choose integration type:
   - Web Checkout (redirect)
   - Checkout Pro (iframe)
   - Payment Link
```

#### Create Payment

**Method: Payment Link (Simplest)**

```http
POST https://api.mercadopago.com/checkout/preferences
Authorization: Bearer ACCESS_TOKEN
Content-Type: application/json

{
  "items": [
    {
      "title": "Consulta médica",
      "description": "Consulta con Dr. Martinez - Factura #123",
      "quantity": 1,
      "currency_id": "ARS",
      "unit_price": 5000.00
    }
  ],
  "payer": {
    "name": "Carlos",
    "surname": "Rodriguez",
    "email": "carlos@example.com",
    "phone": {
      "area_code": "11",
      "number": "12345678"
    }
  },
  "back_urls": {
    "success": "https://yourdomain.com/payment/success",
    "failure": "https://yourdomain.com/payment/failure",
    "pending": "https://yourdomain.com/payment/pending"
  },
  "notification_url": "https://yourdomain.com/payment/webhook",
  "external_reference": "invoice_123",
  "auto_return": "approved",
  "statement_descriptor": "CLINICA CENTRAL"
}
```

**Response:**
```json
{
  "id": "123456789",
  "init_point": "https://www.mercadopago.com.ar/checkout/v1/redirect?pref_id=123456789",
  "sandbox_init_point": "https://sandbox.mercadopago.com.ar/checkout/v1/redirect?pref_id=123456789"
}
```

**Redirect user to `init_point` URL for payment**

#### Webhook (IPN - Instant Payment Notification)

Mercado Pago sends payment status to `notification_url`:

```http
POST https://yourdomain.com/payment/webhook
Content-Type: application/json

{
  "action": "payment.created",
  "api_version": "v1",
  "data": {
    "id": "123456789"
  },
  "date_created": "2025-11-15T10:00:00Z",
  "id": 123456789,
  "live_mode": true,
  "type": "payment",
  "user_id": "987654321"
}
```

**On receiving webhook:**
```
1. Get payment_id from data.id
2. Query payment details:
   GET https://api.mercadopago.com/v1/payments/{id}
   Authorization: Bearer ACCESS_TOKEN

3. Verify payment status:
   - approved: Update invoice as paid
   - pending: Wait for confirmation
   - rejected: Notify patient

4. Update invoice record with:
   - Payment ID
   - Transaction details
   - Status
```

#### Query Payment Status

```http
GET https://api.mercadopago.com/v1/payments/123456789
Authorization: Bearer ACCESS_TOKEN
```

**Response:**
```json
{
  "id": 123456789,
  "status": "approved",
  "status_detail": "accredited",
  "transaction_amount": 5000.00,
  "currency_id": "ARS",
  "payer": {
    "email": "carlos@example.com",
    "identification": {
      "type": "DNI",
      "number": "30123456"
    }
  },
  "payment_method_id": "visa",
  "payment_type_id": "credit_card",
  "transaction_details": {
    "net_received_amount": 4850.00,
    "total_paid_amount": 5000.00,
    "overpaid_amount": 0,
    "installment_amount": 5000.00
  },
  "date_approved": "2025-11-15T10:05:00.000-04:00",
  "external_reference": "invoice_123"
}
```

**Payment Status Values:**
- `approved` - Payment approved
- `pending` - Pending (e.g., bank transfer pending)
- `in_process` - Being processed
- `rejected` - Payment rejected
- `cancelled` - Payment cancelled
- `refunded` - Payment refunded
- `charged_back` - Chargeback

---

## 6. LABORATORY INTEGRATION (HL7)

### 6.1 Overview

**Protocol:** HL7 v2.x (Health Level Seven)
**Use Cases:** Receive lab results electronically from laboratory systems

### 6.2 HL7 Message Types

**ORM - Order Message** (Send lab orders to lab)
**ORU - Observation Result** (Receive lab results from lab)

### 6.3 Sample HL7 ORU Message (Lab Results)

```
MSH|^~\&|LAB|LABORATORY|CLINIC|CLINIC_SYSTEM|20251115100000||ORU^R01|MSG00001|P|2.5
PID|1||P-00123||Rodriguez^Carlos||19800515|M|||Av Corrientes 1234^^Buenos Aires^BA^1000^AR||011-1234-5678
OBR|1|ORD123|RES456|CBC^Hemograma Completo||20251115090000|20251115095000|||||||| Dr.Martinez
OBX|1|NM|HGB^Hemoglobina||14.5|g/dL|12-16|N|||F|||20251115100000
OBX|2|NM|HCT^Hematocrito||42|%|37-47|N|||F|||20251115100000
OBX|3|NM|WBC^Glóbulos Blancos||12.5|10^3/uL|4-11|H|||F|||20251115100000
OBX|4|NM|PLT^Plaquetas||250|10^3/uL|150-400|N|||F|||20251115100000
```

**Segments:**
- **MSH** (Message Header): Metadata
- **PID** (Patient Identification): Patient info
- **OBR** (Observation Request): Order info
- **OBX** (Observation/Result): Individual test results

### 6.4 Parsing HL7 Messages

```
Algorithm:

1. Split message by newline into segments
2. For each segment:
   - Split by | into fields
   - Field 0 = segment type (MSH, PID, OBR, OBX)
   - Fields 1+ = data

3. Parse PID segment:
   - Field 3 = Patient ID
   - Field 5 = Patient Name (Last^First)
   - Field 7 = Date of Birth (YYYYMMDD)

4. Parse OBR segment:
   - Field 2 = Order Number
   - Field 4 = Universal Service ID (test code^test name)
   - Field 7 = Observation Date/Time

5. Parse OBX segments (multiple):
   - Field 2 = Value Type (NM=numeric, ST=string)
   - Field 3 = Test Code^Test Name
   - Field 5 = Result Value
   - Field 6 = Units
   - Field 7 = Reference Range
   - Field 8 = Abnormal Flags (N=normal, L=low, H=high)

6. Match to existing lab order:
   - Use order number (OBR-2)
   - Or patient ID + order date

7. Insert results into database

8. Flag abnormal results

9. Notify doctor
```

### 6.5 Sending HL7 ORM (Lab Orders)

```
MSH|^~\&|CLINIC|CLINIC_SYSTEM|LAB|LABORATORY|20251115100000||ORM^O01|MSG00001|P|2.5
PID|1||P-00123||Rodriguez^Carlos||19800515|M|||Av Corrientes 1234^^Buenos Aires^BA^1000^AR||011-1234-5678
ORC|NW|ORD123|||||||20251115100000
OBR|1|ORD123||CBC^Hemograma Completo||20251115100000||||||Dr.Martinez
```

**Segments:**
- **ORC** (Order Control): Order status (NW=new order)
- **OBR** (Observation Request): Test being ordered

### 6.6 Integration Methods

**Option 1: File-based**
- Export HL7 messages to file
- Lab polls folder and imports
- Lab exports results to file
- System polls and imports

**Option 2: TCP/IP (MLLP - Minimal Lower Layer Protocol)**
- Real-time message exchange
- Connect to lab system via TCP socket
- Send/receive HL7 messages wrapped in MLLP framing

**Option 3: Web Service / REST API**
- Modern approach
- Lab provides REST API
- Convert HL7 to JSON or use HL7 directly
- HTTP-based communication

---

## 7. BACKUP & STORAGE INTEGRATION

### 7.1 Amazon S3 (or compatible)

**Use Cases:** Document storage, database backups, image storage

#### Setup
```
1. Create AWS account
2. Create S3 bucket: clinic-documents-prod
3. Set bucket permissions (private)
4. Obtain IAM credentials:
   - Access Key ID
   - Secret Access Key
5. Set up lifecycle policies (optional)
```

#### Upload File to S3

```python
import boto3

s3_client = boto3.client(
    's3',
    aws_access_key_id='AKIAXXXXXXXX',
    aws_secret_access_key='XXXXXXXXXXXXXXXX',
    region_name='us-east-1'
)

# Upload file
s3_client.upload_file(
    '/path/to/local/file.pdf',
    'clinic-documents-prod',
    'patients/123/prescriptions/rx-001.pdf'
)

# Generate pre-signed URL (temporary access)
url = s3_client.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'clinic-documents-prod', 'Key': 'patients/123/prescriptions/rx-001.pdf'},
    ExpiresIn=3600  # 1 hour
)
```

#### Folder Structure

```
clinic-documents-prod/
├── patients/
│   ├── {patient_id}/
│   │   ├── documents/
│   │   │   └── {filename}
│   │   ├── prescriptions/
│   │   │   └── {filename}
│   │   ├── lab_results/
│   │   │   └── {filename}
│   │   └── images/
│   │       └── {filename}
├── invoices/
│   └── {year}/{month}/{invoice_number}.pdf
└── backups/
    └── {date}/database_backup.sql.gz
```

---

## 8. PDF GENERATION

### 8.1 Overview

**Use Cases:** Prescriptions, invoices, lab orders, reports
**Libraries:** wkhtmltopdf, Puppeteer, PDFKit, or similar

### 8.2 Invoice PDF Requirements

**Must Include:**
- Clinic header (logo, name, address, CUIT)
- Invoice type (A, B, C)
- Point of sale + invoice number
- Invoice date
- CAE (14 digits)
- CAE expiration date
- Patient/customer details
- Line items with descriptions and amounts
- Subtotal, taxes, total
- QR code (AFIP requirement)
- Payment details (if paid)

**Layout:** A4 size, portrait orientation

### 8.3 Prescription PDF Requirements

**Must Include:**
- Doctor name, license number, specialty
- Clinic name and address
- Date
- Prescription number (Rx #)
- Patient name, age, DNI
- Medications with dosage, frequency, duration, quantity
- Doctor signature (digital or scanned)
- Clinic stamp (optional)

**Format:** Standard prescription pad format

---

## 9. ERROR HANDLING & LOGGING

### 9.1 Integration Error Handling Strategy

```
For all external integrations:

1. TIMEOUT HANDLING
   - Set reasonable timeouts (e.g., 30 seconds)
   - If timeout: Retry with exponential backoff
   - Max retries: 3

2. NETWORK ERRORS
   - Connection refused, DNS errors, etc.
   - Log error details
   - Retry automatically

3. API ERRORS (4xx, 5xx)
   - 400 Bad Request: Validation error - fix and retry
   - 401 Unauthorized: Re-authenticate
   - 429 Rate Limit: Back off and retry after delay
   - 500 Server Error: Retry later
   - 503 Service Unavailable: Retry with backoff

4. LOGGING
   - Log all integration requests and responses
   - Include: timestamp, endpoint, request body, response, status
   - Use structured logging (JSON)
   - Store logs for debugging and audit

5. MONITORING
   - Track integration success/failure rates
   - Alert on high failure rates
   - Monitor response times

6. FALLBACK / OFFLINE MODE
   - If integration unavailable: allow manual workarounds
   - Queue requests for later retry
   - Sync when service restored
```

### 9.2 Webhook Security

```
For all incoming webhooks (Twilio, SendGrid, Mercado Pago):

1. VERIFY SIGNATURE
   - Each provider sends signature in header
   - Verify using shared secret
   - Reject if invalid signature

Example (Twilio):
from twilio.request_validator import RequestValidator

validator = RequestValidator(auth_token)
signature = request.headers.get('X-Twilio-Signature')
url = 'https://yourdomain.com/sms/status'
params = request.form.to_dict()

if not validator.validate(url, params, signature):
    return 'Invalid signature', 403

2. IDEMPOTENCY
   - Process each webhook only once
   - Use unique ID (e.g., message SID, payment ID)
   - Check if already processed before handling
   - Store processed IDs in database or cache

3. RETRY HANDLING
   - Webhooks may be sent multiple times
   - Respond with 200 OK quickly
   - Process asynchronously
   - Return success even if already processed
```

---

## 10. ENVIRONMENT CONFIGURATION

### 10.1 Environment Variables

```env
# AFIP
AFIP_ENVIRONMENT=testing # or production
AFIP_CUIT=20123456789
AFIP_CERTIFICATE_PATH=/path/to/cert.pem
AFIP_PRIVATE_KEY_PATH=/path/to/private.key
AFIP_WSAA_URL=https://wsaahomo.afip.gov.ar/ws/services/LoginCms
AFIP_WSFE_URL=https://wswhomo.afip.gov.ar/wsfev1/service.asmx
AFIP_PUNTO_VENTA=1

# Twilio
TWILIO_ACCOUNT_SID=ACXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
TWILIO_AUTH_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
TWILIO_PHONE_NUMBER=+5491187654321
TWILIO_WHATSAPP_NUMBER=whatsapp:+5491187654321

# SendGrid
SENDGRID_API_KEY=SG.XXXXXXXXXXXXXXXXXXXX
SENDGRID_FROM_EMAIL=noreply@clinicacentral.com
SENDGRID_FROM_NAME=Clínica Central

# Mercado Pago
MERCADOPAGO_ACCESS_TOKEN=APP_USR-XXXXXXXXXXXXXXXXXXXX
MERCADOPAGO_PUBLIC_KEY=APP_USR-XXXXXXXXXXXXXXXXXXXX
MERCADOPAGO_ENVIRONMENT=testing # or production

# AWS S3
AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXXXXXXX
AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
AWS_S3_BUCKET=clinic-documents-prod
AWS_REGION=us-east-1

# Application URLs
APP_BASE_URL=https://yourdomain.com
WEBHOOK_BASE_URL=https://yourdomain.com
```

### 10.2 Testing vs Production

**Maintain separate configurations:**

```
Testing:
- AFIP Homologation environment
- Twilio Test Credentials (or test mode)
- SendGrid Sandbox
- Mercado Pago Sandbox
- Separate S3 bucket
- Use test phone numbers and emails

Production:
- AFIP Production environment
- Real Twilio credentials
- Real SendGrid account
- Mercado Pago Production
- Production S3 bucket
- Real phone numbers and emails
```

---

*Integration Specifications Version: 1.0*
*Last Updated: 2025-11-15*
*Complete integration guide for implementation*
