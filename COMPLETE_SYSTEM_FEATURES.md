# Clinical Management System - Complete System Features

## Overview

This document covers image/document management, import/export capabilities, advanced API integrations, AI/ML features, and the complete patient portal specification.

---

# PART 1: IMAGE & DOCUMENT MANAGEMENT

## 1. DOCUMENT MANAGEMENT SYSTEM

### 1.1 Database Schema

```sql
CREATE TABLE documents (
    id                      BIGSERIAL PRIMARY KEY,
    uuid                    UUID UNIQUE NOT NULL DEFAULT gen_random_uuid(),

    -- Classification
    document_type           VARCHAR(100) NOT NULL,
                            -- 'medical_image', 'lab_result', 'prescription', 'consent_form',
                            -- 'insurance_card', 'id_document', 'referral', 'other'

    category                VARCHAR(100), -- Custom categorization
    sub_category            VARCHAR(100),

    -- Association
    patient_id              BIGINT REFERENCES patients(id),
    encounter_id            BIGINT REFERENCES encounters(id),
    lab_order_id            BIGINT REFERENCES lab_orders(id),
    prescription_id         BIGINT REFERENCES prescriptions(id),

    -- File information
    title                   VARCHAR(500) NOT NULL,
    description             TEXT,
    file_name               VARCHAR(500) NOT NULL,
    original_file_name      VARCHAR(500),
    file_path               VARCHAR(1000) NOT NULL, -- S3 path or local path
    file_size               BIGINT, -- bytes
    mime_type               VARCHAR(100),
    file_extension          VARCHAR(20),

    -- Image-specific
    is_medical_image        BOOLEAN DEFAULT false,
    image_modality          VARCHAR(50), -- 'xray', 'ct', 'mri', 'ultrasound', 'photo'
    image_body_part         VARCHAR(200),
    image_width             INTEGER,
    image_height            INTEGER,

    -- DICOM-specific (medical imaging)
    is_dicom                BOOLEAN DEFAULT false,
    dicom_metadata          JSONB,
    /* Example DICOM metadata:
    {
      "patient_name": "Rodriguez, Carlos",
      "study_date": "20251115",
      "modality": "CT",
      "body_part": "Chest",
      "series_description": "Chest CT Angio",
      "instance_number": 1,
      "sop_instance_uid": "1.2.840.113..."
    }
    */

    -- Security
    is_confidential         BOOLEAN DEFAULT true,
    encryption_key_id       VARCHAR(100), -- Reference to encryption key
    file_hash               VARCHAR(64), -- SHA-256 for verification

    -- Versioning
    version                 INTEGER DEFAULT 1,
    parent_document_id      BIGINT REFERENCES documents(id),

    -- Metadata
    document_date           DATE, -- Date of document creation/scan
    tags                    TEXT[], -- Searchable tags

    -- Status
    status                  VARCHAR(50) DEFAULT 'active',
                            -- 'active', 'archived', 'deleted', 'pending_review'

    review_status           VARCHAR(50), -- 'pending', 'approved', 'rejected'
    reviewed_by             BIGINT REFERENCES users(id),
    reviewed_at             TIMESTAMP,

    -- Tracking
    uploaded_by             BIGINT REFERENCES users(id),
    uploaded_at             TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at              TIMESTAMP,

    INDEX idx_documents_patient (patient_id),
    INDEX idx_documents_encounter (encounter_id),
    INDEX idx_documents_type (document_type),
    INDEX idx_documents_uploaded (uploaded_at),
    INDEX idx_documents_tags USING GIN(tags)
);

-- Document access log (audit trail)
CREATE TABLE document_access_log (
    id                      BIGSERIAL PRIMARY KEY,
    document_id             BIGINT NOT NULL REFERENCES documents(id),
    user_id                 BIGINT NOT NULL REFERENCES users(id),
    action                  VARCHAR(50) NOT NULL, -- 'view', 'download', 'print', 'share'
    ip_address              VARCHAR(50),
    user_agent              TEXT,
    accessed_at             TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_doc_access_document (document_id),
    INDEX idx_doc_access_user (user_id),
    INDEX idx_doc_access_date (accessed_at)
);

-- Document sharing
CREATE TABLE document_shares (
    id                      BIGSERIAL PRIMARY KEY,
    document_id             BIGINT NOT NULL REFERENCES documents(id),

    -- Recipient
    shared_with_type        VARCHAR(50), -- 'user', 'patient', 'external'
    shared_with_user_id     BIGINT REFERENCES users(id),
    shared_with_patient_id  BIGINT REFERENCES patients(id),
    shared_with_email       VARCHAR(255), -- For external shares

    -- Permissions
    can_view                BOOLEAN DEFAULT true,
    can_download            BOOLEAN DEFAULT false,
    can_print               BOOLEAN DEFAULT false,

    -- Expiration
    expires_at              TIMESTAMP,

    -- Share link
    share_token             VARCHAR(100) UNIQUE,
    access_count            INTEGER DEFAULT 0,
    max_access_count        INTEGER,

    shared_by               BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_doc_shares_document (document_id),
    INDEX idx_doc_shares_token (share_token)
);
```

### 1.2 Document Upload & Processing

```javascript
/**
 * Upload and process document
 */
async function uploadDocument(file, metadata) {
  // 1. Validate file
  const validation = validateFile(file);
  if (!validation.valid) {
    throw new Error(validation.error);
  }

  // 2. Generate unique filename
  const uniqueFilename = `${uuid()}_${sanitizeFilename(file.originalname)}`;

  // 3. Determine storage path
  const storagePath = buildStoragePath(metadata.patient_id, metadata.document_type);

  // 4. Scan for viruses (optional but recommended)
  await virusScan(file);

  // 5. Extract metadata
  const fileMetadata = await extractMetadata(file);

  // 6. If image, generate thumbnails
  let thumbnails = [];
  if (isImage(file)) {
    thumbnails = await generateThumbnails(file);
  }

  // 7. If DICOM, extract DICOM tags
  let dicomMetadata = null;
  if (isDICOM(file)) {
    dicomMetadata = await extractDICOMMetadata(file);
  }

  // 8. Encrypt if confidential
  let encryptedFile = file;
  let encryptionKeyId = null;
  if (metadata.is_confidential) {
    const encrypted = await encryptFile(file);
    encryptedFile = encrypted.file;
    encryptionKeyId = encrypted.keyId;
  }

  // 9. Upload to storage (S3 or local)
  const uploadResult = await storageService.upload({
    file: encryptedFile,
    path: storagePath,
    filename: uniqueFilename
  });

  // 10. Calculate file hash for integrity
  const fileHash = await calculateSHA256(file);

  // 11. Save to database
  const document = await Document.create({
    ...metadata,
    file_name: uniqueFilename,
    original_file_name: file.originalname,
    file_path: uploadResult.path,
    file_size: file.size,
    mime_type: file.mimetype,
    file_extension: path.extname(file.originalname),
    file_hash: fileHash,
    encryption_key_id: encryptionKeyId,
    dicom_metadata: dicomMetadata,
    image_width: fileMetadata.width,
    image_height: fileMetadata.height,
    uploaded_by: currentUser.id
  });

  // 12. If has OCR capabilities, extract text
  if (shouldOCR(file)) {
    await queueOCRJob(document.id);
  }

  // 13. Trigger workflow if needed
  if (metadata.requires_review) {
    await workflowEngine.trigger('document_review', {
      entity_type: 'document',
      entity_id: document.id
    });
  }

  return document;
}

function validateFile(file) {
  const maxSize = 50 * 1024 * 1024; // 50MB
  const allowedTypes = [
    'image/jpeg', 'image/png', 'image/gif', 'image/tiff',
    'application/pdf',
    'application/dicom',
    'application/msword',
    'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
  ];

  if (file.size > maxSize) {
    return { valid: false, error: 'File size exceeds 50MB limit' };
  }

  if (!allowedTypes.includes(file.mimetype)) {
    return { valid: false, error: 'File type not allowed' };
  }

  return { valid: true };
}
```

