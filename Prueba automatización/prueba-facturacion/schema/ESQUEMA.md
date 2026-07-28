# Contrato de salida

Dos tablas Parquet en la carpeta que indiques con `--out`. Los nombres de tabla,
columna y valores de enum van **en inglés**; el CSV de entrada viene en español y es
tu pipeline el que normaliza. Los tipos se verifican automáticamente.

---

## `invoices.parquet` — una fila por factura

| columna | tipo | nulo | descripción |
|---|---|---|---|
| `invoice_id` | `string` | no | Identificador de la factura. **Clave primaria, única.** |
| `issue_date` | `date32` | no | Fecha de emisión. **No string.** |
| `customer_raw` | `string` | no | Razón social tal como aparece en el origen. |
| `customer_normalized` | `string` | no | Forma canónica que decidas para agrupar el mismo cliente. |
| `customer_country` | `string` | no | País del cliente, ISO 3166-1 alpha-2. |
| `customer_vat` | `string` | **sí** | Identificador fiscal. Nulo si el cliente no tiene. |
| `taxable_base` | `decimal128(12,2)` | no | Base imponible. |
| `vat_rate` | `decimal128(5,2)` | no | Tipo aplicado: `0.00`, `4.00`, `10.00`, `21.00`… |
| `vat_amount` | `decimal128(12,2)` | no | Cuota repercutida. |
| `total_amount` | `decimal128(12,2)` | no | Total de la factura. |
| `currency` | `string` | no | Moneda, ISO 4217. |
| `operation_type` | `string` | no | Régimen fiscal aplicable. Enum, abajo. |
| `source` | `string` | no | `STRUCTURED` (del CSV) o `EXTRACTED` (de un documento). |

Ejemplo:

| invoice_id | issue_date | customer_raw | customer_country | customer_vat | taxable_base | vat_rate | vat_amount | total_amount | currency | operation_type | source |
|---|---|---|---|---|---|---|---|---|---|---|---|
| NL-2024-0002 | 2024-01-07 | Aurelia Retail S.A. | ES | ESA28765436 | 7520.00 | 21.00 | 1579.20 | 9099.20 | EUR | DOMESTIC | STRUCTURED |
| NL-2024-0044 | 2024-05-06 | Bruinsma Data B.V. | NL | NL824567894B01 | 5600.00 | 0.00 | 0.00 | 5600.00 | EUR | INTRA_EU_B2B | EXTRACTED |

## `findings.parquet` — una fila por error detectado

| columna | tipo | nulo | descripción |
|---|---|---|---|
| `invoice_id` | `string` | no | **Clave foránea** → `invoices.invoice_id`. |
| `code` | `string` | no | Tipo de hallazgo. Enum, abajo. |
| `severity` | `string` | no | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`. |
| `amount_at_risk_eur` | `decimal128(12,2)` | no | Importe en riesgo. `0.00` si no aplica. |
| `field` | `string` | sí | Columna afectada. |
| `observed` | `string` | sí | Valor encontrado. |
| `expected` | `string` | sí | Valor que correspondería. |
| `confidence` | `float32` | no | 0.0 – 1.0. |
| `needs_human_review` | `bool` | no | `true` si no puedes decidirlo con los datos disponibles. |
| `explanation` | `string` | no | Una o dos frases. En español, es para el equipo de finanzas. |

Ejemplo:

| invoice_id | code | severity | amount_at_risk_eur | field | observed | expected | confidence | needs_human_review | explanation |
|---|---|---|---|---|---|---|---|---|---|
| NL-2024-0031 | REVERSE_CHARGE_NOT_APPLIED | CRITICAL | 2604.00 | vat_rate | 21.00 | 0.00 | 1.0 | false | Cliente UE con VAT válido: procedía inversión del sujeto pasivo. |
| NL-2024-0017 | AMBIGUOUS_JURISDICTION | LOW | 0.00 | customer_country | GB | | 0.5 | true | Reino Unido salió de la UE en 2021; confirmar tratamiento. |

---

## Enums

**`code`** — los 8 tipos de hallazgo:

| valor | significado |
|---|---|
| `INVALID_VAT` | Identificador fiscal con formato o dígito de control incorrecto. |
| `ARITHMETIC_MISMATCH` | `taxable_base × vat_rate / 100 ≠ vat_amount`, o la suma no da el total. |
| `NUMBERING_GAP` | Hueco en la serie correlativa de facturación. |
| `DUPLICATE_INVOICE` | La misma factura emitida dos veces. |
| `REVERSE_CHARGE_NOT_APPLIED` | Intracomunitaria B2B con IVA español en lugar de 0%. |
| `DUPLICATE_CUSTOMER` | El mismo cliente con la razón social sin normalizar. |
| `INCOMPLETE_DATA` | Falta un campo obligatorio para el régimen declarado. |
| `AMBIGUOUS_JURISDICTION` | No se puede determinar el régimen con los datos disponibles. |

**`severity`**: `CRITICAL` · `HIGH` · `MEDIUM` · `LOW`

**`operation_type`**: `DOMESTIC` · `INTRA_EU_B2B` · `OSS_B2C` · `EXPORT` · `IGIC` · `UNDETERMINED`

**`source`**: `STRUCTURED` · `EXTRACTED`

---

## Reglas que se verifican

- **Importes en `decimal`, fechas en `date32`.** `taxable_base`, `vat_rate`,
  `vat_amount`, `total_amount` y `amount_at_risk_eur` van en `decimal128` con la
  escala indicada. Nada de `float64` para dinero, nada de strings para fechas.
  `confidence` es la única columna en coma flotante, y es `float32`.
- **Integridad referencial:** todo `findings.invoice_id` tiene que existir en
  `invoices.invoice_id`. Sin huérfanos. Si detectas un hueco de numeración
  (`NUMBERING_GAP`), **imputa el hallazgo a la factura inmediatamente posterior al
  hueco** — la primera que sí existe — y pon el identificador que falta en
  `observed`. Nunca a un `invoice_id` inventado.
- **`DUPLICATE_CUSTOMER`:** si un cliente aparece con varias grafías, puedes marcar
  todas las filas del grupo o solo las que se desvían de la forma canónica. Ambas
  lecturas se aceptan.
- `invoices.invoice_id` es única: una fila por factura, incluidos los duplicados
  detectados, que se reportan como hallazgo y no fusionando filas.
- Nombres y orden de columnas exactamente como en las tablas de arriba.

## Auto-verificación

```python
from schema.esquema import INVOICES_SCHEMA, FINDINGS_SCHEMA, validar
print(validar(mi_tabla, INVOICES_SCHEMA))   # lista vacía = correcto
```
