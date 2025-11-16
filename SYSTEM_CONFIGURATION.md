# Clinical Management System - System Configuration & Settings

## Overview

This document provides comprehensive specifications for system configuration, settings management, multi-location support, and administrative controls.

---

## 1. SYSTEM SETTINGS ARCHITECTURE

### 1.1 Settings Database Schema

```sql
CREATE TABLE system_settings (
    id                      BIGSERIAL PRIMARY KEY,

    -- Setting identification
    category                VARCHAR(100) NOT NULL, -- 'clinic', 'scheduling', 'billing', 'communication', etc.
    setting_key             VARCHAR(200) NOT NULL, -- Unique key within category
    setting_name            VARCHAR(200) NOT NULL, -- Human-readable name
    description             TEXT,

    -- Value
    value_type              VARCHAR(50) NOT NULL, -- 'string', 'number', 'boolean', 'json', 'date', 'time'
    value_string            TEXT,
    value_number            DECIMAL(20,4),
    value_boolean           BOOLEAN,
    value_json              JSONB,
    value_date              DATE,
    value_time              TIME,

    -- Default value (for reset)
    default_value           TEXT,

    -- Validation
    validation_rules        JSONB,
    /* Example:
    {
      "required": true,
      "min": 0,
      "max": 100,
      "pattern": "^[0-9]{10}$",
      "allowed_values": ["option1", "option2"]
    }
    */

    -- UI configuration
    ui_section              VARCHAR(100), -- Group settings in UI
    ui_order                INTEGER, -- Display order
    ui_input_type           VARCHAR(50), -- 'text', 'number', 'select', 'textarea', 'checkbox', 'color', 'file'
    ui_placeholder          VARCHAR(200),
    ui_help_text            TEXT,

    -- Multi-location support
    scope                   VARCHAR(50) DEFAULT 'global', -- 'global', 'location', 'provider'
    location_id             BIGINT REFERENCES locations(id),
    provider_id             BIGINT REFERENCES users(id),

    -- Access control
    requires_permission     VARCHAR(100), -- Permission required to modify
    is_system               BOOLEAN DEFAULT false, -- System settings cannot be deleted
    is_visible              BOOLEAN DEFAULT true, -- Show in UI
    is_encrypted            BOOLEAN DEFAULT false, -- Encrypt value (for API keys, passwords)

    -- Versioning
    version                 INTEGER DEFAULT 1,

    -- Metadata
    last_modified_by        BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(category, setting_key, scope, location_id, provider_id),
    INDEX idx_system_settings_category (category),
    INDEX idx_system_settings_key (setting_key),
    INDEX idx_system_settings_scope (scope),
    INDEX idx_system_settings_location (location_id)
);

-- Settings change history
CREATE TABLE system_settings_history (
    id                      BIGSERIAL PRIMARY KEY,
    setting_id              BIGINT REFERENCES system_settings(id) ON DELETE CASCADE,

    -- Change details
    old_value               TEXT,
    new_value               TEXT,
    change_reason           TEXT,

    -- Who changed it
    changed_by              BIGINT REFERENCES users(id),
    changed_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Rollback capability
    can_rollback            BOOLEAN DEFAULT true,

    INDEX idx_settings_history_setting (setting_id),
    INDEX idx_settings_history_changed_at (changed_at)
);

-- Configuration presets/templates
CREATE TABLE configuration_templates (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Template type
    template_type           VARCHAR(50) NOT NULL, -- 'clinic_type', 'specialty', 'region'
    -- Examples: 'general_practice', 'cardiology', 'argentina_default'

    -- Configuration data
    settings                JSONB NOT NULL,
    /* Example:
    {
      "clinic": {
        "clinic_name": "Clínica General",
        "time_zone": "America/Argentina/Buenos_Aires",
        "date_format": "DD/MM/YYYY"
      },
      "scheduling": {
        "default_appointment_duration": 30,
        "allow_overbooking": false
      },
      ...
    }
    */

    -- Usage
    is_default              BOOLEAN DEFAULT false,
    use_count               INTEGER DEFAULT 0,

    -- Status
    is_active               BOOLEAN DEFAULT true,
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_config_templates_type (template_type)
);
```

### 1.2 Configuration Categories

#### 1.2.1 Clinic Information Settings

```sql
-- Default clinic information settings
INSERT INTO system_settings (category, setting_key, setting_name, description, value_type, ui_section, ui_order, ui_input_type, is_system)
VALUES
-- Basic Information
('clinic', 'clinic_name', 'Nombre de la Clínica', 'Nombre completo de la institución médica', 'string', 'basic_info', 1, 'text', true),
('clinic', 'clinic_legal_name', 'Razón Social', 'Razón social para facturación AFIP', 'string', 'basic_info', 2, 'text', true),
('clinic', 'clinic_cuit', 'CUIT', 'CUIT de la clínica para AFIP', 'string', 'basic_info', 3, 'text', true),
('clinic', 'clinic_phone', 'Teléfono Principal', 'Número de contacto principal', 'string', 'basic_info', 4, 'text', true),
('clinic', 'clinic_email', 'Email de Contacto', 'Email principal de la clínica', 'string', 'basic_info', 5, 'text', true),
('clinic', 'clinic_website', 'Sitio Web', 'URL del sitio web', 'string', 'basic_info', 6, 'text', false),

-- Address
('clinic', 'clinic_address', 'Dirección', 'Dirección completa', 'string', 'address', 1, 'textarea', true),
('clinic', 'clinic_city', 'Ciudad', NULL, 'string', 'address', 2, 'text', true),
('clinic', 'clinic_province', 'Provincia', NULL, 'string', 'address', 3, 'select', true),
('clinic', 'clinic_postal_code', 'Código Postal', NULL, 'string', 'address', 4, 'text', true),
('clinic', 'clinic_country', 'País', NULL, 'string', 'address', 5, 'text', true),

-- Operating Hours
('clinic', 'clinic_hours_mon_fri', 'Horario Lunes a Viernes', 'Ej: 08:00-20:00', 'string', 'hours', 1, 'text', false),
('clinic', 'clinic_hours_sat', 'Horario Sábados', 'Ej: 09:00-13:00', 'string', 'hours', 2, 'text', false),
('clinic', 'clinic_hours_sun', 'Horario Domingos', 'Dejar vacío si cerrado', 'string', 'hours', 3, 'text', false),
('clinic', 'clinic_emergency_phone', 'Teléfono de Emergencias', 'Para urgencias fuera de horario', 'string', 'hours', 4, 'text', false),

-- Branding
('clinic', 'clinic_logo_url', 'Logo URL', 'URL del logo para documentos e informes', 'string', 'branding', 1, 'file', false),
('clinic', 'clinic_primary_color', 'Color Primario', 'Color principal de la interfaz', 'string', 'branding', 2, 'color', false),
('clinic', 'clinic_secondary_color', 'Color Secundario', NULL, 'string', 'branding', 3, 'color', false);
```