### 1.3 DICOM Support

```javascript
/**
 * DICOM viewer integration
 */
const dicomViewer = {
  // Parse DICOM file
  async parseDICOM(filePath) {
    const dicomData = await dicomParser.parseDicom(filePath);

    return {
      patientName: dicomData.string('x00100010'),
      patientID: dicomData.string('x00100020'),
      studyDate: dicomData.string('x00080020'),
      modality: dicomData.string('x00080060'),
      bodyPart: dicomData.string('x00180015'),
      seriesDescription: dicomData.string('x0008103e'),
      instanceNumber: dicomData.uint16('x00200013'),
      imageData: dicomData.elements.x7fe00010.value
    };
  },

  // Generate DICOM preview
  async generatePreview(dicomFile) {
    const canvas = createCanvas(512, 512);
    const context = canvas.getContext('2d');

    const imageData = await this.parseDICOM(dicomFile);
    // Render DICOM image to canvas
    renderDICOMImage(context, imageData);

    return canvas.toBuffer('image/png');
  },

  // Viewer configuration
  viewerConfig: {
    tools: ['zoom', 'pan', 'window_level', 'length', 'angle', 'rectangle'],
    initialViewport: {
      scale: 1.0,
      translation: { x: 0, y: 0 }
    }
  }
};
```

### 1.4 Document Viewer UI

```
┌──────────────────────────────────────────────────────────────────┐
│ Document Viewer - Chest X-Ray                              [×]   │
├──────────────────────────────────────────────────────────────────┤
│ Patient: Carlos Rodriguez (DNI: 30123456)                        │
│ Date: Nov 15, 2025    Type: Medical Image - X-Ray               │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │                                                            │ │
│ │                    [Image Preview]                         │ │
│ │                                                            │ │
│ │                 Chest X-Ray - PA View                      │ │
│ │                                                            │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Tools:                                                           │
│ [🔍 Zoom] [↔️ Pan] [💡 Brightness] [📏 Measure] [↻ Rotate]     │
│                                                                  │
│ Metadata:                                                        │
│ ├─ Modality: X-Ray                                              │
│ ├─ Body Part: Chest                                             │
│ ├─ View: PA (Posterior-Anterior)                                │
│ ├─ Date Taken: Nov 15, 2025 10:30                              │
│ ├─ Ordering Physician: Dr. Martinez                             │
│ └─ File Size: 2.4 MB                                            │
│                                                                  │
│ Annotations:                                                     │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Dr. Martinez - Nov 15, 14:00                               │ │
│ │ Normal cardiac silhouette. No acute findings.              │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ [Add Annotation] [Download] [Print] [Share] [Compare]          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.5 Scanning Integration

```javascript
/**
 * Scanner integration (for ID cards, insurance cards, consent forms)
 */
