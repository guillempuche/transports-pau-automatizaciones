# Esquemas de Tablas - Sistema OCR Transports Pau

Definición completa de tablas para el sistema de procesamiento de recibos.

---

## Arquitectura de Datos

```
    ┌─────────────────────┐         ┌─────────────────────┐
    │   recibos_entrada    │         │       recibos        │
    │   (LOG/TRACKING)    │────────▶│   (DATOS PARSEADOS) │
    │                     │   1:1   │                     │
    └─────────────────────┘         └─────────────────────┘
           │                                  │
           │                                  │
           ▼                                  ▼
    Trazabilidad completa            Datos limpios para
    Debug de errores                 contabilidad/oficina
    Estado del proceso               Exportable a ERP
```

---

## Tabla 1: recibos_entrada

Log de ficheros entrantes. Trazabilidad completa del proceso.

### Columnas

| Columna | Tipo | Requerido | Default | Descripcion |
|---------|------|-----------|---------|-------------|
| `id_entrada` | TEXT | Si | auto | PK. UUID o secuencial (RE00001) |
| `fichero_nombre` | TEXT | Si | - | Nombre original del fichero |
| `fichero_url` | URL | No | - | Link al fichero en Google Drive |
| `fichero_size_kb` | NUMBER | No | - | Tamaño en KB |
| `fichero_tipo` | TEXT | No | - | MIME type (image/jpeg, application/pdf) |
| `email_remitente` | TEXT | Si | - | Email del transportista |
| `email_asunto` | TEXT | No | - | Asunto del email original |
| `fecha_recepcion` | DATETIME | Si | now() | Cuando llego el email |
| `airparser_doc_id` | TEXT | No | - | ID del documento en Airparser |
| `estado` | ENUM | Si | "recibido" | Estado del procesamiento |
| `error_detalle` | TEXT | No | - | Mensaje de error si aplica |
| `reintentos` | NUMBER | No | 0 | Numero de reintentos de procesamiento |
| `created_at` | DATETIME | Si | now() | Timestamp de creacion |
| `updated_at` | DATETIME | Si | now() | Timestamp de ultima modificacion |

### Valores: estado

| Valor | Descripcion |
|-------|-------------|
| `recibido` | Fichero recibido, pendiente de procesar |
| `subiendo` | Subiendo a Google Drive |
| `enviado_airparser` | Enviado a Airparser, esperando OCR |
| `completado` | Procesado correctamente, datos en tabla recibos |
| `error` | Error en el procesamiento (ver error_detalle) |
| `descartado` | Fichero descartado (no es recibo, duplicado, etc.) |

### Google Sheets: Cabeceras

```
id_entrada | fichero_nombre | fichero_url | fichero_size_kb | fichero_tipo | email_remitente | email_asunto | fecha_recepcion | airparser_doc_id | estado | error_detalle | reintentos | created_at | updated_at
```

### SQL: CREATE TABLE

```sql
CREATE TABLE recibos_entrada (
    id_entrada VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    fichero_nombre VARCHAR(255) NOT NULL,
    fichero_url TEXT,
    fichero_size_kb INTEGER,
    fichero_tipo VARCHAR(100),
    email_remitente VARCHAR(255) NOT NULL,
    email_asunto VARCHAR(500),
    fecha_recepcion TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    airparser_doc_id VARCHAR(100),
    estado VARCHAR(20) NOT NULL DEFAULT 'recibido'
        CHECK (estado IN ('recibido', 'subiendo', 'enviado_airparser', 'completado', 'error', 'descartado')),
    error_detalle TEXT,
    reintentos INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_estado (estado),
    INDEX idx_fecha_recepcion (fecha_recepcion),
    INDEX idx_email_remitente (email_remitente),
    INDEX idx_airparser_doc_id (airparser_doc_id)
);
```

### Airtable: Campos

