# Campos Airparser - Copiar y Pegar

Referencia rápida para configurar el inbox de prueba en Airparser.

---

## Campo 1: razon_social

**Name:**

```text
razon_social
```

**Type:** String

**Description:**

```text
Nombre del comercio o empresa que emite el recibo. Buscar en la cabecera del documento.

Ejemplos:
- REPSOL ESTACIÓN MARTORELL
- CEPSA LA JONQUERA
- ESTACIÓ DE SERVEI PUIGCERDÀ
- AUTOPISTAS AUMAR S.A.
- E.S. BESCANO
- RALLY ESTACION SERVICIO, S.A.

Reglas:
- Recortar espacios extra al inicio y final
- Colapsar múltiples espacios en uno solo
- Preservar acentos catalanes y españoles (ç, ñ, à, é, ò)
- Usar mayúsculas tal como aparece en el recibo
- Preferir el nombre “identificable” del proveedor (marca/estación) frente a direcciones:
  - Bien: "E.S. BESCANO", "RALLY ESTACION SERVICIO, S.A."
  - Evitar: "CTRA. C-260 PK.28", "17600 - FIGUERES", "CAN CULEBRA S/N ..."
- Si aparece una marca grande (REPSOL/CEPSA/BP/...) y debajo la razón social de la estación, se puede combinar:
  - Ejemplo: "REPSOL - RALLY ESTACION SERVICIO, S.A."
- No confundir con datos del cliente/empresa:
  - Ignorar líneas tipo (ES/CAT): "CLIENTE:", "CLIENT:", "EMPRESA:", "Empresa:", "NIF CLIENTE", "NIF CLIENT", "Dades:", "Datos:"
  - Ejemplo: si aparece "CLIENT: TRANSPORTS ...", eso NO es el proveedor
- Si no se puede determinar el proveedor con alta confianza (solo hay dirección/códigos), dejar el campo vacío
```

**Default Value:** *(dejar vacío)*

---

## Campo 2: fecha

**Name:**

```text
fecha
```

**Type:** String

**Description:**

```text
Fecha de la transacción. Convertir al formato YYYY-MM-DD.

Formatos de entrada aceptados:
- DD/MM/YYYY (15/03/2024)
- DD-MM-YYYY (15-03-2024)
- DD.MM.YYYY (15.03.2024)
- DD/MM/YY (17/12/25)
- DD-MM-YY (17-12-25)
- DD.MM.YY (17.12.25)

Ejemplos de conversión:
- 15/03/2024 → 2024-03-15
- 07-12-2024 → 2024-12-07
- 01.01.2025 → 2025-01-01
- 17/12/25 → 2025-12-17

Ambigüedades:
- Si hay múltiples fechas, usar la fecha de transacción (cerca del TOTAL), no la fecha de impresión
- Si hay fecha y hora, extraer solo la fecha
- Formato americano MM/DD/YYYY es poco probable en España, asumir DD/MM/YYYY
- Si el año viene con 2 dígitos (YY), asumir 20YY (ej: 25 → 2025)
- Etiquetas frecuentes (ES/CAT) a buscar: "Data:", "Fecha:", "Hora:", "Data i hora:", "Fecha y hora:"
- Si hay dudas (fecha parcial, números borrosos, OCR confuso), dejar vacío
```

**Default Value:** *(dejar vacío)*

---

## Campo 3: matricula

**Name:**

```text
matricula
```

**Type:** String

**Description:**