#### 1.2.2 Scheduling Settings

```sql
INSERT INTO system_settings (category, setting_key, setting_name, description, value_type, default_value, ui_section, ui_order, ui_input_type, is_system)
VALUES
-- Appointment Defaults
('scheduling', 'default_appointment_duration', 'Duración Predeterminada de Turno (minutos)', NULL, 'number', '30', 'appointments', 1, 'number', true),
('scheduling', 'min_appointment_duration', 'Duración Mínima de Turno (minutos)', NULL, 'number', '15', 'appointments', 2, 'number', true),
('scheduling', 'max_appointment_duration', 'Duración Máxima de Turno (minutos)', NULL, 'number', '120', 'appointments', 3, 'number', true),
('scheduling', 'appointment_buffer_time', 'Tiempo de Espera entre Turnos (minutos)', 'Buffer entre citas consecutivas', 'number', '5', 'appointments', 4, 'number', false),

-- Booking Rules
('scheduling', 'advance_booking_min_hours', 'Reserva Anticipada Mínima (horas)', 'Horas mínimas de anticipación para agendar', 'number', '2', 'booking_rules', 1, 'number', true),
('scheduling', 'advance_booking_max_days', 'Reserva Anticipada Máxima (días)', 'Días máximos de anticipación', 'number', '90', 'booking_rules', 2, 'number', true),
('scheduling', 'allow_same_day_booking', 'Permitir Reserva el Mismo Día', NULL, 'boolean', 'true', 'booking_rules', 3, 'checkbox', false),
('scheduling', 'allow_overbooking', 'Permitir Sobrecupo', NULL, 'boolean', 'false', 'booking_rules', 4, 'checkbox', false),
('scheduling', 'max_overbook_per_day', 'Máximo Sobrecupo por Día', 'Si sobrecupo permitido', 'number', '3', 'booking_rules', 5, 'number', false),

-- Cancellation Policy
('scheduling', 'cancellation_notice_hours', 'Aviso de Cancelación Requerido (horas)', 'Horas mínimas para cancelar sin penalización', 'number', '24', 'cancellation', 1, 'number', true),
('scheduling', 'cancellation_fee_enabled', 'Cargo por Cancelación Tardía', NULL, 'boolean', 'false', 'cancellation', 2, 'checkbox', false),
('scheduling', 'cancellation_fee_amount', 'Monto del Cargo por Cancelación', NULL, 'number', '500', 'cancellation', 3, 'number', false),
('scheduling', 'no_show_fee_enabled', 'Cargo por Inasistencia', NULL, 'boolean', 'false', 'cancellation', 4, 'checkbox', false),
('scheduling', 'no_show_fee_amount', 'Monto del Cargo por Inasistencia', NULL, 'number', '1000', 'cancellation', 5, 'number', false),

-- Working Hours
('scheduling', 'work_week_start_day', 'Día de Inicio de Semana Laboral', '1=Lunes, 7=Domingo', 'number', '1', 'work_hours', 1, 'select', true),
('scheduling', 'work_day_start_time', 'Inicio de Jornada Laboral', NULL, 'time', '08:00', 'work_hours', 2, 'time', true),
('scheduling', 'work_day_end_time', 'Fin de Jornada Laboral', NULL, 'time', '20:00', 'work_hours', 3, 'time', true),
('scheduling', 'lunch_break_start', 'Inicio de Almuerzo', NULL, 'time', '13:00', 'work_hours', 4, 'time', false),
('scheduling', 'lunch_break_end', 'Fin de Almuerzo', NULL, 'time', '14:00', 'work_hours', 5, 'time', false),

-- Waitlist
('scheduling', 'waitlist_enabled', 'Activar Lista de Espera', NULL, 'boolean', 'true', 'waitlist', 1, 'checkbox', false),
('scheduling', 'waitlist_auto_offer', 'Ofrecer Turnos Cancelados Automáticamente', NULL, 'boolean', 'true', 'waitlist', 2, 'checkbox', false),
('scheduling', 'waitlist_offer_expiry_hours', 'Expiración de Oferta de Turno (horas)', NULL, 'number', '4', 'waitlist', 3, 'number', false);
```

#### 1.2.3 Billing & Financial Settings

