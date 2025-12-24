# Flujo

Sistema de procesamiento automático de recibos de transportistas.

---

## Diagrama del Flujo

```
                                    FLUJO OCR - TRANSPORTS PAU
                                    ===========================

    TRANSPORTISTAS                          MAKE.COM                              AIRPARSER
    ══════════════                          ════════                              ═════════

    ┌─────────────────┐
    │  Transportista  │
    │  envia email    │
    │  con adjuntos   │
    │  (fotos/PDFs)   │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐         ┌─────────────────────────────────────────────────────────┐
    │   Gmail/Outlook │         │                        MAKE.COM                         │
    │   (inbox)       │────────▶│                                                         │
    └─────────────────┘         │  ┌─────────────────────────────────────────────────┐    │
                                │  │ 1. TRIGGER: Watch Emails                        │    │
                                │  │    - Solo emails con adjuntos                   │    │
                                │  │    - Extraer: remitente, hora, ficheros         │    │
                                │  └──────────────────────┬──────────────────────────┘    │
                                │                         │                               │
                                │                         ▼                               │
                                │  ┌─────────────────────────────────────────────────┐    │
                                │  │ 2. ITERATOR: Para cada adjunto                  │    │
                                │  │    - Separar imagenes inline y attachments      │    │
                                │  └──────────────────────┬──────────────────────────┘    │
                                │                         │                               │
                                │                         ▼                               │
                                │  ┌─────────────────────────────────────────────────┐    │
                                │  │ 3. LOG INICIAL en tabla "recibos_entrada"        │    │          ┌─────────────────┐
                                │  │    - fichero_nombre                             │    │          │                 │
                                │  │    - email_remitente                            │    │          │   AIRPARSER     │
                                │  │    - fecha_recepcion                            │    │          │   (OCR + LLM)   │
                                │  │    - estado = "procesando"                      │    │          │                 │
                                │  │    - id_entrada (para tracking)                 │    │          └────────┬────────┘
                                │  └──────────────────────┬──────────────────────────┘    │                   │
                                │                         │                               │                   │
                                │                         ▼                               │                   │
                                │  ┌─────────────────────────────────────────────────┐    │                   │
                                │  │ 4. UPLOAD a Airparser                           │────┼──────────────────▶│
                                │  │    POST /inboxes/{id}/upload                    │    │   file + meta     │
                                │  │    - file: adjunto                              │    │   (id_entrada)    │
                                │  │    - meta: {"id_entrada": "xxx"}                │    │                   │
                                │  └──────────────────────┬──────────────────────────┘    │                   │
                                │                         │                               │                   │
                                │                         │ (doc_id)                      │                   │
                                │                         ▼                               │                   │
                                │  ┌─────────────────────────────────────────────────┐    │                   │
                                │  │ 5. ACTUALIZAR estado en "recibos_entrada"        │    │                   │
                                │  │    - airparser_doc_id = doc_id                  │    │                   │
                                │  │    - estado = "en_airparser"                    │    │                   │
                                │  └─────────────────────────────────────────────────┘    │                   │
                                │                                                         │                   │
                                └─────────────────────────────────────────────────────────┘                   │
                                                                                                              │
                                                                                                              │
                                ┌─────────────────────────────────────────────────────────┐                   │
                                │               TPau_Recibos_Procesar                     │◀──────────────────┘
                                │                                                         │   POST webhook
                                │  ┌─────────────────────────────────────────────────┐    │   (datos parseados)
                                │  │ 6. WEBHOOK: Airparser envia datos parseados     │    │
                                │  │    - razon_social, fecha, matricula               │    │
                                │  │    - importe, categoria, pago         │    │
                                │  │    - __meta__.id_entrada (para vincular)        │    │
                                │  └──────────────────────┬──────────────────────────┘    │
                                │                         │                               │
                                │                         ▼                               │
                                │  ┌─────────────────────────────────────────────────┐    │
                                │  │ 7. VALIDAR datos                                │    │
                                │  │    - Campos obligatorios: fecha, importe        │    │
                                │  │    - Formato fecha YYYY-MM-DD                   │    │
                                │  │    - importe > 0                                │    │
                                │  └──────────────────────┬──────────────────────────┘    │
                                │                         │                               │
                                │            ┌────────────┴────────────┐                  │
                                │            ▼                         ▼                  │
                                │     ┌─────────────┐           ┌─────────────┐           │
                                │     │ VALIDACION  │           │ VALIDACION  │           │
                                │     │    OK       │           │   FALLA     │           │
                                │     └──────┬──────┘           └──────┬──────┘           │
                                │            │                         │                  │
                                │            ▼                         ▼                  │
                                │  ┌──────────────────┐      ┌──────────────────┐         │
                                │  │ 8a. INSERT en    │      │ 8b. INSERT en    │         │
                                │  │ tabla "recibos"   │      │ tabla "recibos"   │         │
                                │  │                  │      │                  │         │
                                │  │ - todos campos   │      │ - campos vacios  │         │
                                │  │ - requiere_      │      │ - requiere_      │         │
                                │  │   revision=false │      │   revision=true  │         │
                                │  │ - revisado=false │      │ - revisado=false │         │
                                │  │ - error_msg=null │      │ - error_msg=     │         │
                                │  │                  │      │   "campo X vacio"│         │
                                │  └──────────────────┘      └──────────────────┘         │
                                │                                                         │
                                └─────────────────────────────────────────────────────────┘


                                        TABLAS BASE DE DATOS
                                        ════════════════════

    ┌──────────────────────────────────────────────────────────────────────────────────────┐
    │                              recibos_entrada (LOG)                                    │
    ├──────────────────────────────────────────────────────────────────────────────────────┤
    │  id_entrada (PK)    │ UUID auto                                                      │
    │  fichero_nombre     │ "recibo_gasoil.jpg"                                            │
    │  fichero_url        │ URL al fichero original (Google Drive/storage)                 │
    │  email_remitente    │ "conductor@email.com"                                          │
    │  fecha_recepcion    │ 2024-12-24 10:30:00                                            │
    │  airparser_doc_id   │ "doc_abc123" (rellenar tras upload)                            │
    │  estado             │ "procesando" | "en_airparser" | "completado" | "error"         │
    │  error_detalle      │ Mensaje de error si aplica                                     │
    │  created_at         │ timestamp auto                                                 │
    └──────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              │ 1:1
                                              ▼
    ┌──────────────────────────────────────────────────────────────────────────────────────┐
    │                              recibos (DATOS PARSEADOS)                                │
    ├──────────────────────────────────────────────────────────────────────────────────────┤
    │  id (PK)            │ UUID auto                                                      │
    │  id_entrada (FK)    │ Vinculo a recibos_entrada                                       │
    │  ─────────────────  │ ───────────────────────────────────────────────────            │
    │  razon_social         │ "REPSOL ESTACIO MARTORELL"                                     │
    │  fecha              │ "2024-12-15" (YYYY-MM-DD)                                      │
    │  matricula          │ "1234ABC"                                                      │
    │  importe            │ 125.50                                                         │
    │  categoria          │ "gasoil" | "peajes" | "limpieza_vehiculos" | "otros"           │
    │  pago     │ "empresa" | "transportista" | null                             │
    │  ─────────────────  │ ───────────────────────────────────────────────────            │
    │  requiere_revision  │ true/false (auto: true si campos criticos vacios)              │
    │  revisado           │ false (auto-cambia a true si usuario edita fila)               │
    │  revisado_por       │ null | "usuari@email.com"                                      │
    │  revisado_at        │ null | timestamp                                               │
    │  error_msg          │ null | "importe vacio"                                         │
    │  ─────────────────  │ ───────────────────────────────────────────────────            │
    │  created_at         │ timestamp auto                                                 │
    │  updated_at         │ timestamp auto (trigger on update)                             │
    └──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Detalle de Pasos

### TPau_Recibos_Entrada (Email → Make → Airparser)

| Paso | Modulo Make | Accion |
|------|-------------|--------|
| 1 | Gmail/Outlook Trigger | Watch emails con adjuntos |
| 2 | Iterator | Iterar sobre cada adjunto |
| 3 | Google Sheets / DB | Insertar fila en `recibos_entrada` |
| 4 | HTTP Request | POST a Airparser `/upload` con meta |
| 5 | Google Sheets / DB | Actualizar `airparser_doc_id` y estado |

### TPau_Recibos_Procesar (Webhook Airparser → Make → DB)

| Paso | Modulo Make | Accion |
|------|-------------|--------|
| 6 | Webhook Trigger | Recibir datos parseados de Airparser |
| 7 | Router + Filters | Validar campos obligatorios |
| 8 | Google Sheets / DB | Insertar en `recibos` con flags |

---

## Columna "revisado" - Logica Automatica

```
                    FLUJO REVISION AUTOMATICA
                    =========================

    ┌─────────────────┐
    │ Datos entran    │
    │ de Airparser    │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────────────────────────┐
    │ Validar campos criticos:            │
    │ - fecha no vacio                    │
    │ - importe > 0                       │
    │ - razon_social no vacio               │
    └────────────────┬────────────────────┘
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
    ┌─────────┐            ┌─────────┐
    │ VALIDO  │            │ INVALIDO│
    └────┬────┘            └────┬────┘
         │                      │
         ▼                      ▼
    ┌─────────────────┐   ┌─────────────────┐
    │ requiere_       │   │ requiere_       │
    │ revision=false  │   │ revision=true   │
    │ revisado=false  │   │ revisado=false  │
    │ error_msg=null  │   │ error_msg=      │
    └─────────────────┘   │ "fecha vacio"   │
                          └─────────────────┘

    ═══════════════════════════════════════════════════════════════

    CUANDO USUARIO EDITA MANUALMENTE (via Google Sheets / App)
    ───────────────────────────────────────────────────────────

    ┌─────────────────┐       ┌──────────────────────────┐
    │ Usuario edita   │──────▶│ Trigger: On Row Update   │
    │ cualquier campo │       │ (Google Sheets/DB)       │
    │ de datos        │       └────────────┬─────────────┘
    └─────────────────┘                    │
                                           ▼
                               ┌───────────────────────┐
                               │ Escenario Make:       │
                               │ - revisado = true     │
                               │ - revisado_por = user │
                               │ - revisado_at = now() │
                               └───────────────────────┘