const scannerIntegration = {
  // TWAIN/WIA scanner support
  async scan(options = {}) {
    const scanConfig = {
      resolution: options.resolution || 300, // DPI
      colorMode: options.colorMode || 'color', // 'color', 'grayscale', 'bw'
      paperSize: options.paperSize || 'A4',
      duplex: options.duplex || false,
      outputFormat: 'pdf' // or 'jpeg', 'png'
    };

    // Trigger scan via scanner API
    const scannedFile = await scannerAPI.scan(scanConfig);

    // Auto-detect document type
    const documentType = await detectDocumentType(scannedFile);

    // Extract text via OCR
    const ocrText = await performOCR(scannedFile);

    // Auto-populate metadata
    const metadata = await extractMetadataFromOCR(ocrText, documentType);

    return {
      file: scannedFile,
      documentType,
      ocrText,
      metadata
    };
  },

  // Detect document type from image
  async detectDocumentType(imageFile) {
    // Use AI/ML model or heuristics
    const features = await extractImageFeatures(imageFile);

    if (features.hasBarcode && features.hasPhoto) {
      return 'id_document';
    } else if (features.hasInsuranceLogo) {
      return 'insurance_card';
    } else if (features.hasLabHeader) {
      return 'lab_result';
    }

    return 'other';
  }
};
```

---

# PART 2: IMPORT & EXPORT

## 2. DATA IMPORT SYSTEM

### 2.1 Import Templates

```sql
CREATE TABLE import_templates (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Configuration
    entity_type             VARCHAR(50) NOT NULL, -- 'patient', 'appointment', 'encounter'
    file_format             VARCHAR(20) NOT NULL, -- 'csv', 'excel', 'json', 'xml'

    -- Field mapping
    field_mapping           JSONB NOT NULL,
    /* Example:
    {
      "columns": [
        {
          "source_column": "First Name",
          "target_field": "first_name",
          "required": true,
          "data_type": "string",
          "validation": "not_empty"
        },
        {
          "source_column": "DOB",
          "target_field": "date_of_birth",
          "required": true,
          "data_type": "date",
          "format": "DD/MM/YYYY"
        }
      ]
    }
    */

    -- Validation rules
    validation_rules        JSONB,

    -- Transformations
    transformations         JSONB,
    /* Example:
    {
      "date_of_birth": "parse_date(value, 'DD/MM/YYYY')",
      "phone": "format_phone(value, 'AR')",
      "insurance_name": "lookup_insurance_by_name(value)"
    }
    */

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE import_jobs (
    id                      BIGSERIAL PRIMARY KEY,
    import_template_id      BIGINT REFERENCES import_templates(id),

    -- File info
    file_name               VARCHAR(500),
    file_path               VARCHAR(1000),
    file_size               BIGINT,

    -- Status
    status                  VARCHAR(50) DEFAULT 'pending',
                            -- 'pending', 'processing', 'completed', 'failed', 'partially_completed'

    -- Progress
    total_rows              INTEGER,
    processed_rows          INTEGER DEFAULT 0,
    successful_rows         INTEGER DEFAULT 0,
    failed_rows             INTEGER DEFAULT 0,

    -- Results
    error_log               JSONB, -- Errors by row number
    created_records         BIGINT[], -- IDs of created records

    -- Timing
    started_at              TIMESTAMP,
    completed_at            TIMESTAMP,

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_import_jobs_status (status),
    INDEX idx_import_jobs_created_by (created_by)
);
```

### 2.2 Import Process

```javascript
/**
 * Import patients from CSV
 */
async function importPatients(file, templateId) {
  // 1. Create import job
  const importJob = await ImportJob.create({
    import_template_id: templateId,
    file_name: file.originalname,
    file_path: file.path,
    file_size: file.size,
    status: 'pending',
    created_by: currentUser.id
  });

  // 2. Parse file
  const template = await ImportTemplate.findById(templateId);
  const rows = await parseFile(file, template.file_format);

  await importJob.update({
    total_rows: rows.length,
    status: 'processing',
    started_at: new Date()
  });

  // 3. Process rows
  const results = {
    successful: [],
    failed: []
  };

  for (let i = 0; i < rows.length; i++) {
    const row = rows[i];

    try {
      // 4. Map fields
      const mappedData = mapFields(row, template.field_mapping);

      // 5. Validate
      const validation = validateData(mappedData, template.validation_rules);
      if (!validation.valid) {
        throw new Error(validation.errors.join(', '));
      }

      // 6. Transform
      const transformedData = await applyTransformations(
        mappedData,
        template.transformations
      );

      // 7. Check for duplicates
      const duplicate = await checkDuplicate(transformedData);
      if (duplicate) {
        throw new Error(`Duplicate DNI: ${transformedData.dni}`);
      }

      // 8. Create record
      const patient = await Patient.create(transformedData);

      results.successful.push({
        row: i + 1,
        id: patient.id
      });

    } catch (error) {
      results.failed.push({
        row: i + 1,
        data: row,
        error: error.message
      });
    }

    // Update progress
    await importJob.update({
      processed_rows: i + 1,
      successful_rows: results.successful.length,
      failed_rows: results.failed.length
    });
  }

  // 9. Finalize
  await importJob.update({
    status: results.failed.length === 0 ? 'completed' : 'partially_completed',
    completed_at: new Date(),
    error_log: results.failed,
    created_records: results.successful.map(r => r.id)
  });

  return importJob;
}
```

### 2.3 Import Templates

**Patient Import Template (CSV)**
```csv
First Name,Last Name,DNI,Date of Birth,Gender,Phone,Email,Insurance,Member Number
Juan,Perez,12345678,15/05/1980,M,11-1234-5678,juan@email.com,OSDE,123456
Maria,Garcia,87654321,20/08/1990,F,11-8765-4321,maria@email.com,Swiss Medical,789012
```

**Appointment Import Template (Excel)**
```
| Patient DNI | Doctor Name  | Date       | Time  | Type          | Notes          |
|-------------|--------------|------------|-------|---------------|----------------|
| 12345678    | Dr. Martinez | 20/11/2025 | 10:00 | Consulta      | First visit    |
| 87654321    | Dr. Garcia   | 21/11/2025 | 14:00 | Control       | Follow-up      |
```

## 2.4 Data Export System

```javascript
/**
 * Export data to various formats
 */
const exportService = {
  /**
   * Export patients to Excel
   */
  async exportPatients(filters, format = 'excel') {
    // 1. Query patients
    const patients = await Patient.findAll({
      where: filters,
      include: [
        { model: Insurance, as: 'insurances' },
        { model: Encounter, as: 'encounters' }
      ]
    });

    // 2. Transform data
    const exportData = patients.map(p => ({
      'Patient Number': p.patient_number,
      'First Name': p.first_name,
      'Last Name': p.last_name,
      'DNI': p.dni,
      'Date of Birth': formatDate(p.date_of_birth),
      'Age': calculateAge(p.date_of_birth),
      'Gender': p.gender,
      'Phone': p.phone_primary,
      'Email': p.email,
      'Primary Insurance': p.insurances.find(i => i.is_primary)?.name || 'N/A',
      'Total Visits': p.encounters.length,
      'Last Visit': p.encounters[0]?.encounter_date || 'Never',
      'Active': p.is_active ? 'Yes' : 'No'
    }));

    // 3. Generate file
    if (format === 'excel') {
      return generateExcel(exportData, 'Patients');
    } else if (format === 'csv') {
      return generateCSV(exportData);
    } else if (format === 'pdf') {
      return generatePDF(exportData, 'Patient Report');
    } else if (format === 'json') {
      return JSON.stringify(exportData, null, 2);
    }
  },

  /**
   * Export financial data for accounting
   */
  async exportFinancialData(startDate, endDate, format = 'excel') {
    const invoices = await Invoice.findAll({
      where: {
        invoice_date: {
          [Op.between]: [startDate, endDate]
        }
      },
      include: ['patient', 'items', 'payments']
    });

    const exportData = invoices.map(inv => ({
      'Invoice Number': inv.invoice_number,
      'Invoice Type': inv.invoice_type,
      'Date': formatDate(inv.invoice_date),
      'Patient': `${inv.patient.last_name}, ${inv.patient.first_name}`,
      'DNI': inv.patient.dni,
      'CAE': inv.cae,
      'Subtotal': inv.subtotal,
      'Tax (IVA)': inv.tax_amount,
      'Total': inv.total_amount,
      'Amount Paid': inv.amount_paid,
      'Balance Due': inv.balance_due,
      'Status': inv.status
    }));

    if (format === 'excel') {
      const workbook = new ExcelJS.Workbook();
      const sheet = workbook.addWorksheet('Invoices');

      // Add headers
      sheet.columns = Object.keys(exportData[0]).map(key => ({
        header: key,
        key: key,
        width: 15
      }));

      // Add data
      exportData.forEach(row => sheet.addRow(row));

      // Add totals row
      const totalsRow = {
        'Invoice Number': 'TOTALS:',
        'Subtotal': { formula: `SUM(G2:G${exportData.length + 1})` },
        'Tax (IVA)': { formula: `SUM(H2:H${exportData.length + 1})` },
        'Total': { formula: `SUM(I2:I${exportData.length + 1})` },
        'Amount Paid': { formula: `SUM(J2:J${exportData.length + 1})` },
        'Balance Due': { formula: `SUM(K2:K${exportData.length + 1})` }
      };
      sheet.addRow(totalsRow);

      // Format
      sheet.getRow(1).font = { bold: true };
      sheet.getRow(exportData.length + 2).font = { bold: true };

      return workbook.xlsx.writeBuffer();
    }

    return exportData;
  },

  /**
   * Export AFIP data for tax reporting
   */
  async exportAFIPReport(month, year) {
    const invoices = await Invoice.findAll({
      where: {
        invoice_date: {
          [Op.between]: [
            `${year}-${month.toString().padStart(2, '0')}-01`,
            `${year}-${month.toString().padStart(2, '0')}-31`
          ]
        },
        cae: { [Op.ne]: null }
      }
    });

    // AFIP format
    const afipData = invoices.map(inv => ({
      'Fecha': formatDate(inv.invoice_date, 'YYYYMMDD'),
      'Tipo': inv.invoice_type,
      'Punto de Venta': inv.punto_venta,
      'Número': inv.invoice_number.split('-')[1],
      'CAE': inv.cae,
      'Vencimiento CAE': formatDate(inv.cae_expiration, 'YYYYMMDD'),
      'Neto Gravado': inv.subtotal,
      'IVA': inv.tax_amount,
      'Total': inv.total_amount
    }));

    return generateCSV(afipData);
  }
};
```

### 2.5 Bulk Operations

```javascript
/**
 * Bulk update patients
 */
async function bulkUpdatePatients(patientIds, updates) {
  const results = {
    successful: [],
    failed: []
  };

  for (const patientId of patientIds) {
    try {
      await Patient.update(updates, {
        where: { id: patientId }
      });

      results.successful.push(patientId);
    } catch (error) {
      results.failed.push({
        id: patientId,
        error: error.message
      });
    }
  }

  return results;
}

/**
 * Bulk send messages
 */
async function bulkSendMessages(patientIds, message, channel = 'sms') {
  const patients = await Patient.findAll({
    where: { id: { [Op.in]: patientIds } }
  });

  const results = {
    sent: [],
    failed: []
  };

  for (const patient of patients) {
    try {
      if (channel === 'sms' && patient.phone_primary) {
        await smsService.send(patient.phone_primary, message);
        results.sent.push(patient.id);
      } else if (channel === 'email' && patient.email) {
        await emailService.send(patient.email, 'Message from Clinic', message);
        results.sent.push(patient.id);
      }
    } catch (error) {
      results.failed.push({
        id: patient.id,
        error: error.message
      });
    }
  }

  return results;
}
```

---

# PART 3: ADVANCED API INTEGRATIONS

## 3.1 Webhook System

```sql
CREATE TABLE webhooks (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Target
    url                     VARCHAR(1000) NOT NULL,
    method                  VARCHAR(10) DEFAULT 'POST', -- GET, POST, PUT

    -- Trigger
    event_types             TEXT[] NOT NULL,
    /* Examples:
       ['patient.created', 'patient.updated', 'appointment.scheduled',
        'encounter.completed', 'invoice.paid']
    */

    -- Authentication
    auth_type               VARCHAR(50), -- 'none', 'basic', 'bearer', 'api_key'
    auth_config             JSONB,
    /* Example:
    {
      "api_key_header": "X-API-Key",
      "api_key_value": "encrypted_key"
    }
    */

    -- Headers
    custom_headers          JSONB,

    -- Filtering
    filter_conditions       JSONB,
    /* Example:
    {
      "patient.created": {
        "insurance_id": [1, 2, 3]  // Only for specific insurances
      }
    }
    */

    -- Retry policy
    max_retries             INTEGER DEFAULT 3,
    retry_delay             INTEGER DEFAULT 300, -- seconds

    -- Status
    is_active               BOOLEAN DEFAULT true,
    last_triggered_at       TIMESTAMP,
    success_count           INTEGER DEFAULT 0,
    failure_count           INTEGER DEFAULT 0,

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE webhook_deliveries (
    id                      BIGSERIAL PRIMARY KEY,
    webhook_id              BIGINT NOT NULL REFERENCES webhooks(id),

    -- Event
    event_type              VARCHAR(100) NOT NULL,
    event_data              JSONB NOT NULL,

    -- Request
    request_url             VARCHAR(1000),
    request_method          VARCHAR(10),
    request_headers         JSONB,
    request_body            JSONB,

    -- Response
    response_status         INTEGER,
    response_headers        JSONB,
    response_body           TEXT,

    -- Status
    status                  VARCHAR(50), -- 'pending', 'success', 'failed', 'retrying'
    attempt_count           INTEGER DEFAULT 1,
    error_message           TEXT,

    -- Timing
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    delivered_at            TIMESTAMP,

    INDEX idx_webhook_deliveries_webhook (webhook_id),
    INDEX idx_webhook_deliveries_status (status),
    INDEX idx_webhook_deliveries_created (created_at)
);
```

### 3.2 Third-Party API Integration Framework

```javascript
/**
 * Generic API integration client
 */
class APIIntegration {
  constructor(config) {
    this.baseURL = config.baseURL;
    this.apiKey = config.apiKey;
    this.authType = config.authType; // 'bearer', 'basic', 'api_key'
    this.headers = config.headers || {};
  }

  async request(method, endpoint, data = null) {
    const headers = {
      ...this.headers,
      'Content-Type': 'application/json'
    };

    // Add authentication
    if (this.authType === 'bearer') {
      headers['Authorization'] = `Bearer ${this.apiKey}`;
    } else if (this.authType === 'api_key') {
      headers['X-API-Key'] = this.apiKey;
    }

    const options = {
      method,
      headers,
      body: data ? JSON.stringify(data) : null
    };

    try {
      const response = await fetch(`${this.baseURL}${endpoint}`, options);

      if (!response.ok) {
        throw new Error(`API request failed: ${response.statusText}`);
      }

      return await response.json();
    } catch (error) {
      // Log error
      await logAPIError(this.baseURL, endpoint, error);
      throw error;
    }
  }

  async get(endpoint) {
    return this.request('GET', endpoint);
  }

  async post(endpoint, data) {
    return this.request('POST', endpoint, data);
  }

  async put(endpoint, data) {
    return this.request('PUT', endpoint, data);
  }

  async delete(endpoint) {
    return this.request('DELETE', endpoint);
  }
}

/**
 * Example: Laboratory API Integration
 */
class LabAPIIntegration extends APIIntegration {
  async submitOrder(labOrder) {
    const orderData = {
      patient: {
        firstName: labOrder.patient.first_name,
        lastName: labOrder.patient.last_name,
        dni: labOrder.patient.dni,
        dateOfBirth: labOrder.patient.date_of_birth
      },
      tests: labOrder.items.map(item => ({
        code: item.test_code,
        name: item.test_name
      })),
      orderNumber: labOrder.order_number,
      priority: labOrder.is_urgent ? 'STAT' : 'ROUTINE'
    };

    const response = await this.post('/orders', orderData);

    // Update order with lab's reference number
    await labOrder.update({
      external_reference: response.labOrderId
    });

    return response;
  }

  async getResults(externalReference) {
    const response = await this.get(`/orders/${externalReference}/results`);

    return response.results.map(r => ({
      test_code: r.testCode,
      test_name: r.testName,
      result_value: r.value,
      result_unit: r.unit,
      reference_range: r.referenceRange,
      is_abnormal: r.flag === 'H' || r.flag === 'L',
      abnormal_flag: r.flag
    }));
  }
}
```

---

# PART 4: AI & ML INTEGRATION

## 4.1 AI-Powered Features

### 4.1.1 Clinical Decision Support

```javascript
/**
 * AI-powered diagnosis suggestions
 */
const diagnosisSuggestion = {
  async suggest(encounterData) {
    const prompt = `
      Patient Presentation:
      Age: ${encounterData.patient_age}
      Gender: ${encounterData.patient_gender}
      Chief Complaint: ${encounterData.chief_complaint}
      Symptoms: ${encounterData.symptoms.join(', ')}
      Vital Signs:
        - BP: ${encounterData.vitals.bp}
        - HR: ${encounterData.vitals.hr}
        - Temp: ${encounterData.vitals.temp}

      Provide:
      1. Top 5 differential diagnoses with ICD-10 codes
      2. Recommended tests/labs
      3. Red flags to watch for
    `;

    const response = await openai.chat.completions.create({
      model: 'gpt-4',
      messages: [
        {
          role: 'system',
          content: 'You are a medical AI assistant. Provide clinical decision support based on patient presentation. Always include relevant ICD-10 codes and emphasize when immediate medical attention is needed.'
        },
        { role: 'user', content: prompt }
      ],
      temperature: 0.3 // Lower temperature for medical accuracy
    });

    return parseDiagnosisSuggestions(response.choices[0].message.content);
  }
};
```

### 4.1.2 Automated Medical Coding

```javascript
/**
 * Auto-suggest ICD-10 codes from clinical notes
 */
async function suggestICD10Codes(clinicalNotes) {
  const response = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: [
      {
        role: 'system',
        content: 'Extract medical conditions from clinical notes and suggest appropriate ICD-10 codes. Return as JSON array.'
      },
      {
        role: 'user',
        content: `Clinical Notes: ${clinicalNotes}\n\nSuggest ICD-10 codes in this format: [{"condition": "...", "icd10": "...", "confidence": 0.0-1.0}]`
      }
    ]
  });

  return JSON.parse(response.choices[0].message.content);
}
```

### 4.1.3 Appointment No-Show Prediction

```javascript
/**
 * Predict likelihood of appointment no-show
 */