```sql
INSERT INTO system_settings (category, setting_key, setting_name, description, value_type, default_value, ui_section, ui_order, ui_input_type, is_encrypted, is_system)
VALUES
-- General Billing
('billing', 'currency', 'Moneda', NULL, 'string', 'ARS', 'general', 1, 'select', false, true),
('billing', 'tax_id_type', 'Tipo de Identificación Fiscal', NULL, 'string', 'CUIT', 'general', 2, 'text', false, true),
('billing', 'apply_vat', 'Aplicar IVA', NULL, 'boolean', 'false', 'general', 3, 'checkbox', false, false),
('billing', 'vat_rate', 'Tasa de IVA (%)', NULL, 'number', '21', 'general', 4, 'number', false, false),
('billing', 'invoice_due_days', 'Días de Vencimiento de Facturas', NULL, 'number', '30', 'general', 5, 'number', false, false),

-- AFIP Configuration
('billing', 'afip_environment', 'Entorno AFIP', 'testing o production', 'string', 'testing', 'afip', 1, 'select', false, true),
('billing', 'afip_cuit', 'CUIT AFIP', NULL, 'string', NULL, 'afip', 2, 'text', false, true),
('billing', 'afip_punto_venta', 'Punto de Venta AFIP', NULL, 'number', '1', 'afip', 3, 'number', false, true),
('billing', 'afip_certificate_path', 'Ruta del Certificado AFIP', NULL, 'string', NULL, 'afip', 4, 'file', false, true),
('billing', 'afip_private_key_path', 'Ruta de Clave Privada AFIP', NULL, 'string', NULL, 'afip', 5, 'file', true, true),
('billing', 'afip_wsaa_url', 'URL WSAA AFIP', NULL, 'string', 'https://wsaahomo.afip.gov.ar/ws/services/LoginCms', 'afip', 6, 'text', false, true),
('billing', 'afip_wsfe_url', 'URL WSFE AFIP', NULL, 'string', 'https://wswhomo.afip.gov.ar/wsfev1/service.asmx', 'afip', 7, 'text', false, true),

-- Payment Methods
('billing', 'accept_cash', 'Aceptar Efectivo', NULL, 'boolean', 'true', 'payment_methods', 1, 'checkbox', false, false),
('billing', 'accept_debit_card', 'Aceptar Tarjeta de Débito', NULL, 'boolean', 'true', 'payment_methods', 2, 'checkbox', false, false),
('billing', 'accept_credit_card', 'Aceptar Tarjeta de Crédito', NULL, 'boolean', 'true', 'payment_methods', 3, 'checkbox', false, false),
('billing', 'accept_bank_transfer', 'Aceptar Transferencia Bancaria', NULL, 'boolean', 'true', 'payment_methods', 4, 'checkbox', false, false),
('billing', 'accept_mercadopago', 'Aceptar Mercado Pago', NULL, 'boolean', 'true', 'payment_methods', 5, 'checkbox', false, false),

-- Mercado Pago
('billing', 'mercadopago_enabled', 'Mercado Pago Activo', NULL, 'boolean', 'false', 'mercadopago', 1, 'checkbox', false, false),
('billing', 'mercadopago_environment', 'Entorno Mercado Pago', 'testing o production', 'string', 'testing', 'mercadopago', 2, 'select', false, false),
('billing', 'mercadopago_public_key', 'Public Key Mercado Pago', NULL, 'string', NULL, 'mercadopago', 3, 'text', false, false),
('billing', 'mercadopago_access_token', 'Access Token Mercado Pago', NULL, 'string', NULL, 'mercadopago', 4, 'text', true, false),

-- Invoice Settings
('billing', 'invoice_prefix', 'Prefijo de Factura', NULL, 'string', 'INV', 'invoices', 1, 'text', false, false),
('billing', 'invoice_starting_number', 'Número Inicial de Factura', NULL, 'number', '1', 'invoices', 2, 'number', false, false),
('billing', 'invoice_footer_text', 'Texto de Pie de Factura', NULL, 'string', NULL, 'invoices', 3, 'textarea', false, false),
('billing', 'auto_send_invoice_email', 'Enviar Factura por Email Automáticamente', NULL, 'boolean', 'true', 'invoices', 4, 'checkbox', false, false);
```

#### 1.2.4 Communication Settings

```sql
INSERT INTO system_settings (category, setting_key, setting_name, description, value_type, default_value, ui_section, ui_order, ui_input_type, is_encrypted, is_system)
VALUES
-- Email Configuration
('communication', 'email_enabled', 'Email Activo', NULL, 'boolean', 'true', 'email', 1, 'checkbox', false, false),
('communication', 'email_provider', 'Proveedor de Email', 'sendgrid, ses, smtp', 'string', 'sendgrid', 'email', 2, 'select', false, false),
('communication', 'email_from_address', 'Email Remitente', NULL, 'string', 'noreply@clinica.com', 'email', 3, 'text', false, true),
('communication', 'email_from_name', 'Nombre Remitente', NULL, 'string', 'Clínica Central', 'email', 4, 'text', false, true),
('communication', 'email_reply_to', 'Email para Respuestas', NULL, 'string', 'info@clinica.com', 'email', 5, 'text', false, false),

-- SendGrid
('communication', 'sendgrid_api_key', 'SendGrid API Key', NULL, 'string', NULL, 'sendgrid', 1, 'text', true, false),

-- SMS Configuration
('communication', 'sms_enabled', 'SMS Activo', NULL, 'boolean', 'true', 'sms', 1, 'checkbox', false, false),
('communication', 'sms_provider', 'Proveedor de SMS', 'twilio, nexmo, local', 'string', 'twilio', 'sms', 2, 'select', false, false),
('communication', 'sms_from_number', 'Número Remitente SMS', NULL, 'string', NULL, 'sms', 3, 'text', false, false),

-- Twilio
('communication', 'twilio_account_sid', 'Twilio Account SID', NULL, 'string', NULL, 'twilio', 1, 'text', false, false),
('communication', 'twilio_auth_token', 'Twilio Auth Token', NULL, 'string', NULL, 'twilio', 2, 'text', true, false),
('communication', 'twilio_phone_number', 'Twilio Phone Number', NULL, 'string', NULL, 'twilio', 3, 'text', false, false),

-- WhatsApp Configuration
('communication', 'whatsapp_enabled', 'WhatsApp Activo', NULL, 'boolean', 'false', 'whatsapp', 1, 'checkbox', false, false),
('communication', 'whatsapp_provider', 'Proveedor de WhatsApp', 'twilio, 360dialog, meta', 'string', 'twilio', 'whatsapp', 2, 'select', false, false),
('communication', 'whatsapp_from_number', 'Número WhatsApp', 'Formato: whatsapp:+54911...', 'string', NULL, 'whatsapp', 3, 'text', false, false),

-- Reminder Settings
('communication', 'reminders_enabled', 'Recordatorios Activos', NULL, 'boolean', 'true', 'reminders', 1, 'checkbox', false, false),
('communication', 'reminder_default_timing_1', 'Primer Recordatorio (horas antes)', NULL, 'number', '24', 'reminders', 2, 'number', false, false),
('communication', 'reminder_default_timing_2', 'Segundo Recordatorio (horas antes)', NULL, 'number', '2', 'reminders', 3, 'number', false, false),
('communication', 'reminder_default_channels', 'Canales Predeterminados', 'JSON array', 'json', '["sms", "email"]', 'reminders', 4, 'text', false, false),
('communication', 'enable_confirmation_workflow', 'Activar Confirmación de Turnos', NULL, 'boolean', 'true', 'reminders', 5, 'checkbox', false, false);
```

