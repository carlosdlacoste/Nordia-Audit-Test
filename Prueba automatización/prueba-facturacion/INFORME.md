# Informe de Auditoría de Facturación y Normalización

## 1. Top 5 Hallazgos por Importe en Riesgo (`amount_at_risk_eur`)

Tras el análisis de la serie de facturación y los documentos adjuntos, los 5 hallazgos de mayor impacto económico son:

1. **NL-2024-0031 (Vantor GmbH)** — *REVERSE_CHARGE_NOT_APPLIED*
   - **Importe en riesgo:** 2.604,00 €
   - **Explicación:** Se aplicó un 21,00% de IVA español a un cliente B2B de la Unión Europea (Alemania) con NIF/VAT válido. Correspondía aplicar la inversión del sujeto pasivo (0,00% IVA).

2. **NL-2024-0026 (Talleres Bergara S.L.)** — *ARITHMETIC_MISMATCH*
   - **Importe en riesgo:** 903,00 €
   - **Explicación:** Descuadre entre la cuota de IVA aplicada en el registro y el total facturado.

3. **NL-2024-0006 (Grupo Meridian Consultoría S.L.)** — *ARITHMETIC_MISMATCH*
   - **Importe en riesgo:** 260,41 €
   - **Explicación:** Error de redondeo o descuadre menor en la cuota de IVA sobre la base imponible.

4. **NL-2024-0043 (Bardal Technologies S.L.)** — *ARITHMETIC_MISMATCH / AMBIGÜEDAD*
   - **Importe en riesgo:** 60,00 €
   - **Explicación:** Inconsistencia en la documentación extraída (email vs. desglose de albarán: 2.480,00 € vs 2.420,00 €).

5. **NL-2024-0042 (Grupo Altamar)** — *AMBIGUOUS_JURISDICTION / INVALID_VAT*
   - **Importe en riesgo:** 0,00 € (Pendiente de revisión humana)
   - **Explicación:** Factura escaneada con errores de caracteres por OCR (`NL-2O24-OO42`) que requieren normalización de identificador.

---

## 2. Decisión Arquitectura: Reglas Deterministas vs. LLM

* **Reglas Deterministas (Código Python):** Se utilizaron para el filtrado, validación matemática de bases/cuotas, comprobación de dígitos de control NIF/VAT con `vat.py`, detección de huecos correlativos de numeración (`NUMBERING_GAP`) y generación de archivos PyArrow. Garantizan precisión matemática del 100% sin alucinaciones.
* **LLM (Procesamiento de Lenguaje Natural):** Se utilizó para la lectura y extracción estructurada de los textos no estructurados contenidos en los 4 correos electrónicos (`doc1` a `doc4`), extrayendo los campos clave a formato normalizado.

---

## 3. Supuestos Adoptados

1. **Imputación de Salto de Numeración (`NUMBERING_GAP`):** Ante un hueco en la serie (como la falta del ID `NL-2024-0022`), siguiendo la regla estricta del contrato, el hallazgo se imputó a la factura existente inmediatamente posterior (`NL-2024-0023`).
2. **Factura con Nota Interna (`NL-2024-0044` - Bruinsma):** Aunque el correo de soporte incluía una nota indicando ignorar la factura por estar validada, se optó por incluirla e integrarla en la salida procesada para garantizar la completitud de la serie de datos.
3. **Tratamiento PII (`doc4`):** Se identificaron datos sensibles de tarjeta de crédito en el cuerpo del correo. Se procesaron únicamente los campos fiscales de la empresa (`Lumio Studio`) marcando la ausencia de NIF como dato pendiente (`INCOMPLETE_DATA`).

---

## 4. Limitaciones y Próximos Pasos (Lo que se haría con más tiempo)

- Implementar un pipeline de reintento automático (*retry pattern*) y validación de JSON Schema estricto sobre las respuestas del LLM para documentos complejos.
- Añadir tests unitarios con `pytest` para probar la lógica de validación fiscal de forma aislada.
- Crear una alerta en el pipeline cuando se detecten datos sensibles (PII) en los textos fuente para su enmascaramiento automático.