const noShowPredictor = {
  async predict(appointmentData) {
    const features = {
      // Patient history
      previous_no_shows: appointmentData.patient.no_show_count,
      total_appointments: appointmentData.patient.appointment_count,
      no_show_rate: appointmentData.patient.no_show_count /
                    appointmentData.patient.appointment_count,

      // Appointment characteristics
      lead_time_days: calculateLeadTime(appointmentData.scheduled_at),
      appointment_hour: new Date(appointmentData.scheduled_at).getHours(),
      day_of_week: new Date(appointmentData.scheduled_at).getDay(),
      is_first_appointment: appointmentData.patient.appointment_count === 0,

      // Demographics
      age: calculateAge(appointmentData.patient.date_of_birth),
      has_insurance: appointmentData.patient.insurances.length > 0,

      // Communication
      reminder_sent: appointmentData.reminder_sent_at !== null,
      confirmed: appointmentData.status === 'confirmed'
    };

    // Use ML model (trained on historical data)
    const prediction = await mlModel.predict(features);

    return {
      no_show_probability: prediction.probability,
      risk_level: prediction.probability > 0.7 ? 'high' :
                  prediction.probability > 0.4 ? 'medium' : 'low',
      recommended_actions: getRecommendedActions(prediction.probability)
    };
  }
};