```

### Implementacion en Make

**Opcion A: Google Sheets con Apps Script**

```javascript
// Trigger onEdit en Google Sheets
function onEdit(e) {
  var sheet = e.source.getActiveSheet();
  if (sheet.getName() === 'recibos') {
    var row = e.range.getRow();
    var col_revisado = 10; // columna "revisado"
    var col_revisado_por = 11;
    var col_revisado_at = 12;

    // Solo si editan columnas de datos (no las de control)
    if (e.range.getColumn() < col_revisado) {
      sheet.getRange(row, col_revisado).setValue(true);
      sheet.getRange(row, col_revisado_por).setValue(Session.getActiveUser().getEmail());
      sheet.getRange(row, col_revisado_at).setValue(new Date());
    }
  }
}
```

**Opcion B: Make Watch Changes**

- Usar modulo "Google Sheets - Watch Changes"
- Filtrar: solo cambios en columnas de datos
- Actualizar columnas revisado/revisado_por/revisado_at

---

## Recomendaciones para Sistema Robusto

### 1. Manejo de Errores en Make

```
    ESTRUCTURA DE ERROR HANDLING
    ============================

    ┌─────────────────────┐
    │ Modulo principal    │
    └──────────┬──────────┘
               │
               ├──────────────────┐
               ▼                  ▼
    ┌──────────────────┐  ┌──────────────────┐
    │ Error Handler    │  │ Continua flujo   │
    │ (Break + Commit) │  │ normal           │
    └────────┬─────────┘  └──────────────────┘
             │
             ▼
    ┌──────────────────────────────┐
    │ Log error en recibos_entrada: │
    │ - estado = "error"           │
    │ - error_detalle = mensaje    │
    └──────────────────────────────┘
