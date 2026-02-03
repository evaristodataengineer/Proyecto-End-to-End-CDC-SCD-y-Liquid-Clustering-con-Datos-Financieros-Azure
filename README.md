# 🏦 Proyecto End-to-End: CDC, SCD y Liquid Clustering con Datos Financieros

## Azure Databricks - Guía Paso a Paso

---

## 📋 Tabla de Contenidos

1. [Descripción del Proyecto](#descripción-del-proyecto)
2. [Arquitectura de la Solución](#arquitectura-de-la-solución)
3. [Requisitos Previos](#requisitos-previos)
4. [Viabilidad de Liquid Clustering en Azure Databricks Standard](#viabilidad-de-liquid-clustering)
5. [Modelo de Datos Financieros](#modelo-de-datos-financieros)
6. [Implementación Paso a Paso](#implementación-paso-a-paso)
7. [Notebooks del Proyecto](#notebooks-del-proyecto)
8. [Optimización y Mejores Prácticas](#optimización-y-mejores-prácticas)
9. [Troubleshooting](#troubleshooting)

---

## 📖 Descripción del Proyecto

Este proyecto implementa un pipeline de datos financieros end-to-end utilizando las características más avanzadas de Delta Lake en Azure Databricks:

| Característica | Descripción |
|----------------|-------------|
| **Change Data Capture (CDC)** | Captura de cambios incrementales en las tablas Delta |
| **Slow Changing Dimensions (SCD)** | Manejo de dimensiones históricas Type 1 y Type 2 |
| **Liquid Clustering** | Optimización automática del layout de datos |
| **Medallion Architecture** | Capas Bronze → Silver → Gold |

### Caso de Uso: Sistema Bancario

Simularemos un sistema bancario con:
- **Clientes**: Información personal y demográfica (SCD Type 2)
- **Cuentas Bancarias**: Estado de cuentas y límites (SCD Type 1 y 2)
- **Transacciones**: Movimientos financieros (Fact Table con CDC)

---

## 🏗️ Arquitectura de la Solución

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AZURE DATABRICKS WORKSPACE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│  │   SOURCE     │    │    BRONZE    │    │    SILVER    │    ┌──────────┐  │
│  │   SYSTEMS    │───▶│    LAYER     │───▶│    LAYER     │───▶│   GOLD   │  │
│  │              │    │              │    │              │    │  LAYER   │  │
│  │ • CSV/JSON   │    │ • Raw Data   │    │ • SCD Type 1 │    │          │  │
│  │ • APIs       │    │ • CDC        │    │ • SCD Type 2 │    │ • KPIs   │  │
│  │ • Databases  │    │   Enabled    │    │ • Cleaned    │    │ • Aggs   │  │
│  └──────────────┘    └──────────────┘    └──────────────┘    └──────────┘  │
│                              │                   │                │        │
│                              ▼                   ▼                ▼        │
│                    ┌─────────────────────────────────────────────────┐     │
│                    │           LIQUID CLUSTERING                     │     │
│                    │  • Optimización automática de archivos          │     │
│                    │  • Data skipping mejorado                       │     │
│                    │  • Sin necesidad de particionamiento manual     │     │
│                    └─────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Flujo de Datos

1. **Bronze Layer**: Ingesta raw con Change Data Feed habilitado
2. **Silver Layer**: Transformaciones con SCD Type 1 y Type 2
3. **Gold Layer**: Agregaciones y métricas financieras
4. **Liquid Clustering**: Aplicado en todas las capas para optimización

---

## ✅ Requisitos Previos

### Azure Databricks Workspace

| Requisito | Mínimo | Recomendado |
|-----------|--------|-------------|
| **Tier** | Standard | Premium (para Unity Catalog completo) |
| **Runtime** | 13.3 LTS | 15.2+ LTS |
| **Cluster Mode** | Single Node | Standard con autoscaling |

### Configuración del Cluster

```json
{
  "spark_version": "15.4.x-scala2.12",
  "node_type_id": "Standard_DS3_v2",
  "num_workers": 2,
  "spark_conf": {
    "spark.databricks.delta.optimizeWrite.enabled": "true",
    "spark.databricks.delta.autoCompact.enabled": "true",
    "spark.databricks.delta.properties.defaults.enableChangeDataFeed": "true"
  }
}
```

---

## 🔍 Viabilidad de Liquid Clustering

### ¿Funciona en Azure Databricks Standard?

| Característica | Standard Tier | Premium Tier |
|----------------|---------------|--------------|
| **Liquid Clustering Manual** (`CLUSTER BY`) | ✅ Sí | ✅ Sí |
| **OPTIMIZE con Clustering** | ✅ Sí | ✅ Sí |
| **Automatic Liquid Clustering** (`CLUSTER BY AUTO`) | ❌ No* | ✅ Sí |
| **Predictive Optimization** | ❌ No | ✅ Sí |

> *Automatic Liquid Clustering requiere Unity Catalog con Predictive Optimization habilitado.

### Requisitos de Runtime para Liquid Clustering

```
DBR 13.3 LTS  → Public Preview (limitaciones)
DBR 14.2+    → DataFrame APIs disponibles
DBR 15.2+    → GA (Generally Available) - RECOMENDADO
DBR 15.4 LTS → Automatic Liquid Clustering disponible
```

### Beneficios de Liquid Clustering vs Partitioning/Z-Order

| Aspecto | Partitioning + Z-Order | Liquid Clustering |
|---------|------------------------|-------------------|
| Cambio de keys | Requiere reescritura | Sin reescritura |
| Mantenimiento | Alto | Bajo |
| Concurrencia | Limitada | Row-level |
| Data Skipping | Bueno | Excelente |
| Cardinality alta | Problemas | Sin problemas |

---

## 📊 Modelo de Datos Financieros

### Diagrama Entidad-Relación

```
┌─────────────────────┐         ┌─────────────────────┐
│     DIM_CLIENTES    │         │    DIM_CUENTAS      │
│  (SCD Type 2)       │         │ (SCD Type 1 & 2)    │
├─────────────────────┤         ├─────────────────────┤
│ PK: cliente_key     │         │ PK: cuenta_key      │
│ BK: cliente_id      │◀────────│ FK: cliente_id      │
│ nombre              │         │ BK: numero_cuenta   │
│ email               │         │ tipo_cuenta         │
│ segmento_cliente    │         │ saldo_actual        │
│ direccion           │         │ limite_credito      │
│ fecha_inicio        │         │ estado              │
│ fecha_fin           │         │ fecha_inicio        │
│ es_actual           │         │ fecha_fin           │
│ version             │         │ es_actual           │
└─────────────────────┘         └─────────────────────┘
                                         │
                                         │
                                         ▼
                               ┌─────────────────────┐
                               │  FACT_TRANSACCIONES │
                               │     (CDC)           │
                               ├─────────────────────┤
                               │ PK: transaccion_id  │
                               │ FK: cuenta_key      │
                               │ FK: cliente_key     │
                               │ tipo_transaccion    │
                               │ monto               │
                               │ fecha_transaccion   │
                               │ canal               │
                               │ estado              │
                               └─────────────────────┘
```

### Tipos de SCD Implementados

#### SCD Type 1 (Sobrescritura)
- **Uso**: Corrección de errores, datos que no requieren historial
- **Ejemplo**: Corrección de email, actualización de teléfono

#### SCD Type 2 (Historial Completo)
- **Uso**: Cambios que requieren trazabilidad histórica
- **Ejemplo**: Cambio de dirección, cambio de segmento de cliente
- **Columnas adicionales**: `fecha_inicio`, `fecha_fin`, `es_actual`, `version`

---

## 🚀 Implementación Paso a Paso

### Paso 1: Configuración del Ambiente

```python
# Crear base de datos
spark.sql("CREATE DATABASE IF NOT EXISTS financial_lakehouse")
spark.sql("USE financial_lakehouse")

# Habilitar CDC a nivel de sesión (opcional)
spark.conf.set("spark.databricks.delta.properties.defaults.enableChangeDataFeed", "true")
```

### Paso 2: Crear Tablas Bronze con CDC

```python
# Ejemplo: Tabla Bronze de Clientes con CDC y Liquid Clustering
spark.sql("""
    CREATE TABLE IF NOT EXISTS bronze_clientes (
        cliente_id STRING,
        nombre STRING,
        email STRING,
        telefono STRING,
        direccion STRING,
        ciudad STRING,
        pais STRING,
        fecha_nacimiento DATE,
        segmento_cliente STRING,
        fecha_registro TIMESTAMP,
        fuente STRING,
        fecha_ingesta TIMESTAMP
    )
    USING DELTA
    CLUSTER BY (cliente_id, fecha_ingesta)
    TBLPROPERTIES (
        'delta.enableChangeDataFeed' = 'true',
        'delta.autoOptimize.optimizeWrite' = 'true'
    )
""")
```

### Paso 3: Implementar SCD Type 2 en Silver

```python
def apply_scd_type2(source_df, target_table, key_columns, tracked_columns):
    """
    Implementa SCD Type 2 usando MERGE
    """
    from delta.tables import DeltaTable
    from pyspark.sql.functions import current_timestamp, lit, col
    
    target = DeltaTable.forName(spark, target_table)
    
    # Merge condition
    merge_condition = " AND ".join([
        f"target.{k} = source.{k}" for k in key_columns
    ]) + " AND target.es_actual = true"
    
    # Update condition (detectar cambios)
    update_condition = " OR ".join([
        f"target.{c} <> source.{c}" for c in tracked_columns
    ])
    
    target.alias("target").merge(
        source_df.alias("source"),
        merge_condition
    ).whenMatchedUpdate(
        condition=update_condition,
        set={
            "fecha_fin": current_timestamp(),
            "es_actual": lit(False)
        }
    ).whenNotMatchedInsert(
        values={
            **{c: col(f"source.{c}") for c in source_df.columns},
            "fecha_inicio": current_timestamp(),
            "fecha_fin": lit(None),
            "es_actual": lit(True),
            "version": lit(1)
        }
    ).execute()
```

### Paso 4: Leer Change Data Feed

```python
# Lectura batch del CDF
changes_df = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", 0) \
    .table("bronze_clientes")

# Lectura streaming del CDF
stream_df = spark.readStream.format("delta") \
    .option("readChangeFeed", "true") \
    .table("bronze_clientes")
```

### Paso 5: Aplicar Liquid Clustering

```python
# Optimizar tabla con Liquid Clustering
spark.sql("OPTIMIZE silver_clientes")

# Ver estadísticas de clustering
spark.sql("DESCRIBE DETAIL silver_clientes")
```

---

## 📓 Notebooks del Proyecto

| Notebook | Descripción |
|----------|-------------|
| `01_Setup_Ambiente.ipynb` | Configuración inicial, creación de base de datos y tablas |
| `02_Bronze_CDC.ipynb` | Ingesta de datos con Change Data Feed |
| `03_Silver_SCD.ipynb` | Implementación de SCD Type 1 y Type 2 |
| `04_Gold_Analytics.ipynb` | Métricas y KPIs financieros |
| `05_Liquid_Clustering.ipynb` | Optimización con Liquid Clustering |

---

## 🎯 Optimización y Mejores Prácticas

### Liquid Clustering

1. **Selección de columnas de clustering**: Máximo 4 columnas, ordenadas por frecuencia de filtrado
2. **Ejecutar OPTIMIZE regularmente**: Después de escrituras significativas
3. **No combinar con partitioning**: Liquid Clustering reemplaza el particionamiento

### Change Data Feed

1. **Retención**: Configurar `delta.logRetentionDuration` adecuadamente
2. **Checkpoints**: Usar checkpoints en streaming para recuperación
3. **Versiones**: Monitorear versiones para evitar pérdida de datos

### SCD

1. **Índices**: Crear índices en columnas de búsqueda frecuente
2. **Compactación**: Usar OPTIMIZE para mantener rendimiento
3. **Cleanup**: Implementar proceso de limpieza de versiones antiguas

---

## 🔧 Troubleshooting

### Error: "Change Data Feed not enabled"

```sql
-- Habilitar CDC en tabla existente
ALTER TABLE nombre_tabla SET TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true')
```

### Error: "Liquid Clustering not supported"

Verificar:
1. Runtime >= 13.3 LTS (recomendado 15.2+)
2. Tabla no tiene particionamiento
3. No se está usando con Z-ORDER

### Error: "Version not found"

```sql
-- Verificar historial disponible
DESCRIBE HISTORY nombre_tabla

-- Ejecutar VACUUM con precaución
VACUUM nombre_tabla RETAIN 168 HOURS
```

---

## 📚 Referencias

- [Delta Lake Change Data Feed](https://docs.databricks.com/delta/delta-change-data-feed.html)
- [Liquid Clustering Documentation](https://docs.databricks.com/delta/clustering.html)
- [SCD Implementation Patterns](https://docs.databricks.com/delta-live-tables/cdc.html)

---