```text
Matrícula del vehículo si aparece en el recibo. Normalizar formato.

Formatos españoles válidos:
- Nuevo: 1234 ABC, 1234ABC, 1234-ABC
- Antiguo: M-1234-AB, B-5678-CD

Ejemplos:
- 5678 XYZ → 5678XYZ
- B-1234-CD → B1234CD
- 9012 ABC → 9012ABC

Reglas:
- Eliminar espacios y guiones
- Convertir a mayúsculas
- Mantener ceros iniciales si aparecen (ej: 0916LDH)
- Si no aparece matrícula, dejar vacío
- Si la matrícula es ilegible o parcial, dejar vacío
- Flota de transporte (camiones): puede haber 2 matrículas:
  - Tractora/camión (vehículo) y remolque/semirremolque (plataforma)
  - Si aparecen varias, elegir la que tenga contexto de vehículo principal:
    - Preferir cerca de: "MATRÍCULA/MATRICULA", "VEHÍCULO/VEHICLE", "TRACTORA", "CAMIÓN/CAMIÓ"
    - Evitar si es claramente remolque: "REMOLQUE/REMOLC", "SEMIRREMOLQUE/SEMIREMOLC", "TRAILER"
  - Si no se puede decidir con alta confianza cuál es la principal, dejar vacío (mejor revisión manual)

Ambigüedades:
- No confundir con números de tarjeta, NIF, o códigos de producto
- La matrícula suele aparecer cerca de "VEHÍCULO", "VEHICLE", "MATRÍCULA", "MATRICULA", "Matricula", o en el encabezado
- OCR típico: O/0, I/1, S/5 pueden confundirse. Si hay duda en 1-2 caracteres, dejar vacío
- Ignorar números de operación/terminal/surtidor:
  - "Nº operación", "Concessió", "N. Autor", "Terminal/Term.", "Torn", "Surtidor/Sortidor", "Ident."
- Etiquetas frecuentes (ES/CAT): "Matrícula:", "Matricula:", "Vehicle:", "Vehículo:", "Tractora:", "Remolc:", "Remolque:"
```

**Default Value:**

```text
""
```

---

## Campo 4: importe

**Name:**

```text
importe
```

**Type:** Decimal

**Description:**

```text
Importe total a pagar en EUR. Número decimal con punto.

Buscar etiquetas:
- TOTAL
- TOTAL A PAGAR
- IMPORTE TOTAL
- TOTAL EUR
- A PAGAR
- TOTAL VENDA / TOTAL VENTA
- TOTAL TARGETA / TOTAL TARJETA
- TOTAL SUBMINISTRAT / TOTAL SUMINISTRADO
- IMPORT (en recibos de gasoil suele ser el total cobrado)
- IMPORT TOTAL / IMPORTE TOTAL
- TOTAL A COBRAR / TOTAL COBRAT
- TOTAL PAGADO / TOTAL PAGAT

Ejemplos:
- 125,50 € → 125.50
- 89.00 EUR → 89.00
- € 245,80 → 245.80

Reglas:
- Convertir coma decimal a punto (125,50 → 125.50)
- Eliminar separadores de miles si aparecen (1.234,56 → 1234.56; 1 234,56 → 1234.56)
- Eliminar símbolo de moneda (€, EUR)
- Usar 2 decimales
- Validación: el importe debe ser > 0; si sale 0, negativo o sin sentido, dejar vacío

Ambigüedades:
- Si hay múltiples importes (base, IVA, subtotal, total), usar el TOTAL FINAL cobrado.
- En recibos de gasoil: usar el total cobrado por el repostaje (no el precio por litro).
- IMPORTANTE (preautorizaciones/reservas): ignorar importes tipo:
  - "TOTAL RESERVAT / TOTAL RESERVADO"
  - "PREAUTORIZACIÓN", "RETENCIÓN", "DEPÓSITO", "FIANZA"
  y priorizar importes tipo:
  - "TOTAL SUBMINISTRAT / TOTAL SUMINISTRADO"
  - "TOTAL VENDA / TOTAL VENTA"
  - "TOTAL TARGETA / TOTAL TARJETA"
  - "IMPORT TOTAL / IMPORTE TOTAL"
- Si hay 2 totales “finales” iguales (duplicados), cualquiera sirve (son el mismo cobro)
- Si solo se ve "IMPORT" sin contexto, comprobar coherencia cuando aplique:
  - Gasoil: \(litros \times precio\) ≈ importe (tolerancia por redondeos)
  - Si no cuadra o hay varias opciones plausibles, dejar vacío
- En peajes/accesos: puede haber base+IVA+total; usar el total (si es legible). Si es “document no deduible” igualmente puede haber total: extraerlo si es claro
- Si no se puede identificar el importe final cobrado con alta confianza, dejar vacío
```

