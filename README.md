# Prueba tecnica - Modelo de fraude en seguros

Entrega compacta de la prueba tecnica de cientifico de datos para construir un modelo de priorizacion de fraude en seguros usando Python y Databricks Free Edition.

La solucion no busca rechazar reclamaciones automaticamente. Su objetivo es generar un `score_fraude` para ordenar los casos de mayor riesgo y apoyar a equipos de investigacion, auditoria y siniestros.

## Estructura del repositorio

```text
.
|-- README.md
|-- data/
|   `-- Muestra_Base_fraude.xlsx
|-- docs/
|   `-- Prueba_tecnica_Cientifico_Datos.pdf
|-- databricks/
|   |-- 01_modelo_fraude_pyspark.py
|   `-- 01_modelo_fraude_pyspark.html
|-- outputs/
|   |-- databricks_model_metrics_real.csv
|   |-- databricks_decile_lift_real.csv
|   `-- databricks_scored_top100_real.csv
`-- deliverables/
    `-- presentacion_sustentacion_fraude_seguros.pptx
```

## Contenido

- `data/Muestra_Base_fraude.xlsx`: base original entregada para la prueba.
- `docs/Prueba_tecnica_Cientifico_Datos.pdf`: enunciado de la prueba tecnica.
- `databricks/01_modelo_fraude_pyspark.py`: notebook/script exportado desde Databricks con preparacion, entrenamiento, evaluacion, scoring y guardado en Delta.
- `databricks/01_modelo_fraude_pyspark.html`: version HTML del notebook para revision rapida sin abrir Databricks.
- `outputs/databricks_model_metrics_real.csv`: metricas reales exportadas desde Databricks.
- `outputs/databricks_decile_lift_real.csv`: lift por deciles exportado desde Databricks.
- `outputs/databricks_scored_top100_real.csv`: top 100 de casos puntuados por mayor `score_fraude`.
- `deliverables/presentacion_sustentacion_fraude_seguros.pptx`: presentacion final con resultados, explicabilidad, scoring, arquitectura conceptual y conclusiones de negocio.

## Objetivo de la solucion

Construir un modelo supervisado para estimar la probabilidad de fraude en reclamaciones de seguros y convertir esa probabilidad en una bandeja priorizada de investigacion.

Salida esperada del scoring:

- `score_fraude`: probabilidad o puntaje relativo de riesgo.
- `decil`: posicion del caso dentro del ranking de riesgo.
- `prioridad`: nivel sugerido de revision.
- `fecha_scoring`: fecha de ejecucion del scoring.
- `version_modelo`: modelo usado para puntuar.

## Flujo tecnico

El notebook de Databricks realiza:

1. Lectura de la tabla fuente.
2. Limpieza de nombres de columnas.
3. Creacion de la variable objetivo binaria `label`.
4. Revision de calidad de datos.
5. Ingenieria de variables temporales.
6. Exclusion de variables con riesgo de fuga de informacion.
7. Split temporal entrenamiento/prueba.
8. Entrenamiento de `logistic_regression`, `random_forest` y `gradient_boosting`.
9. Evaluacion con ROC-AUC y area under PR.
10. Calculo de lift por deciles.
11. Generacion de casos puntuados.
12. Guardado de resultados en tablas Delta.

## Resultado principal

El mejor modelo ejecutado en Databricks fue `random_forest`.

| Modelo | ROC-AUC | Area under PR |
| --- | ---: | ---: |
| Random forest | 0.785 | 0.613 |
| Gradient boosting | 0.706 | 0.523 |
| Logistic regression | 0.294 | 0.260 |

El ranking por deciles evidencia concentracion de fraude en los primeros grupos:

| Decil | Casos | Fraudes | Tasa fraude | Lift |
| ---: | ---: | ---: | ---: | ---: |
| 1 | 320 | 218 | 68.1% | 2.05 |
| 2 | 320 | 226 | 70.6% | 2.12 |
| 3 | 320 | 175 | 54.7% | 1.64 |
| 10 | 319 | 15 | 4.7% | 0.14 |

