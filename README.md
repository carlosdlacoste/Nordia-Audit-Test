# Auditoría de Facturación y Normalización - Nordia Labs

Este proyecto es una solución automatizada para el procesamiento, unificación y auditoría fiscal de datos de facturación (estructurados y no estructurados) para Nordia Labs.

## Lógica y Arquitectura

El pipeline unifica 41 facturas de origen estructurado (`data/facturas.csv`) y facturas extraídas de comunicaciones por correo electrónico (`data/documentos/`).

-   ****Extracción Inteligente (LLM):**** Se utiliza el LLM para estructurar los datos no estructurados de los emails y normalizar campos complejos.
-   ****Motor de Auditoría Fiscal (Python):**** Aplica reglas deterministas para la validación de importes, validación de NIF/VAT con `vat.py`, detección de inconsistencias tributarias (ej. inversión del sujeto pasivo no aplicada), descuadres aritméticos y faltas de correlatividad en la numeración (`NUMBERING_GAP`).
-   ****Contrato de Datos Estricto:**** Genera los archivos de salida garantizando la conformidad de tipos de datos en `PyArrow` (`decimal128`, `date32`, etc.).

## Requisitos Previos

Asegúrate de tener instalado Python 3.10+ y las siguientes dependencias:

Bash

pip install pandas pyarrow  

## Instrucciones de Ejecución

Para procesar las facturas, realizar la auditoría y generar los archivos `.parquet` de salida, ejecuta desde la raíz del proyecto:

Bash

python main.py --out salida/  

### Resultados Generados

Tras la ejecución, se creará la carpeta `salida/` con los siguientes archivos:

-   **`**salida/invoices.parquet**`**: Tabla unificada con todas las facturas procesadas y campos normalizados.
-   **`**salida/findings.parquet**`**: Registro detallado de anomalías, errores de cálculo y riesgos fiscales identificados.

## Documentación Adicional

-   **`**INFORME.md**`**: Contiene el análisis detallado del Top 5 de hallazgos por importe en riesgo, la justificación de decisiones arquitectónicas y supuestos adoptados.