**Default Value:** *(dejar vacío)*

---

## Campo 5: categoria

**Name:**

```text
categoria
```

**Type:** Enum

**Values:**

```text
gasoil
peajes
limpieza_vehiculos
otros
```

**Description:**

```text
Categoría del gasto según el tipo de recibo.

GASOIL - Combustible:
Palabras clave: gasoil, gasóleo, diésel, diesel, combustible, fuel, carburante, repostaje, litros, surtidor
Ejemplos: Repsol, Cepsa, BP, Shell, Galp, estación de servicio

PEAJES - Peajes:
Palabras clave: peaje, peatge, autopista, autovía, toll, via-t, telepeaje, abertis, aumar, acesa, túnel, tunel
También (accesos/entradas con tarifa): comprovant de pas, comprobante de paso, tarifa, porta, entrada
Ejemplos: AP-7, C-32, túneles, autopistas de peaje

LIMPIEZA_VEHICULOS - Lavado/Limpieza:
Palabras clave: lavado, rentat, limpieza, neteja, autolavado, car wash, túnel de lavado, aspirador
Ejemplos: Elefante Azul, IMO, lavado a mano

OTROS - Otros (por defecto):
Todo lo que no encaje en las categorías anteriores
Ejemplos: parking, tienda, restaurante, reparaciones

Reglas:
- Clasificar por palabras clave del recibo
- Si hay duda entre categorías, preferir la más específica
- OTROS es el fallback cuando no hay coincidencia clara
- Si el recibo mezcla conceptos (ej: tienda + combustible), priorizar por lo que domina el recibo:
  - Si hay litros/precio por litro/producto combustible → gasoil
  - Si hay "lavado/túnel/aspirador" y NO hay litros → limpieza_vehiculos
  - Si hay "peaje/via-t/comprovant de pas/tarifa/entrada" → peajes
- Si no hay coincidencia clara, dejar el default (otros)
```

**Default Value:**

```text
otros
```

---

## Campo 6: pago

**Name:**

```text
pago
```

**Type:** Enum

**Values:**

```text
empresa
transportista
```

**Description:**

```text
Método de pago: empresa o conductor.

EMPRESA - Pagado por la empresa:
Indicadores:
- Facturado a "TRANSPORTS PAU" o similar
- Tarjeta de empresa (Solred, Repsol Más, Cepsa Star, etc.)
- "SOLRED" / "DKV" (muy buen indicador de tarjeta de flota)
- "VENDA AMB TARGETA" / "PAGO CON TARJETA" junto con datos de flota (SOLRED/DKV o cliente empresa)
- NIF de la empresa en el recibo
- "CLIENTE: TRANSPORTS PAU"
- Tarjeta de débito/crédito de flota

TRANSPORTISTA - Pagado por el conductor:
Indicadores:
- Pago en efectivo
- "EFECTIVO", "CONTADO", "METÁLICO" (ES)
- "EFECTIU", "AL COMPTAT", "METÀL·LIC" (CAT)
- Tarjeta personal del conductor
- Sin datos fiscales de empresa
- "CONTADO", "EFECTIVO"

Ambigüedades - DEJAR VACÍO si:
- No hay indicación clara del método de pago
- El recibo no muestra si es tarjeta empresa o personal
- Hay datos contradictorios
- Si solo aparece "CRÈDIT/CRÉDITO" sin más contexto de empresa/flota, dejar vacío
- Si solo aparece “pago con tarjeta” sin SOLRED/DKV ni “cliente TRANSPORTS …”, dejar vacío (podría ser tarjeta personal)

Reglas prácticas (prudencia):
- Marcar como `empresa` solo cuando haya una señal fuerte (SOLRED/DKV, "CLIENTE: TRANSPORTS...", o similar)
- Marcar como `transportista` solo cuando haya señal fuerte (EFECTIVO/CONTADO/METÁLICO)
- En caso contrario, dejar vacío

Es preferible dejar vacío para revisión manual que adivinar incorrectamente.
```

