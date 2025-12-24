# Automatizaciones Transports Pau

Sistema de automatización para el procesamiento y extracción de datos de recibos de la flota de Transports Pau.

## Estructura del repositorio

```
├── recibos/                           # Sistema OCR de recibos
│   ├── flujo.md                       # Diagrama completo del flujo Make + Airparser
│   ├── esquemas-tabla.md              # Estructura de tablas (recibos_entrada, recibos)
│   ├── airparser-campos.md            # Campos configurados en Airparser
│   ├── instalacion.md                 # Guía de instalación (Google APIs, Apps Script)
│   ├── especificaciones-digitalizar-recibos.md  # Requisitos del sistema
│   └── AGENTS.md                      # Guía para agentes AI
├── docs/
│   └── recursos.md                    # Enlaces a herramientas (Make, Airparser)
├── .claude/skills/                    # Skills para Claude Code
└── AGENTS.md                          # Guía general para agentes AI
```

---

## Sistema: Digitalización de recibos

### Resumen del flujo

```
Conductor               Make.com                    Airparser
─────────               ────────                    ─────────
   │                        │                           │
   │  Envía email           │                           │
   │  con foto/PDF          │                           │
   │──────────────────────▶│                           │
   │                        │  Upload adjunto           │
   │                        │─────────────────────────▶│
   │                        │                           │  OCR + LLM
   │                        │                           │  extrae datos
   │                        │         Webhook           │
   │                        │◀─────────────────────────│
   │                        │                           │
   │                        │  Guarda en Google Sheets  │
   │                        │  (recibos_entrada + recibos)
```

### Para conductores

1. Hacer foto al recibo (gasoil, peaje, lavado, etc.)
2. Enviar por email a la dirección configurada
3. Listo. El sistema procesa automáticamente.

### Para administración

Los datos se guardan en Google Sheets con:
- Razón social, fecha, matrícula, importe
- Categoría (gasoil, peajes, limpieza_vehiculos, otros)
- Método de pago (empresa o transportista)
- Flags de revisión automática si faltan datos

Ver documentación completa en [`recibos/flujo.md`](recibos/flujo.md).

---

## Instalación

Ver [`recibos/instalacion.md`](recibos/instalacion.md) para:
- Configurar Google APIs en Make.com
- Instalar el Apps Script de auto-revisión

---

## Tecnologías

| Herramienta | Uso |
|-------------|-----|
| [Airparser](https://airparser.com) | OCR + extracción con LLM |
| [Make.com](https://make.com) | Automatización de flujos |
| Google Sheets | Base de datos de recibos |
| Google Drive | Backup de imágenes originales |