#### 1.2.5 Clinical Settings

```sql
INSERT INTO system_settings (category, setting_key, setting_name, description, value_type, default_value, ui_section, ui_order, ui_input_type, is_system)
VALUES
-- EHR Settings
('clinical', 'ehr_note_format', 'Formato de Notas Clínicas', 'soap, narrative, problem_oriented', 'string', 'soap', 'ehr', 1, 'select', false),
('clinical', 'require_finalize_encounter', 'Requerir Finalización de Consulta', NULL, 'boolean', 'true', 'ehr', 2, 'checkbox', false),
('clinical', 'allow_encounter_edit_after_finalize', 'Permitir Edición Después de Finalizar', NULL, 'boolean', 'false', 'ehr', 3, 'checkbox', false),
('clinical', 'encounter_edit_window_hours', 'Ventana de Edición (horas)', 'Horas permitidas para editar después de finalizar', 'number', '24', 'ehr', 4, 'number', false),

-- Vital Signs
('clinical', 'vital_signs_units_temperature', 'Unidad de Temperatura', 'celsius, fahrenheit', 'string', 'celsius', 'vitals', 1, 'select', false),
('clinical', 'vital_signs_units_weight', 'Unidad de Peso', 'kg, lbs', 'string', 'kg', 'vitals', 2, 'select', false),
('clinical', 'vital_signs_units_height', 'Unidad de Altura', 'cm, inches', 'string', 'cm', 'vitals', 3, 'select', false),

-- Prescriptions
('clinical', 'enable_electronic_prescriptions', 'Activar Recetas Electrónicas', NULL, 'boolean', 'true', 'prescriptions', 1, 'checkbox', false),
('clinical', 'require_diagnosis_for_prescription', 'Requerir Diagnóstico para Recetar', NULL, 'boolean', 'true', 'prescriptions', 2, 'checkbox', false),
('clinical', 'check_drug_interactions', 'Verificar Interacciones Medicamentosas', NULL, 'boolean', 'true', 'prescriptions', 3, 'checkbox', false),
('clinical', 'check_allergies', 'Verificar Alergias', NULL, 'boolean', 'true', 'prescriptions', 4, 'checkbox', false),
('clinical', 'default_prescription_refills', 'Número Predeterminado de Recargas', NULL, 'number', '0', 'prescriptions', 5, 'number', false),

-- Laboratory
('clinical', 'lab_integration_enabled', 'Integración de Laboratorio Activa', NULL, 'boolean', 'false', 'laboratory', 1, 'checkbox', false),
('clinical', 'lab_integration_type', 'Tipo de Integración', 'hl7, api, manual', 'string', 'manual', 'laboratory', 2, 'select', false),
('clinical', 'auto_flag_critical_results', 'Marcar Resultados Críticos Automáticamente', NULL, 'boolean', 'true', 'laboratory', 3, 'checkbox', false),
('clinical', 'require_result_review', 'Requerir Revisión de Resultados', NULL, 'boolean', 'true', 'laboratory', 4, 'checkbox', false);
```

#### 1.2.6 Regional & Localization Settings

```sql
INSERT INTO system_settings (category, setting_key, setting_name, description, value_type, default_value, ui_section, ui_order, ui_input_type, is_system)
VALUES
-- Regional Settings
('regional', 'country', 'País', NULL, 'string', 'Argentina', 'regional', 1, 'select', true),
('regional', 'timezone', 'Zona Horaria', NULL, 'string', 'America/Argentina/Buenos_Aires', 'regional', 2, 'select', true),
('regional', 'language', 'Idioma', NULL, 'string', 'es', 'regional', 3, 'select', true),
('regional', 'date_format', 'Formato de Fecha', NULL, 'string', 'DD/MM/YYYY', 'regional', 4, 'select', true),
('regional', 'time_format', 'Formato de Hora', '12h or 24h', 'string', '24h', 'regional', 5, 'select', true),
('regional', 'first_day_of_week', 'Primer Día de la Semana', '0=Sunday, 1=Monday', 'number', '1', 'regional', 6, 'select', true),

-- Number Formatting
('regional', 'decimal_separator', 'Separador Decimal', NULL, 'string', ',', 'formatting', 1, 'select', false),
('regional', 'thousands_separator', 'Separador de Miles', NULL, 'string', '.', 'formatting', 2, 'select', false),
('regional', 'currency_symbol', 'Símbolo de Moneda', NULL, 'string', '$', 'formatting', 3, 'text', false),
('regional', 'currency_position', 'Posición del Símbolo', 'before or after', 'string', 'before', 'formatting', 4, 'select', false);
```