**Default Value:** *(dejar vacío - requiere revisión si ambiguo)*

---

## Casuísticas y confusiones frecuentes (guía rápida)

- **Proveedor vs cliente**: en gasolineras puede aparecer "CLIENTE/EMPRESA" con TRANSPORTS. Eso NO es `razon_social`.
- **Preautorización / reserva** (gasoil): si existe "TOTAL RESERVAT/RESERVADO", NO es el gasto real. Usar "TOTAL SUBMINISTRAT/SUMINISTRADO" o equivalente.
- **Separadores numéricos**: algunos recibos usan miles con punto o espacio. Siempre normalizar a decimal con punto.
- **Matrícula**: si hay duda en caracteres por OCR, mejor vacío.
- **“Document no deduible”**: no impide extraer fecha/importe; es una señal para revisión administrativa (fuera de estos 6 campos).

---

## Resumen de Configuración

| Campo | Nombre Airparser | Tipo | Default | Notas |
| ------- | ---------------- | ------ | ------- | ------- |
| razon_social | `razon_social` | String | - | Cabecera del recibo |
| fecha | `fecha` | String | - | Formato YYYY-MM-DD |
| matricula | `matricula` | String | "" | Vacío si no aparece |
| importe | `importe` | Decimal | - | Total final a pagar |
| categoria | `categoria` | Enum | otros | 4 valores posibles |
| pago | `pago` | Enum | - | Sin default, revisar si ambiguo |

---

## Checklist de Configuración

- [ ] Crear inbox de prueba
- [ ] Añadir 6 campos con nombres exactos (snake_case)
- [ ] Copiar descripciones completas con ejemplos
- [ ] Configurar tipos (String, Decimal, Enum)
- [ ] Configurar valores de Enum
- [ ] Configurar defaults donde aplique
- [ ] Subir recibo de prueba
- [ ] Verificar extracción
- [ ] Exportar schema a JSON para backup

---

## Especificación de Extracción

### Propósito

Convertir imágenes/PDFs de recibos en **registros estructurados y validados** adecuados para automatización (Make/n8n, hojas de cálculo, contabilidad). La precisión y el manejo explícito de incertidumbre son obligatorios.

### Entradas

- Imágenes: JPG / PNG / HEIC
- Documentos: PDFs escaneados
- Idiomas: Catalán, Español (a menudo mezclados)

### Esquema de Salida (Canónico)

```text
razon_social      string
fecha             date (YYYY-MM-DD)
matricula         string
categoria         enum(gasoil|peajes|limpieza_vehiculos|otros)
importe           number (EUR, decimal with dot)
pago              enum(empresa|transportista)
```

Mantener texto OCR crudo y confianza por campo para auditoría/depuración.

### Modelo de Calidad

Cada registro debe incluir:
- Confianza por campo (0–1)
- needs_review boolean
- review_reasons (ej., date_ambiguous, total_not_found, plate_low_confidence)

Preferir en blanco + revisión sobre valores alucinados.

### Pipeline de Procesamiento (Recomendado)

1. Pre-procesar: rotar/enderezar, recortar, mejorar contraste.
2. OCR: capaz de CA/ES; retener texto crudo (y bloques si están disponibles).
3. Parsear: patrones para fecha, importes, matrícula; comercio del encabezado.
4. Clasificar: reglas deterministas primero; asistencia opcional del modelo.
5. Normalizar y validar: aplicar reglas de campos; calcular confianza/banderas de revisión.

### Pruebas

- Pruebas basadas en fixtures con recibos reales anonimizados.
- Cobertura: cada categoría, CA-intensivo + ES-intensivo, total ambiguo, matrícula faltante.
- Salida esperada + banderas de revisión requeridas.

### Protección de Datos

- Los recibos son sensibles.
- Limitar la retención de imágenes crudas.
- Aplicar control de acceso y políticas de retención al texto OCR.

### Criterios de Aceptación

- Produce el esquema canónico.
- Clasificación y normalización deterministas.
- Manejo explícito de incertidumbre.