```

### 2. Reintentos Automaticos

| Error | Accion | Reintentos |
|-------|--------|------------|
| Airparser 429 (rate limit) | Esperar y reintentar | 3x con backoff |
| Airparser 500 | Reintentar | 3x |
| Webhook timeout | Reintentar | 2x |
| Validacion falla | No reintentar | Marcar requiere_revision |

### 3. Monitoreo y Alertas

```
    ALERTAS RECOMENDADAS
    ====================

    ┌─────────────────────────────────────────────────┐
    │ ALERTA 1: Errores acumulados                    │
    │ Si > 5 errores en 1 hora → Email a admin       │
    ├─────────────────────────────────────────────────┤
    │ ALERTA 2: Cola atascada                         │
    │ Si documentos en "procesando" > 30 min → Alerta│
    ├─────────────────────────────────────────────────┤
    │ ALERTA 3: Revision pendiente                    │
    │ Si > 10 filas con requiere_revision → Notificar│
    └─────────────────────────────────────────────────┘
```

### 4. Campos de Auditoria Recomendados

Añadir a ambas tablas:

| Campo | Tipo | Descripcion |
|-------|------|-------------|
| `created_at` | timestamp | Cuando se creo |
| `updated_at` | timestamp | Ultima modificacion |
| `procesado_por` | string | "make_scenario_1" / "manual" |
| `version` | integer | Para detectar conflictos |

### 5. Separacion de Tablas (Recomendado)

**Por que 2 tablas?**

```
recibos_entrada (LOG)          recibos (DATOS)
════════════════════          ══════════════
- Trazabilidad completa       - Datos limpios para contabilidad
- Debug de errores            - Vista para oficina
- Vinculo fichero original    - Exportable a ERP
- Estado del proceso          - Solo filas "exitosas"
```

### 6. Idempotencia

**Problema:** Un webhook puede llegar 2 veces (retry de Airparser).

**Solucion:**

```
Antes de INSERT en recibos:
  1. Buscar si existe fila con mismo airparser_doc_id
  2. Si existe → UPDATE en vez de INSERT
  3. Si no existe → INSERT
