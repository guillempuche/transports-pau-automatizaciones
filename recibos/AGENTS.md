# Módulo de Recibos

## Propósito

Extracción automática de datos de recibos de conductores de camión mediante OCR.

```
Email (foto recibo) → Make.com → Airparser (OCR) → Webhook → Make.com → Google Sheets + Drive
```

---

## Arquitectura del Sistema

### Flujo de Datos

```
TRANSPORTISTA                   MAKE.COM                        AIRPARSER
─────────────                   ────────                        ─────────
Email con adjuntos
       │
       ▼
TPau_Recibos_Entrada ──────────────────────────────────────────▶ OCR + LLM
  1. Watch emails                                                    │
  2. Iterator adjuntos                                               │
  3. INSERT recibos_entrada                                          │
  4. POST /upload a Airparser                                        │
  5. UPDATE airparser_doc_id                                         │
                                                                     │
TPau_Recibos_Procesar ◀────────────────────────────────────────── Webhook
  6. Recibir datos parseados
  7. Validar campos
  8. INSERT/UPDATE recibos
```

### Tablas del Sistema

| Tabla | Propósito | Campos clave |
|-------|-----------|--------------|
| `recibos_entrada` | Log de ficheros entrantes (trazabilidad) | id_entrada, fichero_nombre, estado, airparser_doc_id |
| `recibos` | Datos parseados (contabilidad) | id, razon_social, fecha, matricula, importe, categoria, pago |

---

## Escenarios Make.com

| Escenario | Trigger | Función |
|-----------|---------|---------|
| `TPau_Recibos_Entrada` | Watch Emails (Gmail/Outlook) | Procesar adjuntos → Drive → Airparser |
| `TPau_Recibos_Procesar` | Webhook Airparser | Validar datos → INSERT en recibos |
| `TPau_Recibos_Revision` | Watch Changes (Sheets) | Marcar revisado=true automáticamente |

---

## Campos Extraídos (OCR)

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `razon_social` | TEXT | Nombre del comercio/empresa |
| `fecha` | DATE | Fecha transacción (YYYY-MM-DD) |
| `matricula` | TEXT | Matrícula normalizada |
| `importe` | DECIMAL | Importe total EUR |
| `categoria` | ENUM | gasoil, peajes, limpieza_vehiculos, otros |
| `pago` | ENUM | empresa, transportista |

---

## Categorías y Detección

| Categoría | Keywords detectados |
|-----------|---------------------|
| `gasoil` | gasoil, diesel, combustible, fuel, carburant, litros, surtidor |
| `peajes` | peatge, peaje, autopista, toll, via-t, telepeaje, tunel |
| `limpieza_vehiculos` | rentat, lavado, neteja, limpieza, autolavado, car wash |
| `otros` | *(fallback)* |

## Tipo de Pago

| Valor | Indicadores |
|-------|-------------|
| `empresa` | SOLRED, DKV, tarjeta empresa, "CLIENT: TRANSPORTS" |
| `transportista` | efectivo, contado, metalico, efectiu |
| *(vacío)* | Sin indicadores claros → requiere revisión |

---

## Estados del Sistema

### Estado de Entrada (`recibos_entrada.estado`)

| Estado | Descripción |
|--------|-------------|
| `recibido` | Fichero recibido, pendiente |
| `subiendo` | Subiendo a Google Drive |
| `enviado_airparser` | Esperando OCR |
| `completado` | Procesado OK |
| `error` | Error (ver error_detalle) |
| `descartado` | No es recibo/duplicado |

### Estado de Recibo (`recibos.estado`)

| Estado | Descripción |
|--------|-------------|
| `PROCESADO` | Recibo procesado correctamente |
| `REVISION` | Pendiente de revisión manual |
| `ERROR` | Error en OCR |
| `DUPLICADO` | Recibo duplicado |

---

## Validaciones Automáticas

### Campos Obligatorios (requiere_revision = TRUE si falla)

```
fecha NO vacío AND
importe > 0 AND
razon_social NO vacío AND
pago NO vacío
```

### Lógica de Revisión

```
SI usuario edita (razon_social, fecha, matricula, importe, categoria, pago):
  → revisado = TRUE
  → revisado_por = email usuario
  → revisado_at = timestamp
```

---

## Documentación del Módulo

| Documento | Contenido |
|-----------|-----------|
| `especificaciones-digitalizar-recibos.md` | Especificaciones de entrada |
| `airparser-campos.md` | Campos configurados en Airparser |
| `esquemas-tabla.md` | Esquemas completos de tablas |
| `flujo.md` | Flujo completo con diagramas |
| `instalacion.md` | Guía de instalación |

## Skills

| Skill | Uso |
|-------|-----|
| `entradas-del-recibo` | Esquema de campos a extraer de recibos |
| `airparser-api` | Guía de integración con Airparser |
| `nanonets-api` | Guía alternativa con Nanonets |

---

## Estructura Drive

```
TransportsPau_Recibos/
├── 1_Entrada/           # Backups sin procesar (por fecha)
├── 2_Procesados/        # Recibos OK (por año/mes/categoria)
├── 3_Revision/          # Pendientes revisión manual
├── 4_Errores/           # Recibos con error
└── _Config/             # Documentación
```

## Nomenclatura Archivos

```
[YYYYMMDD]_[CATEGORIA]_[RAZON_SOCIAL]_[IMPORTE]_[MATRICULA]_[ID].[ext]
Ejemplo: 20241215_gasoil_REPSOL_125.50_1234ABC_TK00001.jpg
```

---

## Herramientas

- **Airparser:** OCR principal + LLM
- **Make.com:** Automatización de flujos (3 escenarios)
- **Google Sheets:** Tablas recibos_entrada y recibos
- **Google Drive:** Almacenamiento de ficheros