| Campo | Tipo Airtable | Configuracion |
|-------|---------------|---------------|
| id_entrada | Autonumber o Formula | Prefijo "RE" + numero |
| fichero_nombre | Single line text | - |
| fichero_url | URL | - |
| fichero_size_kb | Number | Integer |
| fichero_tipo | Single line text | - |
| email_remitente | Email | - |
| email_asunto | Long text | - |
| fecha_recepcion | Date | Include time |
| airparser_doc_id | Single line text | - |
| estado | Single select | 6 opciones |
| error_detalle | Long text | - |
| reintentos | Number | Integer, min 0 |
| created_at | Created time | - |
| updated_at | Last modified time | - |

---

## Tabla 2: recibos

Datos parseados de los recibos. Tabla principal para contabilidad.

### Columnas

| Columna | Tipo | Requerido | Default | Descripcion |
|---------|------|-----------|---------|-------------|
| **Identificadores** |
| `id` | TEXT | Si | auto | PK. UUID o secuencial (TK00001) |
| `id_entrada` | TEXT | No | - | FK a recibos_entrada |
| **Datos OCR** |
| `razon_social` | TEXT | No | - | Nombre del comercio/empresa emisora |
| `fecha` | DATE | No | - | Fecha de transaccion (YYYY-MM-DD) |
| `matricula` | TEXT | No | - | Matricula del vehiculo (normalizada) |
| `importe` | DECIMAL | No | - | Importe total en EUR |
| `categoria` | ENUM | Si | "otros" | Tipo de gasto |
| `pago` | ENUM | No | - | Quien paga |
| **Archivo** |
| `link_archivo` | URL | No | - | Enlace al archivo en Drive |
| **Control de revision** |
| `requiere_revision` | BOOLEAN | Si | false | TRUE si hay campos criticos vacios |
| `revisado` | BOOLEAN | Si | false | TRUE si un humano reviso/edito |
| `revisado_por` | TEXT | No | - | Email del usuario que reviso |
| `revisado_at` | DATETIME | No | - | Cuando se reviso |
| `error_msg` | TEXT | No | - | Motivo de requiere_revision |
| **Estado** |
| `estado` | ENUM | Si | "PROCESADO" | Estado general del recibo |
| `notes` | TEXT | No | - | Observaciones manuales |
| **Auditoria** |
| `created_at` | DATETIME | Si | now() | Timestamp de creacion |
| `updated_at` | DATETIME | Si | now() | Timestamp de ultima modificacion |
| `source` | TEXT | No | "airparser" | Origen de los datos |

### Valores: categoria

| Valor | Keywords (ES/CAT) | Descripcion |
|-------|-------------------|-------------|
| `gasoil` | gasoil, diesel, combustible, fuel, carburant, litros, surtidor | Combustible |
| `peajes` | peatge, peaje, autopista, toll, via-t, telepeaje, tunel | Peajes |
| `limpieza_vehiculos` | rentat, lavado, neteja, limpieza, autolavado, car wash | Lavado/Limpieza |
| `otros` | *(fallback)* | Otros gastos |

### Valores: pago

| Valor | Indicadores | Descripcion |
|-------|-------------|-------------|
| `empresa` | SOLRED, DKV, tarjeta empresa, "CLIENT: TRANSPORTS" | Pagado por empresa |
| `transportista` | efectivo, contado, metalico, efectiu | Pagado por conductor |
| *(vacio)* | Sin indicadores claros | Requiere revision |

### Valores: estado

| Valor | Descripcion |
|-------|-------------|
| `PROCESADO` | Recibo procesado correctamente |
| `REVISION` | Pendiente de revision manual |
| `ERROR` | Error en el procesamiento OCR |
| `DUPLICADO` | Recibo duplicado (descartado) |

### Google Sheets: Cabeceras

```
id | id_entrada | razon_social | fecha | matricula | importe | categoria | pago | link_archivo | requiere_revision | revisado | revisado_por | revisado_at | error_msg | estado | notes | created_at | updated_at | source
```

### SQL: CREATE TABLE