function getRecommendedActions(probability) {
  if (probability > 0.7) {
    return [
      'Send additional reminder 2 hours before appointment',
      'Call patient to confirm',
      'Consider overbooking this slot'
    ];
  } else if (probability > 0.4) {
    return [
      'Send reminder as scheduled',
      'Monitor confirmation status'
    ];
  }

  return ['Standard reminder protocol'];
}
```

### 4.1.4 Smart Appointment Scheduling

```javascript
/**
 * AI-powered appointment slot recommendation
 */
const smartScheduler = {
  async recommendSlots(patientId, providerId, preferences = {}) {
    // Get patient history
    const patient = await Patient.findById(patientId, {
      include: ['appointments', 'insurances']
    });

    // Get provider availability
    const availability = await getProviderAvailability(providerId, {
      startDate: preferences.startDate || new Date(),
      days: 14
    });

    // Analyze patterns
    const patientPreferences = analyzePatientPatterns(patient.appointments);

    // Score each available slot
    const scoredSlots = availability.map(slot => ({
      ...slot,
      score: calculateSlotScore(slot, patientPreferences, preferences)
    }));

    // Sort by score
    scoredSlots.sort((a, b) => b.score - a.score);

    return scoredSlots.slice(0, 5); // Top 5 recommendations
  }
};

function analyzePatientPatterns(appointments) {
  const preferredDays = getMostCommonDays(appointments);
  const preferredTimes = getMostCommonTimes(appointments);
  const avgNoShowTime = getAverageNoShowTimeSlot(appointments);

  return {
    preferredDays,
    preferredTimes,
    avgNoShowTime
  };
}

function calculateSlotScore(slot, patientPreferences, requestPreferences) {
  let score = 0;

  // Preferred day of week
  if (patientPreferences.preferredDays.includes(slot.dayOfWeek)) {
    score += 30;
  }

  // Preferred time
  const slotHour = slot.time.getHours();
  if (patientPreferences.preferredTimes.includes(slotHour)) {
    score += 25;
  }

  // Avoid times when patient typically no-shows
  if (patientPreferences.avgNoShowTime &&
      Math.abs(slotHour - patientPreferences.avgNoShowTime) < 2) {
    score -= 20;
  }

  // Request preferences
  if (requestPreferences.preferredDays?.includes(slot.dayOfWeek)) {
    score += 20;
  }

  // Urgency - sooner is better for urgent requests
  if (requestPreferences.urgent) {
    const daysAway = (slot.date - new Date()) / (1000 * 60 * 60 * 24);
    score += Math.max(0, 20 - daysAway * 2);
  }

  return score;
}
```

### 4.1.5 Prescription Drug Interaction AI

```javascript
/**
 * Advanced AI-powered drug interaction checking
 */
async function checkDrugInteractionsAI(medications) {
  const prompt = `
    Analyze potential drug interactions for this medication list:
    ${medications.map((m, i) => `${i + 1}. ${m.name} ${m.dosage}`).join('\n')}

    Patient info:
    - Age: ${patient.age}
    - Chronic conditions: ${patient.conditions.join(', ')}
    - Allergies: ${patient.allergies.join(', ')}

    Provide:
    1. Significant interactions (with severity rating)
    2. Recommendations for safer alternatives
    3. Monitoring requirements
    4. Patient education points
  `;

  const response = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: [
      { role: 'system', content: 'You are a clinical pharmacist AI assistant.' },
      { role: 'user', content: prompt }
    ]
  });

  return parseInteractionReport(response.choices[0].message.content);
}
```

### 4.1.6 Medical Image Analysis (Future)

```javascript
/**
 * AI-powered medical image analysis
 * (Integration with services like Google Cloud Healthcare API)
 */