#### 1.2.7 Security Settings

```sql
INSERT INTO system_settings (category, setting_key, setting_name, description, value_type, default_value, ui_section, ui_order, ui_input_type, requires_permission, is_system)
VALUES
-- Authentication
('security', 'password_min_length', 'Longitud Mínima de Contraseña', NULL, 'number', '8', 'authentication', 1, 'number', 'settings:update', true),
('security', 'password_require_uppercase', 'Requerir Mayúsculas', NULL, 'boolean', 'true', 'authentication', 2, 'checkbox', 'settings:update', true),
('security', 'password_require_lowercase', 'Requerir Minúsculas', NULL, 'boolean', 'true', 'authentication', 3, 'checkbox', 'settings:update', true),
('security', 'password_require_numbers', 'Requerir Números', NULL, 'boolean', 'true', 'authentication', 4, 'checkbox', 'settings:update', true),
('security', 'password_require_special', 'Requerir Caracteres Especiales', NULL, 'boolean', 'true', 'authentication', 5, 'checkbox', 'settings:update', true),
('security', 'password_expiry_days', 'Expiración de Contraseña (días)', '0 = nunca expira', 'number', '90', 'authentication', 6, 'number', 'settings:update', false),
('security', 'mfa_enabled', 'Autenticación de Dos Factores', NULL, 'boolean', 'false', 'authentication', 7, 'checkbox', 'settings:update', false),
('security', 'mfa_required_for_roles', 'Roles que Requieren MFA', 'JSON array', 'json', '["ADMIN", "DOCTOR"]', 'authentication', 8, 'text', 'settings:update', false),

-- Session Management
('security', 'session_timeout_minutes', 'Tiempo de Sesión (minutos)', NULL, 'number', '480', 'sessions', 1, 'number', 'settings:update', true),
('security', 'max_concurrent_sessions', 'Sesiones Concurrentes Máximas', NULL, 'number', '3', 'sessions', 2, 'number', 'settings:update', false),
('security', 'auto_logout_on_close', 'Cerrar Sesión al Cerrar Navegador', NULL, 'boolean', 'false', 'sessions', 3, 'checkbox', 'settings:update', false),

-- Login Security
('security', 'max_login_attempts', 'Intentos de Login Máximos', NULL, 'number', '5', 'login_security', 1, 'number', 'settings:update', true),
('security', 'login_lockout_duration_minutes', 'Duración de Bloqueo (minutos)', NULL, 'number', '30', 'login_security', 2, 'number', 'settings:update', true),
('security', 'enable_ip_whitelist', 'Activar Lista Blanca de IPs', NULL, 'boolean', 'false', 'login_security', 3, 'checkbox', 'settings:update', false),
('security', 'allowed_ip_addresses', 'IPs Permitidas', 'JSON array', 'json', '[]', 'login_security', 4, 'textarea', 'settings:update', false),

-- Audit & Compliance
('security', 'enable_audit_logging', 'Activar Logging de Auditoría', NULL, 'boolean', 'true', 'audit', 1, 'checkbox', 'settings:update', true),
('security', 'audit_retention_days', 'Retención de Logs (días)', NULL, 'number', '2555', 'audit', 2, 'number', 'settings:update', true),
('security', 'log_all_data_access', 'Registrar Todo Acceso a Datos', NULL, 'boolean', 'true', 'audit', 3, 'checkbox', 'settings:update', true),
('security', 'sensitive_actions_require_reason', 'Acciones Sensibles Requieren Justificación', NULL, 'boolean', 'true', 'audit', 4, 'checkbox', 'settings:update', true);
```

---

## 2. MULTI-LOCATION CONFIGURATION

### 2.1 Location-Specific Settings

```sql
-- Locations table (enhanced)
ALTER TABLE locations ADD COLUMN IF NOT EXISTS configuration JSONB;
ALTER TABLE locations ADD COLUMN IF NOT EXISTS inherits_from_global BOOLEAN DEFAULT true;
ALTER TABLE locations ADD COLUMN IF NOT EXISTS custom_settings_count INTEGER DEFAULT 0;

-- Location-specific settings example
-- Create setting for specific location (overrides global)
INSERT INTO system_settings (
    category, setting_key, setting_name, value_type, value_number,
    scope, location_id
) VALUES (
    'scheduling', 'default_appointment_duration', 'Duración Predeterminada de Turno',
    'number', 45,
    'location', 2  -- Location ID 2 has different default duration
);

-- View for effective settings (considers location overrides)
CREATE VIEW effective_settings AS
SELECT
    COALESCE(ls.id, gs.id) as id,
    gs.category,
    gs.setting_key,
    gs.setting_name,
    COALESCE(ls.value_string, gs.value_string) as value_string,
    COALESCE(ls.value_number, gs.value_number) as value_number,
    COALESCE(ls.value_boolean, gs.value_boolean) as value_boolean,
    COALESCE(ls.value_json, gs.value_json) as value_json,
    COALESCE(ls.value_date, gs.value_date) as value_date,
    COALESCE(ls.value_time, gs.value_time) as value_time,
    COALESCE(ls.location_id, 0) as location_id,
    CASE WHEN ls.id IS NOT NULL THEN 'location' ELSE 'global' END as source
FROM system_settings gs
LEFT JOIN system_settings ls
    ON ls.category = gs.category
    AND ls.setting_key = gs.setting_key
    AND ls.scope = 'location'
WHERE gs.scope = 'global';
```

### 2.2 Location Settings UI