```sql
CREATE TABLE recibos (
    id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    id_entrada VARCHAR(36),

    -- Datos OCR
    razon_social VARCHAR(255),
    fecha DATE,
    matricula VARCHAR(20),
    importe DECIMAL(10,2),
    categoria VARCHAR(20) NOT NULL DEFAULT 'otros'
        CHECK (categoria IN ('gasoil', 'peajes', 'limpieza_vehiculos', 'otros')),
    pago VARCHAR(20)
        CHECK (pago IN ('empresa', 'transportista') OR pago IS NULL),

    -- Archivo
    link_archivo TEXT,

    -- Control de revision
    requiere_revision BOOLEAN NOT NULL DEFAULT FALSE,
    revisado BOOLEAN NOT NULL DEFAULT FALSE,
    revisado_por VARCHAR(255),
    revisado_at TIMESTAMP,
    error_msg TEXT,

    -- Estado
    estado VARCHAR(20) NOT NULL DEFAULT 'PROCESADO'
        CHECK (estado IN ('PROCESADO', 'REVISION', 'ERROR', 'DUPLICADO')),
    notes TEXT,

    -- Auditoria
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    source VARCHAR(50) DEFAULT 'airparser',

    -- Foreign key
    FOREIGN KEY (id_entrada) REFERENCES recibos_entrada(id_entrada),

    -- Indices
    INDEX idx_fecha (fecha),
    INDEX idx_categoria (categoria),
    INDEX idx_matricula (matricula),
    INDEX idx_estado (estado),
    INDEX idx_requiere_revision (requiere_revision),
    INDEX idx_revisado (revisado)
);

-- Trigger para marcar revisado automaticamente
DELIMITER //
CREATE TRIGGER recibos_auto_revisado
BEFORE UPDATE ON recibos
FOR EACH ROW
BEGIN
    IF (OLD.razon_social != NEW.razon_social OR
        OLD.fecha != NEW.fecha OR
        OLD.matricula != NEW.matricula OR
        OLD.importe != NEW.importe OR
        OLD.categoria != NEW.categoria OR
        OLD.pago != NEW.pago) THEN
        SET NEW.revisado = TRUE;
        SET NEW.revisado_at = CURRENT_TIMESTAMP;
    END IF;
END//
DELIMITER ;
```

### Airtable: Campos

| Campo | Tipo Airtable | Configuracion |
|-------|---------------|---------------|
| id | Autonumber o Formula | Prefijo "TK" + numero |
| id_entrada | Link to another record | Link a recibos_entrada |
| razon_social | Single line text | - |
| fecha | Date | Format YYYY-MM-DD |
| matricula | Single line text | - |
| importe | Currency | EUR, 2 decimales |
| categoria | Single select | 4 opciones + colores |
| pago | Single select | 2 opciones |
| link_archivo | URL | - |
| requiere_revision | Checkbox | - |
| revisado | Checkbox | - |
| revisado_por | Collaborator o Email | - |
| revisado_at | Date | Include time |
| error_msg | Long text | - |
| estado | Single select | 4 opciones + colores |
| notes | Long text | - |
| created_at | Created time | - |
| updated_at | Last modified time | - |
| source | Single line text | Default "airparser" |

---

## Logica de Campos Automaticos

### requiere_revision = TRUE cuando:

```
SI (fecha ES VACIO) O
   (importe ES VACIO O importe <= 0) O
   (razon_social ES VACIO) O
   (pago ES VACIO)
ENTONCES requiere_revision = TRUE
SINO requiere_revision = FALSE
```

### estado segun condiciones:

```
SI (error en procesamiento OCR)
   ENTONCES estado = "ERROR"
SI (requiere_revision = TRUE)
   ENTONCES estado = "REVISION"
SI (documento duplicado detectado)
   ENTONCES estado = "DUPLICADO"
SINO estado = "PROCESADO"
```

### revisado = TRUE automatico:

```
CUANDO usuario edita cualquier columna de datos:
  - razon_social, fecha, matricula, importe, categoria, pago
ENTONCES:
  - revisado = TRUE
  - revisado_por = email del usuario
  - revisado_at = timestamp actual
```

---

## Ejemplo de Datos

### recibos_entrada