```

En Make: usar modulo "Search Row" antes de "Add Row"

### 7. Backup del Fichero Original

```
    ┌─────────────┐
    │ Adjunto     │
    │ del email   │
    └──────┬──────┘
           │
           ├──────────────────┐
           ▼                  ▼
    ┌─────────────────┐  ┌─────────────────┐
    │ Google Drive    │  │ Airparser       │
    │ (backup)        │  │ (procesado)     │
    │                 │  │                 │
    │ carpeta:        │  │                 │
    │ /recibos/2024/12 │  │                 │
    └─────────────────┘  └─────────────────┘
```

Guardar URL de Drive en `recibos_entrada.fichero_url`

### 8. Timeouts y Limites

| Operacion | Timeout recomendado |
|-----------|---------------------|
| Upload a Airparser | 60 segundos |
| Webhook response | Inmediato (200 OK) |
| Procesamiento Airparser | Esperar via webhook, no polling |

---

## Esquema Make Completo

```
    TPau_Recibos_Entrada
    ════════════════════

    [Gmail Watch]
         │
         ▼
    [Filter: tiene adjuntos?]──No──▶ [Stop]
         │
        Yes
         │
         ▼
    [Iterator: adjuntos]
         │
         ▼
    [Google Drive: subir fichero]
         │
         ▼
    [Google Sheets: INSERT recibos_entrada]
         │
         ▼
    [HTTP: POST Airparser /upload]
         │
         ├────[Error Handler]────▶ [Update estado="error"]
         │
         ▼
    [Google Sheets: UPDATE airparser_doc_id]


    TPau_Recibos_Procesar
    ═════════════════════

    [Webhook: Airparser]
         │
         ▼
    [Set Variables: extraer campos]
         │
         ▼
    [Router]
         │
         ├────[Filter: campos OK]────▶ [Sheets: INSERT recibos (requiere_revision=false)]
         │                                    │
         │                                    ▼
         │                            [Sheets: UPDATE recibos_entrada estado="completado"]
         │
         └────[Filter: campos KO]────▶ [Sheets: INSERT recibos (requiere_revision=true)]
                                              │
                                              ▼
                                       [Sheets: UPDATE recibos_entrada estado="completado"]


    TPau_Recibos_Revision
    ═════════════════════

    [Google Sheets: Watch Changes on "recibos"]
         │
         ▼
    [Filter: columna editada < col_revisado?]
         │
        Yes
         │
         ▼
    [Google Sheets: UPDATE fila]
      - revisado = true
      - revisado_por = {{user}}
      - revisado_at = {{now}}
```

---

## Siguiente Pasos

1. [ ] Crear inbox en Airparser con esquema de campos
2. [ ] Crear Google Sheets con estructura de tablas
3. [ ] Configurar `TPau_Recibos_Entrada` en Make
4. [ ] Configurar webhook en Airparser
5. [ ] Configurar `TPau_Recibos_Procesar` en Make
6. [ ] Configurar `TPau_Recibos_Revision` en Make
7. [ ] Prueba end-to-end con recibo real
8. [ ] Configurar alertas de error