```
┌──────────────────────────────────────────────────────────────────┐
│ Configuración por Ubicación                            [Guardar] │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Ubicación: [Sede Centro ▼]                                      │
│                                                                  │
│ ☑ Heredar configuración global                                  │
│                                                                  │
│ Configuraciones Personalizadas (3):                             │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │                                                            │ │
│ │ Horario de Atención                                        │ │
│ │ ├─ Lunes a Viernes: 07:00 - 21:00  [Usar global] [Editar]│ │
│ │ └─ Sábados: 08:00 - 14:00           [Usar global] [Editar]│ │
│ │                                                            │ │
│ │ Facturación                                                │ │
│ │ └─ Punto de Venta AFIP: 2           [Usar global] [Editar]│ │
│ │                                                            │ │
│ │ Recordatorios                                              │ │
│ │ └─ Canales: SMS, Email              [Usar global] [Editar]│ │
│ │                                                            │ │
│ └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ [+ Personalizar Configuración]                                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. CONFIGURATION MANAGEMENT UI

### 3.1 Settings Dashboard

```
┌──────────────────────────────────────────────────────────────────┐
│ Configuración del Sistema                       [Buscar: ______]│
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ ┌──────────────┐  ┌──────────────────────────────────────────┐ │
│ │ Categorías   │  │ Información de la Clínica                 │ │
│ ├──────────────┤  ├──────────────────────────────────────────┤ │
│ │► Clínica     │  │                                          │ │
│ │  Programación│  │ Información Básica                       │ │
│ │  Facturación │  │ ┌────────────────────────────────────┐   │ │
│ │  Comunicación│  │ │ Nombre de la Clínica:              │   │ │
│ │  Clínico     │  │ │ [Clínica Central               ]   │   │ │
│ │  Regional    │  │ │                                    │   │ │
│ │  Seguridad   │  │ │ Razón Social:                      │   │ │
│ │  Avanzado    │  │ │ [Clínica Central S.A.          ]   │   │ │
│ └──────────────┘  │ │                                    │   │ │
│                   │ │ CUIT:                              │   │ │
│                   │ │ [30-12345678-9                 ]   │   │ │
│                   │ │                                    │   │ │
│                   │ │ Teléfono:                          │   │ │
│                   │ │ [011-4567-8900                 ]   │   │ │
│                   │ │                                    │   │ │
│                   │ │ Email:                             │   │ │
│                   │ │ [info@clinicacentral.com       ]   │   │ │
│                   │ └────────────────────────────────────┘   │ │
│                   │                                          │ │
│                   │ Dirección                                │ │
│                   │ ┌────────────────────────────────────┐   │ │
│                   │ │ Dirección Completa:                │   │ │
│                   │ │ [Av. Corrientes 1234, Piso 3   ]   │   │ │
│                   │ │                                    │   │ │
│                   │ │ Ciudad:        Provincia:          │   │ │
│                   │ │ [Buenos Aires] [Buenos Aires  ▼]   │   │ │
│                   │ │                                    │   │ │
│                   │ │ Código Postal:                     │   │ │
│                   │ │ [C1043           ]                 │   │ │
│                   │ └────────────────────────────────────┘   │ │
│                   │                                          │ │
│                   │                   [Cancelar] [Guardar]   │ │
│                   └──────────────────────────────────────────┘ │
│                                                                  │
│ Cambios Recientes:                                               │
│ • Punto de Venta AFIP cambiado de 1 a 2 - hace 2 horas         │
│ • Recordatorios SMS activados - hace 1 día                      │
│ • Logo actualizado - hace 3 días                                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 Configuration Import/Export

```sql
-- Configuration export function
CREATE TABLE configuration_exports (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,
    description             TEXT,

    -- Export scope
    export_scope            VARCHAR(50) NOT NULL, -- 'all', 'category', 'location'
    categories              VARCHAR(100)[],
    location_id             BIGINT REFERENCES locations(id),

    -- Export data
    configuration_data      JSONB NOT NULL,
    settings_count          INTEGER,

    -- Metadata
    exported_by             BIGINT REFERENCES users(id),
    exported_at             TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_config_exports_scope (export_scope)
);

-- Import log
CREATE TABLE configuration_imports (
    id                      BIGSERIAL PRIMARY KEY,
    export_id               BIGINT REFERENCES configuration_exports(id),

    -- Import details
    import_type             VARCHAR(50), -- 'full_replace', 'merge', 'preview_only'

    -- Results
    settings_imported       INTEGER,
    settings_skipped        INTEGER,
    settings_failed         INTEGER,

    -- Validation
    validation_errors       JSONB,
    conflicts_detected      JSONB,
    conflict_resolution     VARCHAR(50), -- 'keep_existing', 'overwrite', 'manual'

    -- Status
    status                  VARCHAR(50) DEFAULT 'pending',
    -- 'pending', 'validating', 'in_progress', 'completed', 'failed', 'rolled_back'

    -- Rollback
    can_rollback            BOOLEAN DEFAULT true,
    previous_configuration  JSONB, -- Backup before import
    rolled_back_at          TIMESTAMP,

    imported_by             BIGINT REFERENCES users(id),
    imported_at             TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_config_imports_status (status)
);
```

### 3.3 Configuration Templates

