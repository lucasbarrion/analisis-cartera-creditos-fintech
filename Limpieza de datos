## 🧹 Limpieza de Datos en Snowflake

La limpieza de los datos se resuelve en distintos momentos del pipeline, dependiendo de qué tipo de problema ataca cada técnica.

### Conversión segura de tipos de datos

Los datos llegan a las tablas de aterrizaje (`_raw`) con todas las columnas en formato texto, para no arriesgar que una carga falle por un valor inesperado. La limpieza de tipos ocurre en el proceso automático (Stream + Task) que mueve los datos hacia las tablas finales, usando funciones que convierten de forma segura sin romper la carga si encuentran un valor corrupto:

```sql
destino.monto_solicitado   = TRY_TO_DECIMAL(origen.monto_solicitado, 18, 2),
destino.plazo_meses        = TRY_TO_NUMBER(origen.plazo_meses),
destino.tasa_nominal_anual = TRY_TO_DECIMAL(origen.tasa_nominal_anual, 5, 2)
```

A diferencia de un `CAST` tradicional, `TRY_TO_DECIMAL`/`TRY_TO_NUMBER` devuelven `NULL` en vez de detener todo el proceso si un valor no se puede convertir — así una fila con datos corruptos no tira abajo la carga completa.

### Columnas de control derivadas

Durante la misma carga se generan columnas que no vienen en el dato original, pero que son necesarias para el análisis posterior (por ejemplo, detectar pagos con montos negativos, o cuotas que todavía no vencieron):

```sql
destino.monto_pagado_inconsistente = IFF(TRY_TO_DECIMAL(origen.monto_pagado, 18, 2) < 0, TRUE, FALSE),
destino.es_cuota_futura            = IFF(origen.fecha_vencimiento > CURRENT_DATE(), TRUE, FALSE)
```

Estas dos columnas se usan luego como filtro en las vistas de hallazgos, para no contaminar los cálculos de mora con cuotas que ni siquiera vencieron todavía.

### Corrección de valores negativos

Se detectaron pagos cargados con montos negativos (probablemente error de tipeo o de exportación en el sistema origen). En vez de descartar esas filas, se corrige el valor con `ABS()` y al mismo tiempo se deja registrada la anomalía en la columna de control `monto_pagado_inconsistente`, para no perder trazabilidad de qué filas fueron corregidas:

```sql
destino.monto_pagado = ABS(TRY_TO_DECIMAL(origen.monto_pagado, 18, 2))
```

### Cálculo controlado de días de atraso

El cálculo de `dias_atraso` (`DATEDIFF` entre vencimiento y pago) tenía dos casos borde que había que contemplar explícitamente: cuotas que todavía no se pagaron (no se puede calcular atraso, tiene que quedar en `NULL`, no en 0 ni en un número negativo) y pagos anticipados (pagados antes de la fecha de vencimiento, que arrojan un `DATEDIFF` negativo y no representan mora real):

```sql
destino.dias_atraso = CASE
    WHEN origen.fecha_pago IS NULL THEN NULL
    WHEN DATEDIFF(DAY, origen.fecha_vencimiento, origen.fecha_pago) < 0 THEN 0
    ELSE DATEDIFF(DAY, origen.fecha_vencimiento, origen.fecha_pago)
END
```

### Estandarización de categorías inconsistentes

Un relevamiento sobre la columna `PROVINCIA` mostró que un mismo valor real llegaba escrito de formas distintas por errores de tipeo, mayúsculas inconsistentes o espacios de más (por ejemplo, `"CABA"`, `"C.A.B.A."` y `"Capital Federal"` referían al mismo lugar). Esto hubiera partido artificialmente los mismos datos en categorías separadas al analizarlos.

Para resolverlo, se normalizaron los valores comparándolos con `UPPER()` y `TRIM()` (para ignorar diferencias de mayúsculas/minúsculas y espacios en blanco al inicio o final del texto) y actualizándolos a un único valor canónico por provincia:

```sql
UPDATE originacion_creditos SET provincia = 'CABA'
WHERE UPPER(TRIM(provincia)) IN ('CABA', 'C.A.B.A.', 'CAPITAL FEDERAL');

UPDATE originacion_creditos SET provincia = 'Buenos Aires'
WHERE UPPER(TRIM(provincia)) IN ('BUENOS AIRES', 'BS AS');

UPDATE originacion_creditos SET provincia = 'Sin especificar'
WHERE provincia IS NULL OR TRIM(provincia) = '';
```

El mismo criterio se aplicó sobre las demás provincias de la cartera, y se replicó también en `dim_cliente`, ya que esta dimensión se había poblado copiando los datos desde la misma tabla original.

### Manejo explícito de NULL en los filtros

Las columnas de control (`es_cuota_futura`, `monto_pagado_inconsistente`) quedaron en `NULL` para las filas históricas cargadas antes de que existiera este proceso automático. Un filtro directo como `WHERE es_cuota_futura = FALSE` descarta también las filas en `NULL` (porque en SQL, `NULL = FALSE` no da verdadero, da desconocido), perdiendo datos históricos válidos sin querer. Por eso, en las vistas de hallazgos, los filtros se escriben con `COALESCE`:

```sql
WHERE COALESCE(c.es_cuota_futura, FALSE) = FALSE
  AND COALESCE(c.monto_pagado_inconsistente, FALSE) = FALSE
```

### Eliminación de duplicados

El mismo patrón de `ROW_NUMBER()` se usó en dos lugares del proyecto: en `dim_cliente`, donde clientes con más de un crédito solicitado terminaban repetidos, y en `bcra_indicadores`, donde una recarga sin control de duplicados había duplicado los registros mensuales de inflación. En ambos casos se resolvió quedándose con una sola fila por clave:

```sql
CREATE OR REPLACE TABLE dim_cliente AS
SELECT cliente_id, provincia, edad_cliente, ingreso_declarado_ars
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY cliente_id ORDER BY fecha_solicitud DESC) AS rn
    FROM originacion_creditos
)
WHERE rn = 1;
```

```sql
CREATE OR REPLACE TABLE bcra_indicadores AS
SELECT * EXCLUDE (rn)
FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY fecha ORDER BY fecha_carga) AS rn
    FROM bcra_indicadores
)
WHERE rn = 1;
```

### Resumen de errores de calidad de datos detectados y corregidos

- **Valores de texto inconsistentes:** la columna `PROVINCIA` tenía el mismo dato real escrito de formas distintas. Se normalizó con `UPPER()` + `TRIM()`.
- **Montos negativos:** algunos pagos llegaron cargados con `monto_pagado` en negativo. Se corrigieron con `ABS()`, dejando registrada la anomalía en `monto_pagado_inconsistente`.
- **Registros duplicados:** en `dim_cliente` y en `bcra_indicadores`. Se resolvió con `ROW_NUMBER()` quedándose con una sola fila por clave.
- **Valores NULL mal interpretados en filtros:** filtrar directo por `= FALSE` descartaba filas históricas en `NULL` sin querer. Se corrigió con `COALESCE(columna, FALSE)`.
- **Días de atraso mal calculados en casos borde:** pagos anticipados y cuotas sin pagar. Se resolvió con lógica `CASE` que distingue ambos casos.
- **Datos futuros contaminando los cálculos:** cuotas que todavía no vencieron. Se creó la columna de control `es_cuota_futura` para excluirlas de los análisis.
