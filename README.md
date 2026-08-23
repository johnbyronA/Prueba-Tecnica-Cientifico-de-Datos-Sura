# Prueba tecnica - Modelo de fraude en seguros

Este repositorio contiene la entrega compacta de la prueba tecnica de cientifico de datos para un modelo de fraude en seguros. La solucion se enfoca en priorizar casos de mayor riesgo mediante un `score_fraude`, no en tomar decisiones automaticas de rechazo.

## Archivos incluidos

```text
.
|-- Muestra_Base_fraude.xlsx
|-- Prueba_tecnica_Cientifico_Datos.pdf
|-- 01_modelo_fraude_pyspark.py
|-- 01_modelo_fraude_pyspark.html
|-- databricks_model_metrics_real.csv
|-- databricks_decile_lift_real.csv.csv
|-- databricks_scored_top100_real.csv
|-- sustentacion_requerimientos_prueba_nueva_v2.pptx
`-- README.md
```

## Descripcion de los archivos

- `Muestra_Base_fraude.xlsx`: base original de la prueba tecnica.
- `Prueba_tecnica_Cientifico_Datos.pdf`: enunciado de la prueba.
- `01_modelo_fraude_pyspark.py`: notebook/script descargado desde Databricks con el flujo de preparacion, entrenamiento, evaluacion, scoring y guardado de resultados en Delta.
- `01_modelo_fraude_pyspark.html`: version HTML del notebook/modelo para revision rapida sin abrir Databricks.
- `databricks_model_metrics_real.csv`: metricas reales exportadas desde Databricks para comparar los modelos entrenados.
- `databricks_decile_lift_real.csv.csv`: tabla real exportada desde Databricks con deciles, tasa de fraude y lift.
- `databricks_scored_top100_real.csv`: top 100 de casos puntuados por mayor `score_fraude`, exportado desde Databricks.
- `sustentacion_requerimientos_prueba_nueva_v2.pptx`: presentacion final de sustentacion con resultados, explicabilidad, scoring, arquitectura conceptual y conclusiones de negocio.

## Objetivo de la solucion

Construir un modelo supervisado para estimar la probabilidad de fraude en reclamaciones de seguros y convertir esa probabilidad en una bandeja priorizada de investigacion.

La salida esperada no es una decision automatica, sino una priorizacion operativa:

- `score_fraude`: probabilidad estimada o puntaje relativo de riesgo.
- `decil`: grupo de riesgo segun orden descendente del score.
- `prioridad`: nivel de atencion recomendado para negocio.
- `fecha_scoring`: fecha en la que se ejecuto el scoring.
- `version_modelo`: referencia del modelo usado.

## Resumen tecnico

El notebook de Databricks realiza:

1. Lectura de la tabla fuente.
2. Limpieza de nombres de columnas.
3. Creacion de variable objetivo binaria `label`.
4. Analisis de calidad de datos.
5. Ingenieria de variables temporales.
6. Exclusion de variables con riesgo de fuga de informacion.
7. Split temporal entrenamiento/prueba.
8. Entrenamiento de tres modelos:
   - `logistic_regression`
   - `random_forest`
   - `gradient_boosting`
9. Evaluacion con ROC-AUC y area under PR.
10. Calculo de lift por deciles.
11. Generacion de casos puntuados.
12. Guardado de resultados en tablas Delta.

## Resultado principal en Databricks

El mejor modelo ejecutado en Databricks fue `random_forest`.

| Modelo | ROC-AUC | Area under PR |
| --- | ---: | ---: |
| Random forest | 0.785 | 0.613 |
| Gradient boosting | 0.706 | 0.523 |
| Logistic regression | 0.294 | 0.260 |

El ranking por deciles mostro concentracion de fraude en los primeros grupos:

| Decil | Casos | Fraudes | Tasa fraude | Lift |
| ---: | ---: | ---: | ---: | ---: |
| 1 | 320 | 218 | 68.1% | 2.05 |
| 2 | 320 | 226 | 70.6% | 2.12 |
| 3 | 320 | 175 | 54.7% | 1.64 |
| 10 | 319 | 15 | 4.7% | 0.14 |

Interpretacion: el modelo es util para ordenar la cola de investigacion, porque los primeros deciles concentran una tasa de fraude superior a los ultimos deciles.

## Archivos de resultados reales

Los tres archivos `databricks_*_real.csv` son exportaciones directas de los resultados obtenidos en Databricks:

- `databricks_model_metrics_real.csv`: permite validar que el modelo ganador fue `random_forest`.
- `databricks_decile_lift_real.csv.csv`: permite revisar la concentracion de fraude por deciles y el lift del score.
- `databricks_scored_top100_real.csv`: permite inspeccionar casos priorizados, score asociado, decil y variables de negocio usadas para interpretar patrones.

Estos archivos se incluyen para que el evaluador pueda revisar los resultados sin necesidad de ejecutar nuevamente el notebook.

## Consultas SQL utiles en Databricks

Estas consultas se pueden ejecutar en el `SQL Editor` de Databricks despues de correr el notebook.

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

### Consulta de bandeja priorizada

```sql
SELECT *
FROM workspace.default.vw_fraude_bandeja_priorizada
ORDER BY score_fraude DESC
LIMIT 100;
```

## Decisiones tecnicas clave

- Se usa split temporal para simular mejor el comportamiento futuro del modelo.
- Se excluyen variables con riesgo de leakage, como estado final, cierre del siniestro, pagos o informacion posterior al momento de scoring.
- Se generan variables temporales en dias para capturar demoras y antiguedades operativas.
- Se evalua con metricas adecuadas para fraude: ROC-AUC, area under PR y lift por deciles.
- El score se interpreta como una herramienta de priorizacion, no como prueba definitiva de fraude.

## Lectura de negocio

La solucion permite focalizar la capacidad investigativa en los casos con mayor probabilidad de fraude. En lugar de revisar todos los casos por igual, el negocio puede empezar por los deciles 1 y 2, donde se concentra una mayor tasa de fraude observada.

El uso recomendado es una bandeja de trabajo para analistas, investigadores o auditoria, con trazabilidad de score, fecha de scoring y version del modelo.

## Limitaciones

- La etiqueta historica puede reflejar sesgos de investigacion previa.
- El modelo requiere validacion con datos posteriores antes de usarse en produccion.
- El score no reemplaza evidencia documental ni criterio experto.
- Debe monitorearse la estabilidad del score, lift por deciles, drift de variables y desempeno con etiquetas confirmadas.