```sql
-- Default configuration templates for Argentina
INSERT INTO configuration_templates (name, description, template_type, settings)
VALUES
-- General Practice Template
('Consultorio General - Argentina', 'Configuración predeterminada para consultorios de medicina general en Argentina', 'general_practice',
'{
  "clinic": {
    "country": "Argentina",
    "timezone": "America/Argentina/Buenos_Aires",
    "language": "es",
    "date_format": "DD/MM/YYYY",
    "currency": "ARS"
  },
  "scheduling": {
    "default_appointment_duration": 30,
    "advance_booking_min_hours": 2,
    "advance_booking_max_days": 90,
    "allow_overbooking": false,
    "cancellation_notice_hours": 24
  },
  "billing": {
    "apply_vat": false,
    "invoice_due_days": 30,
    "afip_environment": "testing"
  },
  "communication": {
    "reminders_enabled": true,
    "reminder_default_timing_1": 24,
    "reminder_default_timing_2": 2,
    "reminder_default_channels": ["sms", "email"]
  }
}'::jsonb),

-- Specialty Clinic Template
('Clínica de Especialidad - Argentina', 'Para clínicas especializadas (cardiología, dermatología, etc.)', 'specialty',
'{
  "clinic": {
    "country": "Argentina",
    "timezone": "America/Argentina/Buenos_Aires",
    "language": "es"
  },
  "scheduling": {
    "default_appointment_duration": 45,
    "advance_booking_max_days": 180,
    "allow_overbooking": true,
    "max_overbook_per_day": 3,
    "cancellation_fee_enabled": true,
    "cancellation_fee_amount": 1000
  },
  "clinical": {
    "require_finalize_encounter": true,
    "check_drug_interactions": true,
    "require_diagnosis_for_prescription": true
  }
}'::jsonb);
```

---

## 4. CONFIGURATION VALIDATION & BUSINESS RULES

### 4.1 Validation Rules

```javascript
/**
 * Configuration validation system
 */
const validationRules = {
  'scheduling.default_appointment_duration': {
    type: 'number',
    min: 5,
    max: 480,
    required: true,
    errorMessage: 'La duración debe estar entre 5 y 480 minutos'
  },

  'clinic.clinic_cuit': {
    type: 'string',
    pattern: /^\d{2}-\d{8}-\d{1}$/,
    required: true,
    errorMessage: 'CUIT debe tener formato XX-XXXXXXXX-X'
  },

  'clinic.clinic_email': {
    type: 'string',
    pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
    required: true,
    errorMessage: 'Email inválido'
  },

  'billing.afip_punto_venta': {
    type: 'number',
    min: 1,
    max: 9999,
    required: true,
    errorMessage: 'Punto de venta debe estar entre 1 y 9999'
  },

  'security.password_min_length': {
    type: 'number',
    min: 6,
    max: 32,
    required: true,
    errorMessage: 'Longitud de contraseña debe estar entre 6 y 32'
  }
};

/**
 * Validate setting value
 */
async function validateSetting(settingKey, value, category) {
  const fullKey = `${category}.${settingKey}`;
  const rule = validationRules[fullKey];

  if (!rule) return { valid: true };

  // Type validation
  if (rule.type === 'number') {
    const num = parseFloat(value);
    if (isNaN(num)) {
      return { valid: false, error: 'Debe ser un número' };
    }
    if (rule.min !== undefined && num < rule.min) {
      return { valid: false, error: `Debe ser mayor o igual a ${rule.min}` };
    }
    if (rule.max !== undefined && num > rule.max) {
      return { valid: false, error: `Debe ser menor o igual a ${rule.max}` };
    }
  }

  // Pattern validation
  if (rule.pattern && !rule.pattern.test(value)) {
    return { valid: false, error: rule.errorMessage };
  }

  // Required validation
  if (rule.required && !value) {
    return { valid: false, error: 'Este campo es requerido' };
  }

  return { valid: true };
}
```

### 4.2 Configuration Dependencies

```javascript
/**
 * Configuration dependencies
 * Some settings depend on others being set first
 */
const dependencies = {
  'billing.mercadopago_access_token': {
    requires: ['billing.mercadopago_enabled'],
    condition: (settings) => settings['billing.mercadopago_enabled'] === true,
    errorMessage: 'Debe activar Mercado Pago primero'
  },

  'communication.sendgrid_api_key': {
    requires: ['communication.email_enabled', 'communication.email_provider'],
    condition: (settings) =>
      settings['communication.email_enabled'] === true &&
      settings['communication.email_provider'] === 'sendgrid',
    errorMessage: 'Requerido cuando SendGrid está seleccionado como proveedor'
  },

  'scheduling.max_overbook_per_day': {
    requires: ['scheduling.allow_overbooking'],
    condition: (settings) => settings['scheduling.allow_overbooking'] === true,
    errorMessage: 'Sobrecupo debe estar activado'
  }
};

/**
 * Check configuration dependencies
 */
async function checkDependencies(settingKey, category, allSettings) {
  const fullKey = `${category}.${settingKey}`;
  const dependency = dependencies[fullKey];

  if (!dependency) return { valid: true };

  // Check if required settings exist and condition is met
  const requiredSettings = {};
  for (const reqKey of dependency.requires) {
    requiredSettings[reqKey] = allSettings[reqKey];
  }

  if (!dependency.condition(requiredSettings)) {
    return {
      valid: false,
      error: dependency.errorMessage,
      requiredSettings: dependency.requires
    };
  }

  return { valid: true };
}
```

---

## 5. CONFIGURATION BACKUP & RESTORE

### 5.1 Automatic Configuration Backups

```sql
CREATE TABLE configuration_backups (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) NOT NULL,

    -- Backup type
    backup_type             VARCHAR(50) NOT NULL, -- 'automatic', 'manual', 'pre_import', 'pre_upgrade'

    -- Configuration snapshot
    configuration_snapshot  JSONB NOT NULL,
    settings_count          INTEGER,

    -- Metadata
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Restore capability
    can_restore             BOOLEAN DEFAULT true,
    restored_at             TIMESTAMP,
    restored_by             BIGINT REFERENCES users(id),

    INDEX idx_config_backups_type (backup_type),
    INDEX idx_config_backups_created (created_at)
);

-- Automatic backup trigger (before significant changes)
CREATE OR REPLACE FUNCTION backup_configuration_before_change()
RETURNS TRIGGER AS $$
BEGIN
    -- Create backup before significant setting changes
    IF NEW.is_system = true OR OLD.value_string != NEW.value_string THEN
        INSERT INTO configuration_backups (
            name, backup_type, configuration_snapshot, settings_count, created_by
        )
        SELECT
            'Auto backup - ' || current_timestamp,
            'automatic',
            jsonb_agg(
                jsonb_build_object(
                    'category', category,
                    'setting_key', setting_key,
                    'value', COALESCE(value_string, value_number::text, value_boolean::text)
                )
            ),
            COUNT(*),
            NEW.last_modified_by
        FROM system_settings
        WHERE scope = 'global';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 5.2 Configuration Restore Process

```javascript
/**
 * Restore configuration from backup
 */
