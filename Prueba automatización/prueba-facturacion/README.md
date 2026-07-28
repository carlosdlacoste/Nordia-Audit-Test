
# Prueba técnica — Auditoría de facturación
No hace falta que lo termines todo. Una entrega parcial bien razonada vale más que una completa a ciegas.

## El caso
Nordia Labs S.L. (Madrid) vende servicios digitales a clientes en España, la UE, Reino Unido y EEUU. Su facturación del primer semestre de 2024 tiene errores y nadie los ha revisado. Además, cuatro facturas nunca llegaron al sistema: están sueltas en
emails.

**Detecta los errores y entrega un informe priorizado.**

## Qué te damos
```
data/facturas.csv       40 facturas ya estructuradas
data/documentos/         4 emails con facturas sin estructurar
schema/ESQUEMA.md        el formato de salida
schema/esquema.py        esquemas pyarrow
provider/llm.py          acceso al LLM (modo mock gratuito)
vat.py                   validación de NIF/VAT de 8 países
```
Necesitas `pyarrow` (escribir Parquet) y, si vas a llamar a la API real, `anthropic`.
`pandas` es opcional. Con `MOCK_LLM=1` basta `pyarrow`.

## Las dos partes

**Parte A — las 40 facturas del CSV.** Detecta descuadres, identificadores fiscales inválidos, tipos de IVA que no corresponden a la operación, duplicados, huecos en la numeración, clientes con la razón social escrita de varias formas, datos ausentes.

**Parte B — los 4 emails.** Extrae los datos de factura y haz que pasen por la misma validación de la Parte A. Aquí es donde usarás el LLM.

Todo va a los **mismos dos archivos** de salida, con `source` marcando la procedencia.

## Qué entregas
```
salida/invoices.parquet
salida/findings.parquet
INFORME.md
tu código, con main.py
```
Punto de entrada: `python main.py --out salida/`, funcionando con `MOCK_LLM=1`
(sin coste ni red) y también sin esa variable.

### `INFORME.md` — una página
1. **Top 5 hallazgos por importe en riesgo**, ordenados, con el importe.
2. **Reglas vs. LLM**: qué resolviste con código determinista, qué con el modelo, y por qué. Si usaste el modelo, cuál y por qué ese.
3. **Lo que no resolviste** y qué harías con más tiempo.
4. **Supuestos** que tomaste ante cualquier ambigüedad.

## El formato de salida es un contrato
Lee `schema/ESQUEMA.md`. Se verifica automáticamente:

Las dos tablas van con **nombres de columna y valores de enum en inglés**, como un
esquema de base de datos. El CSV de entrada está en español: normalizarlo es parte
del trabajo.

- **Importes en `decimal`, no `float`.** Es dinero.
- **Fechas en `date32`, no string.**
- `invoice_id` único en `invoices`; todo `invoice_id` de `findings` debe existir allí.
- `code`, `severity`, `operation_type` y `source` solo admiten valores del enum.

Auto-verifícate antes de entregar:

```python
from schema.esquema import INVOICES_SCHEMA, validar
print(validar(mi_tabla, INVOICES_SCHEMA))   # lista vacía = correcto
```

## El LLM

```python
from provider.llm import completar
respuesta = completar("Extrae los datos de la factura de doc1: ...", schema=MI_SCHEMA)
```

Con `MOCK_LLM=1` devuelve respuestas grabadas: gratis, deterministas, sin red. Desarrolla así. **Son respuestas reales de un modelo, con sus imperfecciones.**

Sin esa variable llama a la API leyendo `ANTHROPIC_API_KEY` del entorno. **No incluyas ninguna clave en tu entrega:** no la necesitas y es descarte automático. La clave la inyectamos nosotros al ejecutar tu código.

## Reglas fiscales (España)

| Situación | IVA |
|---|---|
| Cliente en España | 21% general, 10% reducido, 4% superreducido |
| Cliente UE **con** VAT válido (B2B) | 0% — inversión del sujeto pasivo |
| Cliente UE **sin** VAT (particular, B2C) | tipo del país del cliente (régimen OSS) |
| Cliente fuera de la UE | 0% — exportación |
| Cliente en Canarias | aplica **IGIC**, no IVA |

Tipos OSS de referencia: DE 19 · FR 20 · IT 22 · PT 23 · NL 21 · IE 23.
Reino Unido salió de la UE el 01/01/2021.

Validación de identificadores fiscales: **usa `vat.py`**, que ya implementa formato y dígito de control de ES, DE, FR, IT, PT, NL, IE y GB. No hay acceso a VIES ni a internet: nada de comprobaciones online.

## Dos cosas claras

**No sobre-construyas.**
**Usa las herramientas que uses normalmente**, IA incluida. Nos interesa tu criterio.

Ante cualquier ambigüedad en estas instrucciones: decide, documenta el supuesto en el `INFORME.md` y sigue. Eso también lo medimos.