Interpretacion: el modelo sirve para priorizar investigaciones, porque los primeros deciles concentran una tasa de fraude mayor que los ultimos.

## Consultas SQL utiles en Databricks

### Ranking de modelos

```sql
SELECT *
FROM workspace.default.fraude_model_metrics
ORDER BY area_under_pr DESC;
```

### Lift por deciles

```sql
SELECT *
FROM workspace.default.fraude_decile_lift
ORDER BY decil;
```

### Captura acumulada por decil

```sql
WITH base AS (
  SELECT
    decil,
    casos,
    fraudes,
    tasa_fraude,
    lift,
    SUM(fraudes) OVER (ORDER BY decil) AS fraudes_acumulados,
    SUM(fraudes) OVER () AS fraudes_totales
  FROM workspace.default.fraude_decile_lift
)
SELECT
  decil,
  casos,
  fraudes,
  tasa_fraude,
  lift,
  fraudes_acumulados,
  ROUND(fraudes_acumulados / fraudes_totales, 4) AS captura_acumulada
FROM base
ORDER BY decil;
```

### Top 100 casos con mayor score

```sql
SELECT *
FROM workspace.default.fraude_scored_test
ORDER BY score_fraude DESC
LIMIT 100;
```

### Segmentos de mayor riesgo

```sql
SELECT
  Ramo_Desc,
  Nombre_Canal_Comercial,
  REGIONAL,
  Cobertura,
  COUNT(*) AS casos,
  AVG(label) AS tasa_fraude,
  AVG(score_fraude) AS score_promedio
FROM workspace.default.fraude_scored_test
GROUP BY
  Ramo_Desc,
  Nombre_Canal_Comercial,
  REGIONAL,
  Cobertura
HAVING COUNT(*) >= 20
ORDER BY tasa_fraude DESC, casos DESC
LIMIT 30;
```

### Vista de bandeja priorizada

```sql
CREATE OR REPLACE VIEW workspace.default.vw_fraude_bandeja_priorizada AS
SELECT
  current_date() AS fecha_scoring,
  'random_forest' AS version_modelo,
  CASE
    WHEN decil <= 2 THEN 'Alta'
    WHEN decil <= 5 THEN 'Media'
    ELSE 'Baja'
  END AS prioridad,
  score_fraude,
  decil,
  label,
  Ramo_Desc,
  Nombre_Canal_Comercial,
  REGIONAL,
  SUCURSAL,
  Cobertura,
  Tipo_apertura
FROM workspace.default.fraude_scored_test;
```

## Decisiones tecnicas clave

- Split temporal para aproximar comportamiento futuro.
- Exclusion de variables con riesgo de leakage, como estado final, cierre, pagos o informacion posterior al momento de scoring.
- Creacion de variables temporales para capturar demoras y antiguedades operativas.
- Evaluacion con metricas adecuadas para fraude: ROC-AUC, area under PR y lift por deciles.
- Interpretacion del score como herramienta de priorizacion, no como prueba definitiva de fraude.

## Lectura de negocio

La solucion permite focalizar la capacidad investigativa en los casos con mayor probabilidad de fraude. En lugar de revisar todos los casos por igual, el negocio puede iniciar por los deciles 1 y 2, donde se concentra mayor tasa de fraude observada.

Uso recomendado: bandeja de trabajo para investigadores o auditoria, con trazabilidad de score, decil, fecha de scoring y version del modelo.

## Limitaciones

- La etiqueta historica puede reflejar sesgos de investigacion previa.
- El modelo requiere validacion con datos posteriores antes de usarse en produccion.
- El score no reemplaza evidencia documental ni criterio experto.
- Deben monitorearse drift, lift por deciles, distribucion de score y desempeno con etiquetas confirmadas.