const imageAnalysis = {
  async analyzeCXR(imageFile) {
    // Send X-ray to AI service
    const response = await googleHealthcareAPI.analyzeCXR(imageFile);

    return {
      findings: response.findings,
      /* Example:
      [
        { finding: 'Cardiomegaly', confidence: 0.85, location: {...} },
        { finding: 'Pleural effusion', confidence: 0.72, location: {...} }
      ]
      */
      normalityScore: response.normalityScore,
      recommendedActions: response.recommendations
    };
  }
};
```

---

# PART 5: PATIENT PORTAL

## 5.1 Patient Portal Database Schema

```sql
CREATE TABLE patient_portal_users (
    id                      BIGSERIAL PRIMARY KEY,
    patient_id              BIGINT UNIQUE NOT NULL REFERENCES patients(id),

    -- Credentials
    email                   VARCHAR(255) UNIQUE NOT NULL,
    password_hash           VARCHAR(255) NOT NULL,

    -- Verification
    email_verified          BOOLEAN DEFAULT false,
    email_verification_token VARCHAR(100),
    email_verified_at       TIMESTAMP,

    -- Password reset
    password_reset_token    VARCHAR(100),
    password_reset_expires  TIMESTAMP,

    -- Security
    two_factor_enabled      BOOLEAN DEFAULT false,
    two_factor_secret       VARCHAR(255),

    -- Status
    is_active               BOOLEAN DEFAULT true,
    last_login_at           TIMESTAMP,
    login_count             INTEGER DEFAULT 0,

    -- Preferences
    preferences             JSONB,
    /* Example:
    {
      "language": "es",
      "notifications": {
        "email": true,
        "sms": true,
        "push": false
      },
      "communication_preference": "email"
    }
    */

    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_portal_users_email (email),
    INDEX idx_portal_users_patient (patient_id)
);

CREATE TABLE patient_portal_sessions (
    id                      BIGSERIAL PRIMARY KEY,
    portal_user_id          BIGINT NOT NULL REFERENCES patient_portal_users(id),
    session_token           VARCHAR(255) UNIQUE NOT NULL,
    expires_at              TIMESTAMP NOT NULL,
    ip_address              VARCHAR(50),
    user_agent              TEXT,
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_portal_sessions_token (session_token),
    INDEX idx_portal_sessions_user (portal_user_id)
);
```

## 5.2 Patient Portal Features

### 5.2.1 Dashboard

```
┌──────────────────────────────────────────────────────────────────┐
│ [Logo] Patient Portal               Carlos Rodriguez    [Logout] │
├──────────────────────────────────────────────────────────────────┤
│ [Dashboard] [Appointments] [Medical Records] [Messages] [Profile]│
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Welcome back, Carlos!                                            │
│                                                                  │
│ ┌──────────────────────────────────────────────────────────────┐│
│ │ 🗓️ Upcoming Appointments                                     ││
│ ├──────────────────────────────────────────────────────────────┤│
│ │ Nov 20, 2025 - 10:00 AM                                      ││
│ │ Dr. Martinez - Follow-up consultation                        ││
│ │ Location: Main Clinic                                        ││
│ │ [View Details] [Cancel] [Reschedule]                         ││
│ ├──────────────────────────────────────────────────────────────┤│
│ │ Dec 5, 2025 - 2:00 PM                                        ││
│ │ Dr. Garcia - Lab results review                              ││
│ │ [View Details] [Cancel] [Reschedule]                         ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ ┌─────────────────────────┐ ┌───────────────────────────────┐  │
│ │ 📋 Recent Lab Results   │ │ 💊 Active Prescriptions       │  │
│ ├─────────────────────────┤ ├───────────────────────────────┤  │
│ │ Nov 15, 2025            │ │ Ibuprofeno 400mg              │  │
│ │ Hemograma Completo      │ │ Take 1 every 8 hours          │  │
│ │ [View Results]          │ │ Refills: 2                    │  │
│ │                         │ │ [Request Refill]              │  │
│ │ Oct 20, 2025            │ │                               │  │
│ │ Glucose Test            │ │ Atorvastatina 20mg            │  │
│ │ [View Results]          │ │ Take 1 daily at night         │  │
│ │                         │ │ Refills: 5                    │  │
│ └─────────────────────────┘ └───────────────────────────────┘  │
│                                                                  │
│ ┌──────────────────────────────────────────────────────────────┐│
│ │ 💰 Billing & Payments                                        ││
│ ├──────────────────────────────────────────────────────────────┤│
│ │ Outstanding Balance: $1,500.00                               ││
│ │                                                              ││
│ │ Recent Invoices:                                             ││
│ │ • Invoice #0001-123 - $500.00 - Paid ✓                      ││
│ │ • Invoice #0001-124 - $1,000.00 - Pending [Pay Now]         ││
│ │ • Invoice #0001-125 - $500.00 - Partial ($250 paid)         ││
│ │                                           [Pay Remaining]    ││
│ └──────────────────────────────────────────────────────────────┘│
│                                                                  │
│ [+ Schedule New Appointment]                                     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2.2 Appointment Scheduling (Patient Self-Service)

```
┌──────────────────────────────────────────────────────────────────┐
│ Schedule New Appointment                                    [×]  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Step 1 of 4: Select Doctor                                      │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ ○ Dr. Juan Martinez - General Medicine                     │ │
│ │   Available: Mon, Wed, Fri                                 │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ ○ Dr. Maria Garcia - Cardiology                            │ │
│ │   Available: Tue, Thu                                      │ │
│ ├────────────────────────────────────────────────────────────┤ │
│ │ ● Dr. Sofia Lopez - Dermatology                            │ │
│ │   Available: Mon-Fri                                       │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│                                         [Cancel]  [Next Step →] │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ Schedule New Appointment                                    [×]  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Step 2 of 4: Select Date & Time                                 │
│ Doctor: Dr. Sofia Lopez                                          │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ November 2025                            [< Prev] [Next >] │ │
│ │                                                            │ │
│ │ Mon    Tue    Wed    Thu    Fri    Sat    Sun             │ │
│ │        1      2      3      4      5      6               │ │
│ │ 8      9     10     11     12     13     14               │ │
│ │ 15    16     17     18     19     20     21               │ │
│ │ 22    23     24    [25]    26     27     28               │ │
│ │ 29    30                                                  │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Available Times for Nov 25:                                      │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Morning:                                                   │ │
│ │ [09:00] [09:30] [10:00] [10:30] [11:00] [11:30]           │ │
│ │                                                            │ │
│ │ Afternoon:                                                 │ │
│ │ [14:00] [14:30] [15:00] [15:30] [16:00] [16:30]           │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│                                   [← Back]  [Cancel]  [Next →]  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ Schedule New Appointment                                    [×]  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Step 3 of 4: Appointment Details                                │
│                                                                  │
│ Appointment Type:                                                │
│ ● First Consultation  ○ Follow-up  ○ Procedure                  │
│                                                                  │
│ Reason for Visit:                                                │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Skin rash on arms that started last week. Itchy and red.  │ │
│ │                                                            │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Insurance:                                                       │
│ [OSDE - Plan 210 ▼]                                             │
│                                                                  │
│ ☑ Send me appointment reminders                                 │
│   ☑ Email (24 hours before)                                     │
│   ☑ SMS (2 hours before)                                        │
│                                                                  │
│                                   [← Back]  [Cancel]  [Next →]  │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ Confirm Appointment                                         [×]  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Step 4 of 4: Confirm Details                                    │
│                                                                  │
│ Please review your appointment details:                          │
│                                                                  │
│ Doctor:      Dr. Sofia Lopez (Dermatology)                       │
│ Date:        Tuesday, November 25, 2025                          │
│ Time:        10:00 AM - 10:30 AM                                 │
│ Type:        First Consultation                                  │
│ Location:    Main Clinic                                         │
│              Av. Corrientes 1234, Buenos Aires                   │
│                                                                  │
│ Reason:      Skin rash on arms                                   │
│ Insurance:   OSDE - Plan 210                                     │
│                                                                  │
│ Reminders:   Email & SMS                                         │
│                                                                  │
│ ⓘ Please arrive 15 minutes early to complete paperwork          │
│                                                                  │
│                           [← Back]  [Cancel]  [Confirm Appointment]│
└──────────────────────────────────────────────────────────────────┘
```

