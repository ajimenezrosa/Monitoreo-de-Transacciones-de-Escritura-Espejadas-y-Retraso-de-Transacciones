# Documentación del Script SQL: Monitoreo de Transacciones de Escritura Espejadas y Retraso de Transacciones

**Propiedad de JOSE ALEJANDRO JIMENEZ ROSA**

![Encabezado Elegante](https://i.ytimg.com/vi/U94hDjd9M2k/hq720.jpg?sqp=-oaymwEhCK4FEIIDSFryq4qpAxMIARUAAAAAGAElAADIQj0AgKJD&rs=AOn4CLDaHjeupvt0YtOA4Nk2ZfazPrdHAw)


---

## Descripción General



Este script SQL está diseñado para monitorear y analizar el rendimiento de las transacciones de escritura espejadas y los retrasos de transacciones en un entorno de SQL Server. Captura métricas de rendimiento, espera un período especificado, vuelve a verificar las métricas y luego agrega y presenta los datos para su análisis.

---

## Desglose del Script

### 1. Configuración Inicial y Recolección de Métricas
El script comienza verificando si existe una tabla temporal `#perf` y la elimina si es así. Luego, crea una nueva tabla `#perf` para almacenar las métricas de rendimiento.

```sql
IF OBJECT_ID('tempdb..#perf') IS NOT NULL
    DROP TABLE #perf

SELECT IDENTITY (int, 1,1) id
    ,instance_name
    ,CAST(cntr_value * 1000 AS DECIMAL(19,2)) [mirrorWriteTrnsMS]
    ,CAST(NULL AS DECIMAL(19,2)) [trnDelayMS]
INTO #perf
FROM sys.dm_os_performance_counters perf
WHERE perf.counter_name LIKE 'Mirrored Write Transactions/sec%'
    AND object_name LIKE 'SQLServer:Database Replica%'
```

### 2. Actualización de Métricas de Retraso de Transacciones
El script actualiza la columna `trnDelayMS` en la tabla `#perf` con los valores de retraso de transacciones.

```sql
UPDATE p
SET p.[trnDelayMS] = perf.cntr_value
FROM #perf p
INNER JOIN sys.dm_os_performance_counters perf ON p.instance_name = perf.instance_name
WHERE perf.counter_name LIKE 'Transaction Delay%'
    AND object_name LIKE 'SQLServer:Database Replica%'
    AND trnDelayMS IS NULL
```

### 3. Período de Espera
Se introduce una demora de 5 minutos para permitir que los contadores de rendimiento se actualicen.

```sql
WAITFOR DELAY '00:05:00'
GO
```

### 4. Recolección de Métricas Nuevamente
El script vuelve a verificar las métricas de rendimiento e inserta los nuevos datos en la tabla `#perf`.

```sql
INSERT INTO #perf
(
    instance_name
    ,mirrorWriteTrnsMS
    ,trnDelayMS
)
SELECT instance_name
    ,CAST(cntr_value * 1000 AS DECIMAL(19,2)) [mirrorWriteTrnsMS]
    ,NULL
FROM sys.dm_os_performance_counters perf
WHERE perf.counter_name LIKE 'Mirrored Write Transactions/sec%'
    AND object_name LIKE 'SQLServer:Database Replica%'
```

### 5. Actualización de Métricas de Retraso de Transacciones Nuevamente
El script actualiza nuevamente la columna `trnDelayMS` con los últimos valores de retraso de transacciones.

```sql
UPDATE p
SET p.[trnDelayMS] = perf.cntr_value
FROM #perf p
INNER JOIN sys.dm_os_performance_counters perf ON p.instance_name = perf.instance_name
WHERE perf.counter_name LIKE 'Transaction Delay%'
    AND object_name LIKE 'SQLServer:Database Replica%'
    AND trnDelayMS IS NULL
```

### 6. Agregación y Presentación de Datos
El script agrega los datos recopilados y los presenta en un formato legible.

```sql
;WITH AG_Stats AS
(
    SELECT AR.replica_server_name,
           HARS.role_desc,
           Db_name(DRS.database_id) [DBName]
    FROM   sys.dm_hadr_database_replica_states DRS
    INNER JOIN sys.availability_replicas AR ON DRS.replica_id = AR.replica_id
    INNER JOIN sys.dm_hadr_availability_replica_states HARS ON AR.group_id = HARS.group_id
        AND AR.replica_id = HARS.replica_id
),
Check1 AS
(
    SELECT DISTINCT p1.instance_name
        ,p1.mirrorWriteTrnsMS
        ,p1.trnDelayMS
    FROM #perf p1
    INNER JOIN
    (
        SELECT instance_name, MIN(id) minId
        FROM #perf p2
        GROUP BY instance_name
    ) p2 ON p1.instance_name = p2.instance_name
),
Check2 AS
(
    SELECT DISTINCT p1.instance_name
        ,p1.mirrorWriteTrnsMS
        ,p1.trnDelayMS
    FROM #perf p1
    INNER JOIN
    (
        SELECT instance_name, MAX(id) minId
        FROM #perf p2
        GROUP BY instance_name
    ) p2 ON p1.instance_name = p2.instance_name
),
AggregatedChecks AS
(
    SELECT DISTINCT c1.instance_name
        , c2.mirrorWriteTrnsMS - c1.mirrorWriteTrnsMS mirrorWriteTrnsMS
        , c2.trnDelayMS - c1.trnDelayMS trnDelayMS
    FROM Check1 c1
    INNER JOIN Check2 c2 ON c1.instance_name = c2.instance_name
),
Pri_CommitTime AS
(
    SELECT replica_server_name
        , DBName
    FROM AG_Stats
    WHERE role_desc = 'PRIMARY'
),
Sec_CommitTime AS
(
    SELECT replica_server_name
        , DBName
    FROM AG_Stats
    WHERE role_desc = 'SECONDARY'
)
SELECT p.replica_server_name [primary_replica]
    , p.[DBName] AS [DatabaseName]
    , s.replica_server_name [secondary_replica]
    , CAST(CASE WHEN ac.trnDelayMS = 0 THEN 1 ELSE ac.trnDelayMS END AS DECIMAL(19,2) / ac.mirrorWriteTrnsMS) sync_lag_MS
FROM Pri_CommitTime p
LEFT JOIN Sec_CommitTime s ON [s].[DBName] = [p].[DBName]
LEFT JOIN AggregatedChecks ac ON ac.instance_name = p.DBName
```

---

## Interpretación de los Datos

- **Primary Replica**: El servidor principal para la base de datos.
- **Database Name**: El nombre de la base de datos que se está monitoreando.
- **Secondary Replica**: El servidor secundario para la base de datos.
- **Sync Lag (MS)**: El retraso de sincronización en milisegundos, calculado como la diferencia en el retraso de transacciones dividido por la diferencia en las transacciones de escritura espejadas.

---

## Conclusión

Este script ayuda a monitorear el rendimiento de las réplicas de bases de datos de SQL Server capturando y analizando las transacciones de escritura espejadas y los retrasos de transacciones. Los datos agregados proporcionan información sobre el retraso de sincronización entre las réplicas primarias y secundarias, lo cual es crucial para mantener el rendimiento y la fiabilidad de la base de datos.

---

**Etiqueta:** `SQL Server`, `Monitoreo`, `Rendimiento`, `Bases de Datos`

