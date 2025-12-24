# Guía de Instalación

## Conectar servicios Google a Make.com

Sigue la guía oficial de Make.com:

- [Connect to Google services using a custom OAuth client](https://help.make.com/connect-to-google-services-using-a-custom-oauth-client)

### APIs a activar en Google Cloud Console

En **APIs & Services > Library**, habilita:

1. **Gmail API** - [Documentación](https://apps.make.com/google-email)
2. **Google Drive API** - [Documentación](https://apps.make.com/google-drive)
3. **Google Sheets API** - [Documentación](https://apps.make.com/google-sheets)

### Scopes requeridos

En **OAuth consent screen > Data Access**, añade:

```
https://www.googleapis.com/auth/gmail.readonly
https://www.googleapis.com/auth/gmail.send
https://www.googleapis.com/auth/drive
https://www.googleapis.com/auth/spreadsheets
https://www.googleapis.com/auth/userinfo.email
```

| Scope | Uso en el flujo |
|-------|-----------------|
| `gmail.readonly` | Leer emails con adjuntos de recibos |
| `gmail.send` | Enviar notificaciones de error |
| `drive` | Crear carpetas y subir imágenes |
| `spreadsheets` | Escribir/leer hoja Gastos_Flota |
| `userinfo.email` | Identificar usuario conectado |

### Redirect URI

En **Credentials > OAuth 2.0 Client**, añade los tres:

```
https://www.make.com/oauth/cb/google-email
https://www.integromat.com/oauth/cb/google-restricted
https://www.integromat.com/oauth/cb/google/
```

| Redirect URI | Servicio |
|--------------|----------|
| `.../google-email` | Gmail |
| `.../google-restricted` | Google Drive |
| `.../google/` | Google Sheets |

### Publicar la app (importante)

En **OAuth consent screen > Audience**, haz clic en **Publish app**.

| Estado | Reautorización |
|--------|----------------|
| Testing | Cada 7 días |
| In production | Cada 6 meses (@gmail) |

> **Nota**: Si aparece "Needs verification", puedes conectar igualmente tu app no verificada. Make.com permite conexiones a apps no verificadas, aunque Google podría restringirlo en el futuro.

### Recursos

- [Verificación de apps OAuth de Google](https://support.google.com/cloud/answer/13463073?hl=es-419)
- [Configurar pantalla de consentimiento OAuth](https://support.google.com/cloud/answer/10311615)

---

## Configurar Apps Script de Auto-Revisión

Este script marca automáticamente `revisado=TRUE` cuando un usuario edita campos de datos en la hoja `recibos`.

### 1. Abrir el editor de Apps Script

1. Abre tu Google Spreadsheet con las hojas `recibos_entrada` y `recibos`
2. Ve a **Extensiones > Apps Script**
3. Se abrirá el editor en una nueva pestaña

### 2. Pegar el código

1. Borra el contenido por defecto (`function myFunction() {}`)
2. Copia y pega el código de la sección "Apps Script: Auto-Revision en Google Sheets" de `esquemas-tabla.md`
3. Guarda el proyecto: **Ctrl+S** o icono de disquete
4. Ponle un nombre al proyecto (ej: "AutoRevision_Recibos")

### 3. Configurar el trigger automático

El trigger `onEdit` debe instalarse manualmente para que funcione con todos los usuarios:

1. En el editor de Apps Script, haz clic en el icono de **reloj** (Activadores / Triggers) en el menú lateral izquierdo
2. Haz clic en **+ Añadir activador** (esquina inferior derecha)
3. Configura:

| Campo | Valor |
|-------|-------|
| Función | `onEdit` |
| Implementación | `Principal` |
| Origen del evento | `Desde hoja de cálculo` |
| Tipo de evento | `Al editar` |
| Configuración de notificaciones de error | `Notificarme inmediatamente` (opcional) |

4. Haz clic en **Guardar**
5. Google pedirá permisos. Haz clic en **Revisar permisos** > selecciona tu cuenta > **Avanzado** > **Ir a [nombre del proyecto]** > **Permitir**

### 4. Ajustar índices de columnas

El script asume un orden específico de columnas. Verifica que coincida con tu hoja `recibos`:

```javascript
// Columnas de datos (ajustar según orden real en tu hoja)
// A=id, B=id_entrada, C=razon_social, D=fecha, E=matricula, F=importe, G=categoria, H=pago
var dataColumns = [3, 4, 5, 6, 7, 8]; // C a H

// Columnas de control (ajustar según tu hoja)
var COL_REVISADO = 11;      // K
var COL_REVISADO_POR = 12;  // L
var COL_REVISADO_AT = 13;   // M
```

Si tus columnas están en otro orden, ajusta los números. La columna A = 1, B = 2, etc.

### 5. Probar el funcionamiento

1. Abre la hoja `recibos`
2. Edita cualquier celda en las columnas de datos (C a H) de una fila existente
3. Verifica que automáticamente:
   - `revisado` (columna K) cambia a `TRUE`
   - `revisado_por` (columna L) muestra tu email
   - `revisado_at` (columna M) muestra la fecha/hora actual

### 6. Función de validación (opcional)

La función `validateRows()` marca `requiere_revision=TRUE` en filas con datos incompletos. Para ejecutarla:

**Manualmente:**
1. En Apps Script, selecciona `validateRows` en el desplegable de funciones
2. Haz clic en **Ejecutar**

**Automáticamente (con trigger):**
1. Añade otro activador con:
   - Función: `validateRows`
   - Tipo de evento: `Al cambiar` (onChange)

### Troubleshooting

| Problema | Solución |
|----------|----------|
| El script no se ejecuta | Verifica que el trigger esté activo en la pestaña "Activadores" |
| Error de permisos | Vuelve a autorizar: Ejecutar > Revisar permisos |
| `revisado_por` muestra vacío | Normal si el usuario no ha dado permisos de email al script |
| Solo funciona para ti | Asegúrate de usar trigger instalable (no simple) siguiendo el paso 3 |

### Limitaciones

- El trigger `onEdit` no se activa con cambios hechos por scripts o APIs (solo ediciones manuales)
- Para detectar cambios de Make.com, usa el escenario `TPau_Recibos_Revision` con Watch Changes