### 5.2.3 View Medical Records

```
┌──────────────────────────────────────────────────────────────────┐
│ Medical Records                                                  │
├──────────────────────────────────────────────────────────────────┤
│ [Overview] [Encounters] [Lab Results] [Imaging] [Prescriptions] │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Recent Encounters:                                               │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Nov 15, 2025 - Dr. Martinez                                │ │
│ │ Diagnosis: Hypertension (I10)                              │ │
│ │ Treatment: Medication adjustment, lifestyle changes        │ │
│ │ [View Full Notes] [Download Summary]                       │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Oct 20, 2025 - Dr. Garcia                                  │ │
│ │ Diagnosis: Type 2 Diabetes (E11.9)                         │ │
│ │ Treatment: Metformin 500mg, diet counseling                │ │
│ │ [View Full Notes] [Download Summary]                       │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Lab Results:                                                     │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ 📊 Nov 15, 2025 - Hemograma Completo                       │ │
│ │                                                            │ │
│ │ Test               Result      Reference    Status        │ │
│ │ Hemoglobina        14.5 g/dL   12-16       ✓ Normal      │ │
│ │ Glóbulos Blancos   12.5 10³/μL 4-11        ⚠ High        │ │
│ │ Plaquetas          250 10³/μL  150-400     ✓ Normal      │ │
│ │                                                            │ │
│ │ Doctor's Note: Slightly elevated WBC. Monitor for         │ │
│ │ infection symptoms. Follow-up if persists.                 │ │
│ │                                                            │ │
│ │ [View Full Report] [Download PDF] [Share with Doctor]     │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Medical Images:                                                  │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ 🩻 Oct 10, 2025 - Chest X-Ray                              │ │
│ │                                                            │ │
│ │ [Thumbnail Image]                                          │ │
│ │                                                            │ │
│ │ Finding: Normal cardiac silhouette. No acute findings.     │ │
│ │                                                            │ │
│ │ [View Image] [Download] [Share]                            │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2.4 Prescription Management

```
┌──────────────────────────────────────────────────────────────────┐
│ My Prescriptions                                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Active Medications:                                              │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ 💊 Ibuprofeno 400mg                                         │ │
│ │    Take 1 tablet every 8 hours with food                   │ │
│ │    Duration: 7 days                                        │ │
│ │    Prescribed: Nov 10, 2025 by Dr. Martinez                │ │
│ │    Refills Remaining: 2                                    │ │
│ │    Next Refill Available: Dec 10, 2025                     │ │
│ │                                                            │ │
│ │    [View Prescription] [Request Refill] [Download PDF]     │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ 💊 Atorvastatina 20mg (Chronic)                            │ │
│ │    Take 1 tablet daily at bedtime                          │ │
│ │    Prescribed: Jan 15, 2025 by Dr. Garcia                  │ │
│ │    Refills Remaining: 5                                    │ │
│ │                                                            │ │
│ │    ⓘ Important: Do not stop without consulting doctor     │ │
│ │                                                            │ │
│ │    [View Prescription] [Request Refill]                    │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Prescription History:                                            │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Oct 20, 2025 - Amoxicilina 500mg (Completed)              │ │
│ │ Sep 15, 2025 - Ibuprofeno 400mg (Completed)               │ │
│ │ [View All History]                                         │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Request Refill:                                                  │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Select medication to refill:                               │ │
│ │ [Ibuprofeno 400mg ▼]                                       │ │
│ │                                                            │ │
│ │ Reason for refill:                                         │ │
│ │ [Still experiencing symptoms as prescribed]                │ │
│ │                                                            │ │
│ │ Preferred pharmacy:                                        │ │
│ │ [Farmacity - Av. Corrientes 2000 ▼]                       │ │
│ │                                                            │ │
│ │           [Cancel]  [Submit Refill Request]                │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2.5 Billing & Payments

