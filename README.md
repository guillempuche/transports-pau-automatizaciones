# Automatizaciones Transports Pau

Repositorio de flujos de automatización para la empresa. Actualmente incluye el sistema de digitalización de recibos de conductores.

## Estructura del repositorio

```
├── recibos/                # Configuración y especificaciones de OCR
│   ├── airparser-campos.md # Campos configurados en Airparser
│   └── especificaciones-digitalizar-recibos.md
├── docs/                   # Documentación general
│   └── recursos.md         # Enlaces a herramientas
├── AGENTS.md               # Guía para agentes AI
└── TODO.md                 # Tareas pendientes
```

---

## Flujo: Digitalización de recibos

### 1. Configuración inicial (oficina, una sola vez)

- Creas cuenta en Aiparser.
- Creas un "mailbox" de recibos, por ejemplo: `recibos-flota@aiparser.io`.
- En Make, creas un escenario:
  - Disparador: "New parsed document" de Aiparser.
  - Acción 1: guardar los campos (fecha, importe, proveedor, IVA, líneas) en una hoja de cálculo o base de datos.
  - Acción 2 (opcional): enviar notificación a email interno si falta algún dato o si el importe supera X.

### 2. Uso diario por los conductores

- Cada conductor tiene en el móvil un contacto llamado "Enviar recibo gasoil" con el email `recibos-flota@aiparser.io`.
- Después de repostar o recibir un recibo:
  - Abre la app de cámara o de correo (en español/catalán).
  - Hace la foto al recibo.
  - Pulsa "Compartir > Correo" y lo envía al contacto "Enviar recibo gasoil".
- No entra nunca en Aiparser, Make ni ve nada en inglés: solo manda un email con foto.

### 3. Procesamiento automático

- Aiparser recibe el email y procesa el adjunto: extrae fecha, total, proveedor y, cuando exista, líneas de detalle (cantidad, precio, concepto).
- Make detecta que hay un nuevo documento parseado:
  - Crea una nueva fila en una hoja tipo "Gastos_Flota" con todos los campos.
  - Puedes añadir reglas sencillas: si el proveedor contiene "Cepsa" pon la categoría "Combustible", etc.

### 4. Revisión en oficina

- Administración abre la hoja de cálculo o vista en tu ERP y ve:
  - Una fila por recibo, con datos ya limpios.
  - Opcional: enlace a la imagen original guardada en Google Drive/OneDrive (la puedes subir desde Make).

### 5. Escalado sin mantenimiento técnico

- Si mañana cambias de gasolinera o de formato de recibo:
  - Los conductores siguen usando exactamente el mismo flujo (foto → email).
  - Aiparser adapta el reconocimiento automáticamente sin que tengas que redefinir zonas ni plantillas manuales.