| id_entrada | fichero_nombre | email_remitente | fecha_recepcion | airparser_doc_id | estado |
|------------|----------------|-----------------|-----------------|------------------|--------|
| RE00001 | gasoil_repsol.jpg | conductor1@gmail.com | 2024-12-15 10:30 | doc_abc123 | completado |
| RE00002 | peatge_ap7.pdf | conductor2@gmail.com | 2024-12-15 11:00 | doc_def456 | completado |
| RE00003 | recibo_borroso.jpg | conductor1@gmail.com | 2024-12-15 12:00 | doc_ghi789 | completado |
| RE00004 | corrupto.pdf | conductor3@gmail.com | 2024-12-15 13:00 | - | error |

### recibos

| id | id_entrada | razon_social | fecha | matricula | importe | categoria | pago | requiere_revision | revisado | estado |
|----|------------|------------|------|-----------|--------------|-----------|----------------|-------------------|----------|-------|
| TK00001 | RE00001 | REPSOL - E.S. MARTORELL | 2024-12-15 | 1234ABC | 125.50 | gasoil | empresa | FALSE | FALSE | PROCESADO |
| TK00002 | RE00002 | AUTOPISTAS AUMAR | 2024-12-15 | 1234ABC | 8.50 | peajes | empresa | FALSE | FALSE | PROCESADO |
| TK00003 | RE00003 | | 2024-12-16 | 5678DEF | 89.00 | gasoil | | TRUE | FALSE | REVISION |
| TK00004 | RE00003 | CEPSA | 2024-12-16 | 5678DEF | 89.00 | gasoil | transportista | TRUE | TRUE | REVISION |

> TK00004 es la misma fila que TK00003 despues de revision manual (revisado=TRUE)

---

## Nomenclatura de Archivos en Drive

### Formato

```
[YYYYMMDD]_[CATEGORIA]_[RAO_SOCIAL]_[IMPORT]_[MATRICULA]_[ID].[ext]
```

### Ejemplos

```
20241215_gasoil_REPSOL_125.50_1234ABC_TK00001.jpg
20241215_peajes_AUTOPISTAS-AUMAR_8.50_1234ABC_TK00002.pdf
20241216_gasoil_CEPSA_89.00_5678DEF_TK00003.jpg
20241216_otros_DESCONOCIDO_0.00_SIN-MATRICULA_TK00005.pdf
```

### Reglas de Normalizacion

| Campo | Regla |
|-------|-------|
| Fecha | YYYYMMDD sin separadores |
| Categoria | Minusculas, sin acentos |
| Rao social | Mayusculas, espacios → guiones, max 30 chars |
| Importe | Punto decimal, 2 decimales |
| Matricula | Sin espacios ni guiones, mayusculas |
| Si vacio | Usar placeholder: "SIN-MATRICULA", "DESCONOCIDO", "0.00" |

---

## Estructura de Carpetas en Drive

```
TransportsPau_Recibos/
├── 1_Entrada/                     <- Backups sin procesar (por fecha)
│   ├── 2024-12/
│   │   ├── 2024-12-15/
│   │   └── 2024-12-16/
│   └── 2025-01/
│
├── 2_Procesados/                  <- Recibos OK
│   ├── 2024/
│   │   ├── 12_Desembre/
│   │   │   ├── gasoil/
│   │   │   ├── peajes/
│   │   │   ├── limpieza_vehiculos/
│   │   │   └── otros/
│   │   └── ...
│   └── 2025/
│
├── 3_Revision/                    <- Pendientes de revision manual
│
├── 4_Errores/                     <- Recibos con error (ilegibles, corruptos)
│
└── _Config/                       <- Documentacion interna
    ├── esquema.md
    └── ejemplos/
```

---

## Apps Script: Auto-Revision en Google Sheets