```
┌──────────────────────────────────────────────────────────────────┐
│ Billing & Payments                                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Account Summary:                                                 │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Total Outstanding: $1,500.00                               │ │
│ │ [Pay Now]                                                  │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Recent Invoices:                                                 │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Invoice #0001-00000124                           PENDING   │ │
│ │ Date: Nov 15, 2025                                         │ │
│ │ Description: Medical consultation - Dr. Martinez           │ │
│ │ Amount: $1,000.00                                          │ │
│ │ Insurance Coverage: $800.00 (OSDE)                         │ │
│ │ Your Responsibility: $200.00                               │ │
│ │                                                            │ │
│ │ [View Invoice] [Pay $200.00] [Download PDF]                │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Invoice #0001-00000123                             PAID ✓  │ │
│ │ Date: Oct 20, 2025                                         │ │
│ │ Description: Lab tests                                     │ │
│ │ Amount: $500.00                                            │ │
│ │ Paid: Oct 20, 2025 via Credit Card                        │ │
│ │                                                            │ │
│ │ [View Invoice] [View Receipt] [Download]                   │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Payment Options:                                                 │
│                                                                  │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Pay Invoice #0001-00000124                                 │ │
│ │                                                            │ │
│ │ Amount Due: $200.00                                        │ │
│ │                                                            │ │
│ │ Payment Method:                                            │ │
│ │ ● Credit/Debit Card                                        │ │
│ │ ○ Bank Transfer                                            │ │
│ │ ○ Mercado Pago                                             │ │
│ │                                                            │ │
│ │ Card Information:                                          │ │
│ │ Card Number: [____ ____ ____ ____]                        │ │
│ │ Name: [Carlos Rodriguez]                                   │ │
│ │ Expiry: [MM] / [YY]  CVV: [___]                           │ │
│ │                                                            │ │
│ │ ☑ Save this card for future payments                      │ │
│ │                                                            │ │
│ │ 🔒 Secure payment processing                              │ │
│ │                                                            │ │
│ │                    [Cancel]  [Pay $200.00]                 │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ Payment History:                                                 │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ Oct 20, 2025 - $500.00 - Credit Card ****1234             │ │
│ │ Sep 15, 2025 - $300.00 - Cash                             │ │
│ │ Aug 10, 2025 - $450.00 - Mercado Pago                     │ │
│ │ [View All Payments]                                        │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2.6 Messages (Secure Messaging)

```
┌──────────────────────────────────────────────────────────────────┐
│ Messages                                              [+ New]     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ ┌────────────────┐ ┌──────────────────────────────────────────┐│
│ │ Conversations  │ │ Dr. Juan Martinez                        ││
│ ├────────────────┤ ├──────────────────────────────────────────┤│
│ │ 🩺 Dr. Martinez│ │                                          ││
│ │   Nov 16, 10am │ │ Nov 15, 2025 2:00 PM                     ││
│ │   Lab results  │ │ Dr. Martinez:                            ││
│ │                │ │ Your lab results are in. Glucose levels  ││
│ │ 👨‍⚕️ Reception  │ │ are slightly elevated. Let's discuss at ││
│ │   Nov 14, 3pm  │ │ your next appointment.                   ││
│ │   Appointment  │ │                                          ││
│ │                │ │ Nov 16, 2025 10:00 AM                    ││
│ │ 💊 Pharmacy    │ │ You:                                     ││
│ │   Nov 12, 11am │ │ Thank you doctor. Should I make any      ││
│ │   Prescription │ │ dietary changes in the meantime?         ││
│ │                │ │                                          ││
│ └────────────────┘ │ Nov 16, 2025 10:30 AM                    ││
│                    │ Dr. Martinez:                            ││
│                    │ Yes, reduce sugar intake and increase    ││
│                    │ physical activity. We'll create a plan   ││
│                    │ at your appointment.                     ││
│                    │                                          ││
│                    ├──────────────────────────────────────────┤│
│                    │ Type your message...          [Send]     ││
│                    │ [📎 Attach File]                         ││
│                    └──────────────────────────────────────────┘│
│                                                                  │
│ ⓘ Messages are secure and HIPAA-compliant. Replies within 24hrs │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2.7 Profile & Settings

```
┌──────────────────────────────────────────────────────────────────┐
│ My Profile                                                       │
├──────────────────────────────────────────────────────────────────┤
│ [Personal Info] [Contact] [Insurance] [Notifications] [Security]│
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Personal Information:                                            │
│                                                                  │
│ Name:              Carlos Rodriguez                              │
│ DNI:               30123456                                      │
│ Date of Birth:     May 15, 1980 (45 years old)                  │
│ Gender:            Male                                          │
│ Blood Type:        O+                                            │
│                                                                  │
│ Contact Information:                                             │
│                                                                  │
│ Email:             [carlos@email.com]                   [Edit]   │
│ Phone (Primary):   [+54 9 11 1234-5678]                [Edit]   │
│ Phone (Secondary): [+54 9 11 8765-4321]                [Edit]   │
│                                                                  │
│ Address:           [Av. Corrientes 1234, Apt 5B]        [Edit]   │
│ City:              [Buenos Aires]                       [Edit]   │
│ Postal Code:       [1043]                               [Edit]   │
│                                                                  │
│ Emergency Contact:                                               │
│                                                                  │
│ Name:              [Ana Rodriguez]                      [Edit]   │
│ Relationship:      [Spouse]                             [Edit]   │
│ Phone:             [+54 9 11 9999-9999]                [Edit]   │
│                                                                  │
│ Insurance Information:                                           │
│                                                                  │
│ Primary: OSDE - Plan 210                                         │
│ Member Number: 123456789                                         │
│ [Upload Insurance Card Photo]                                    │
│                                                                  │
│ Notification Preferences:                                        │
│                                                                  │
│ Appointment Reminders:                                           │
│ ☑ Email (24 hours before)                                       │
│ ☑ SMS (2 hours before)                                          │
│ ☐ WhatsApp                                                      │
│                                                                  │
│ Lab Results:                                                     │
│ ☑ Email when results available                                  │
│ ☑ Portal notification                                           │
│                                                                  │
│ Security Settings:                                               │
│                                                                  │
│ Password:          ••••••••••                           [Change] │
│                                                                  │
│ Two-Factor Authentication:                                       │
│ ☐ Enable 2FA for additional security              [Enable]      │
│                                                                  │
│ Login Activity:                                                  │
│ Last login: Nov 16, 2025 10:30 AM from 192.168.1.100            │
│ [View All Login History]                                        │
│                                                                  │
│                           [Cancel]  [Save Changes]               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 5.3 Patient Portal API Endpoints

```
Authentication:
POST   /api/portal/register              Register new patient portal account
POST   /api/portal/login                 Login to portal
POST   /api/portal/logout                Logout
POST   /api/portal/password/reset        Request password reset
POST   /api/portal/password/change       Change password
POST   /api/portal/verify-email          Verify email address

Dashboard:
GET    /api/portal/dashboard              Get dashboard data (appointments, results, etc.)

Appointments:
GET    /api/portal/appointments           Get patient's appointments
POST   /api/portal/appointments           Schedule new appointment
GET    /api/portal/appointments/available Get available time slots
PUT    /api/portal/appointments/:id/cancel Cancel appointment
PUT    /api/portal/appointments/:id/reschedule Reschedule appointment

Medical Records:
GET    /api/portal/encounters             Get patient's encounters
GET    /api/portal/encounters/:id         Get encounter details
GET    /api/portal/lab-results            Get lab results
GET    /api/portal/lab-results/:id        Get specific lab result
GET    /api/portal/documents              Get uploaded documents
GET    /api/portal/documents/:id/download Download document

Prescriptions:
GET    /api/portal/prescriptions          Get prescriptions
GET    /api/portal/prescriptions/:id      Get prescription details
POST   /api/portal/prescriptions/:id/refill Request refill
GET    /api/portal/prescriptions/:id/pdf  Download prescription PDF

Billing:
GET    /api/portal/invoices               Get invoices
GET    /api/portal/invoices/:id           Get invoice details
POST   /api/portal/invoices/:id/pay       Make payment
GET    /api/portal/payments               Get payment history

Messages:
GET    /api/portal/messages               Get messages
POST   /api/portal/messages               Send message
GET    /api/portal/messages/:id           Get message thread
PUT    /api/portal/messages/:id/read      Mark as read

Profile:
GET    /api/portal/profile                Get profile
PUT    /api/portal/profile                Update profile
POST   /api/portal/profile/photo          Upload profile photo
```

---

*Complete System Features Specification Version: 1.0*
*Last Updated: 2025-11-15*
*Comprehensive coverage of all advanced features*