async function restoreConfiguration(backupId, options = {}) {
  const { dryRun = false, conflictResolution = 'keep_existing' } = options;

  // Get backup
  const backup = await ConfigurationBackup.findByPk(backupId);
  if (!backup || !backup.can_restore) {
    throw new Error('Backup cannot be restored');
  }

  const snapshot = backup.configuration_snapshot;
  const results = {
    restored: 0,
    skipped: 0,
    conflicts: []
  };

  // Create current backup first
  if (!dryRun) {
    await createConfigurationBackup('pre_restore', 'Before restoring from backup');
  }

  // Restore each setting
  for (const setting of snapshot) {
    const current = await SystemSetting.findOne({
      where: {
        category: setting.category,
        setting_key: setting.setting_key,
        scope: 'global'
      }
    });

    if (current) {
      // Check for conflicts
      if (current.value_string !== setting.value) {
        results.conflicts.push({
          key: `${setting.category}.${setting.setting_key}`,
          currentValue: current.value_string,
          backupValue: setting.value
        });

        if (conflictResolution === 'keep_existing') {
          results.skipped++;
          continue;
        }
      }

      // Update setting
      if (!dryRun) {
        await current.update({ value_string: setting.value });
      }
      results.restored++;
    }
  }

  return results;
}
```

---

## 6. ADVANCED CONFIGURATION FEATURES

### 6.1 Configuration Search & Filtering

```sql
-- Full-text search on settings
CREATE INDEX idx_settings_search ON system_settings
USING gin(to_tsvector('spanish',
    COALESCE(setting_name, '') || ' ' ||
    COALESCE(description, '') || ' ' ||
    COALESCE(category, '')
));

-- Search query
SELECT
    category,
    setting_key,
    setting_name,
    description,
    value_string,
    ts_rank(
        to_tsvector('spanish', setting_name || ' ' || COALESCE(description, '')),
        plainto_tsquery('spanish', 'recordatorio')
    ) as rank
FROM system_settings
WHERE to_tsvector('spanish', setting_name || ' ' || COALESCE(description, ''))
    @@ plainto_tsquery('spanish', 'recordatorio')
ORDER BY rank DESC;
```

### 6.2 Configuration Monitoring & Alerts

```sql
CREATE TABLE configuration_alerts (
    id                      BIGSERIAL PRIMARY KEY,

    -- Alert configuration
    setting_category        VARCHAR(100),
    setting_key             VARCHAR(200),

    -- Alert condition
    alert_condition         VARCHAR(50), -- 'value_changed', 'value_equals', 'value_below', 'value_above'
    threshold_value         TEXT,

    -- Notification
    alert_priority          VARCHAR(50) DEFAULT 'normal', -- 'low', 'normal', 'high', 'critical'
    notify_roles            VARCHAR(50)[], -- Roles to notify
    notify_users            BIGINT[], -- Specific users to notify
    notification_channels   VARCHAR(50)[], -- 'email', 'sms', 'in_app'

    -- Status
    is_active               BOOLEAN DEFAULT true,
    last_triggered_at       TIMESTAMP,

    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_config_alerts_setting (setting_category, setting_key)
);

-- Example: Alert when AFIP environment changes to production
INSERT INTO configuration_alerts (
    setting_category, setting_key, alert_condition, threshold_value,
    alert_priority, notify_roles, notification_channels
) VALUES (
    'billing', 'afip_environment', 'value_equals', 'production',
    'critical', ARRAY['ADMIN', 'BILLING_MANAGER'], ARRAY['email', 'sms', 'in_app']
);
```

### 6.3 Feature Flags

```sql
CREATE TABLE feature_flags (
    id                      BIGSERIAL PRIMARY KEY,
    name                    VARCHAR(200) UNIQUE NOT NULL,
    description             TEXT,

    -- Flag status
    is_enabled              BOOLEAN DEFAULT false,

    -- Rollout strategy
    rollout_percentage      INTEGER DEFAULT 0, -- 0-100
    rollout_users           BIGINT[], -- Specific users
    rollout_roles           VARCHAR(50)[], -- Specific roles
    rollout_locations       BIGINT[], -- Specific locations

    -- Environment
    enabled_environments    VARCHAR(50)[], -- 'development', 'testing', 'production'

    -- Dependencies
    requires_features       VARCHAR(200)[], -- Other features that must be enabled
    conflicts_with          VARCHAR(200)[], -- Mutually exclusive features

    -- Metadata
    created_by              BIGINT REFERENCES users(id),
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_feature_flags_enabled (is_enabled)
);

-- Example feature flags
INSERT INTO feature_flags (name, description, is_enabled, enabled_environments)
VALUES
('ai_clinical_decision_support', 'AI-powered clinical decision support using GPT-4', false, ARRAY['testing']),
('patient_portal', 'Patient self-service portal', true, ARRAY['production']),
('whatsapp_reminders', 'WhatsApp appointment reminders', false, ARRAY['testing']),
('telemedicine', 'Telemedicine video consultations', false, ARRAY['development']),
('prescription_auto_refill', 'Automatic prescription refill requests', false, ARRAY['testing']);
```

---

*System Configuration Specification Version: 1.0*
*Last Updated: 2025-11-16*
*Comprehensive configuration management for clinical system*