```javascript
/**
 * Trigger onEdit para marcar automaticamente revisado=TRUE
 * cuando un usuario edita columnas de datos.
 *
 * Instalar: Extensions > Apps Script > Triggers > onEdit
 */
function onEdit(e) {
  var sheet = e.source.getActiveSheet();

  // Solo aplicar en hoja "recibos"
  if (sheet.getName() !== 'recibos') return;

  var editedCol = e.range.getColumn();
  var editedRow = e.range.getRow();

  // Columnas de datos (ajustar segun orden real)
  // Asumiendo: A=id, B=id_entrada, C=razon_social, D=fecha, E=matricula,
  //            F=importe, G=categoria, H=pago
  var dataColumns = [3, 4, 5, 6, 7, 8]; // C a H

  // Columnas de control
  var COL_REVISADO = 11;      // K
  var COL_REVISADO_POR = 12;  // L
  var COL_REVISADO_AT = 13;   // M

  // Si editaron una columna de datos
  if (dataColumns.includes(editedCol) && editedRow > 1) {
    var userEmail = Session.getActiveUser().getEmail() || 'manual';
    var now = new Date();

    sheet.getRange(editedRow, COL_REVISADO).setValue(true);
    sheet.getRange(editedRow, COL_REVISADO_POR).setValue(userEmail);
    sheet.getRange(editedRow, COL_REVISADO_AT).setValue(now);
  }
}

/**
 * Funcion auxiliar para validar y marcar requiere_revision
 * Ejecutar manualmente o con trigger onChange
 */
function validateRows() {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('recibos');
  var data = sheet.getDataRange().getValues();
  var headers = data[0];

  // Indices de columnas (ajustar segun orden)
  var COL_FECHA = headers.indexOf('fecha');
  var COL_IMPORT = headers.indexOf('importe');
  var COL_RAZON = headers.indexOf('razon_social');
  var COL_REQ_REV = headers.indexOf('requiere_revision');
  var COL_ERROR = headers.indexOf('error_msg');
  var COL_ESTADO = headers.indexOf('estado');

  for (var i = 1; i < data.length; i++) {
    var row = data[i];
    var errors = [];

    if (!row[COL_FECHA]) errors.push('fecha vacio');
    if (!row[COL_IMPORT] || row[COL_IMPORT] <= 0) errors.push('importe invalido');
    if (!row[COL_RAZON]) errors.push('razon_social vacio');

    var requiresReview = errors.length > 0;
    var rowNum = i + 1;

    sheet.getRange(rowNum, COL_REQ_REV + 1).setValue(requiresReview);
    sheet.getRange(rowNum, COL_ERROR + 1).setValue(errors.join(', '));
    sheet.getRange(rowNum, COL_ESTADO + 1).setValue(requiresReview ? 'REVISION' : 'PROCESADO');
  }
}
```

---

## Validaciones en Make.com

### Router con Filtros

```
Webhook Airparser
      │
      ▼
   [Router]
      │
      ├─── Filtro 1: Datos completos ───────────────▶ INSERT (requiere_revision=false)
      │    Condicion:
      │    fecha IS NOT EMPTY AND
      │    importe > 0 AND
      │    razon_social IS NOT EMPTY
      │
      └─── Filtro 2: Datos incompletos ─────────────▶ INSERT (requiere_revision=true)
           Condicion: (fallback/else)
           error_msg = campos vacios concatenados
```

### Formulas Make para error_msg

```
{{if(isEmpty(fecha); "fecha vacio, "; "")}}{{if(or(isEmpty(importe); importe <= 0); "importe invalido, "; "")}}{{if(isEmpty(razon_social); "razon_social vacio"; "")}}
```

---

## Checklist de Implementacion

### Google Sheets

- [ ] Crear hoja `recibos_entrada` con 14 columnas
- [ ] Crear hoja `recibos` con 19 columnas
- [ ] Configurar validacion de datos para enums (categoria, estado, etc.)
- [ ] Configurar formato condicional (rojo si requiere_revision=TRUE)
- [ ] Instalar Apps Script para auto-revision
- [ ] Configurar trigger onEdit

### Make.com

- [ ] `TPau_Recibos_Entrada`: Email → Drive → recibos_entrada → Airparser
- [ ] `TPau_Recibos_Procesar`: Webhook Airparser → validacion → recibos
- [ ] `TPau_Recibos_Revision`: Watch Changes → actualizar revisado
- [ ] Error handlers en cada escenario
- [ ] Alertas por email si > 5 errores/hora

### Airparser

- [ ] Crear inbox con esquema de 6 campos
- [ ] Configurar webhook hacia Make
- [ ] Probar con 3-5 recibos reales

### Google Drive

- [ ] Crear estructura de carpetas
- [ ] Permisos de acceso para Make
