# Mantenimiento General

Script completo de mantenimiento que incluye verificación de integridad, actualización de estadísticas, limpieza de historial y reporte de estado general de la instancia.

---

## Configuración

| Variable | Descripción | Default |
|---|---|---|
| `@BaseDatos` | Base a mantener | `'Ventas'` |
| `@ModoEjecucion` | `0`=solo reporte, `1`=ejecutar | `0` |
| `@DiasHistorial` | Días de historial a conservar | `90` |

---

## 1. Estado general de la instancia

```sql
-- Versión y edición
SELECT
    @@SERVERNAME                            AS servidor,
    SERVERPROPERTY('Edition')               AS edicion,
    SERVERPROPERTY('ProductVersion')        AS version,
    SERVERPROPERTY('Collation')             AS collation;

-- Tiempo de actividad
SELECT
    sqlserver_start_time,
    DATEDIFF(DAY,  sqlserver_start_time, GETDATE()) AS dias_activo,
    DATEDIFF(HOUR, sqlserver_start_time, GETDATE()) % 24 AS horas
FROM sys.dm_os_sys_info;
```

---

## 2. Estado de todas las bases de datos

```sql
SELECT
    name                    AS base_datos,
    state_desc              AS estado,
    recovery_model_desc     AS recovery_model,
    log_reuse_wait_desc     AS log_wait,
    is_read_only,
    is_auto_shrink_on,      -- debería ser 0
    is_auto_close_on,       -- debería ser 0 en servidores
    compatibility_level
FROM sys.databases
ORDER BY name;
```

---

## 3. Tamaño de archivos por base

```sql
SELECT
    DB_NAME(database_id)                        AS base_datos,
    name                                        AS archivo,
    type_desc,
    physical_name,
    CAST(size * 8.0 / 1024 AS DECIMAL(10,2))   AS size_MB,
    CASE max_size
        WHEN -1 THEN 'Ilimitado'
        ELSE CAST(CAST(max_size * 8.0 / 1024 AS DECIMAL(10,2)) AS NVARCHAR(20)) + ' MB'
    END                                         AS max_size,
    growth                                      AS crecimiento
FROM sys.master_files
ORDER BY database_id, type;
```

---

## 4. Verificación de integridad (DBCC CHECKDB)

```sql
-- Verificar integridad completa
DBCC CHECKDB ('Ventas') WITH NO_INFOMSGS, ALL_ERRORMSGS;

-- Solo estructura física (más rápido)
DBCC CHECKDB ('Ventas') WITH PHYSICAL_ONLY;

-- Ver la última vez que se corrió CHECKDB por base
SELECT
    name AS base_datos,
    DATABASEPROPERTYEX(name, 'LastGoodDBCCCheckDB') AS ultimo_checkdb
FROM sys.databases
WHERE database_id > 4
ORDER BY name;
```

---

## 5. Actualización de estadísticas

```sql
-- Toda la base
USE Ventas;
EXEC sp_updatestats;

-- Tabla específica con escaneo completo
UPDATE STATISTICS Clientes WITH FULLSCAN;

-- Ver estadísticas desactualizadas
SELECT
    OBJECT_NAME(s.object_id)    AS tabla,
    s.name                      AS estadistica,
    sp.last_updated,
    sp.rows,
    sp.modification_counter     AS modificaciones_desde_update
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE OBJECT_NAME(s.object_id) IS NOT NULL
  AND sp.modification_counter > 0
ORDER BY sp.modification_counter DESC;
```

---

## 6. Limpieza de historial

```sql
DECLARE @FechaCorte DATETIME = DATEADD(DAY, -90, GETDATE());

-- Limpiar historial de jobs
EXEC msdb.dbo.sp_purge_jobhistory @oldest_date = @FechaCorte;

-- Limpiar historial de backups y restores
EXEC msdb.dbo.sp_delete_backuphistory @oldest_date = @FechaCorte;
```

### Ver cuánto se limpiaría antes de ejecutar

```sql
-- Registros de jobs a eliminar
SELECT COUNT(*) AS registros_jobs
FROM msdb.dbo.sysjobhistory
WHERE run_date < CONVERT(INT, CONVERT(NVARCHAR, DATEADD(DAY, -90, GETDATE()), 112));

-- Registros de backups a eliminar
SELECT COUNT(*) AS registros_backups
FROM msdb.dbo.backupset
WHERE backup_start_date < DATEADD(DAY, -90, GETDATE());
```

---

## 7. Estado de jobs y últimas ejecuciones

```sql
-- Jobs con su último resultado
SELECT
    j.name      AS job,
    j.enabled,
    CASE h.run_status
        WHEN 0 THEN 'Fallido'
        WHEN 1 THEN 'Exitoso'
        WHEN 2 THEN 'Reintentando'
        WHEN 3 THEN 'Cancelado'
        WHEN 4 THEN 'En progreso'
        ELSE 'Sin historial'
    END         AS ultimo_resultado,
    CAST(h.run_date AS NVARCHAR(8)) AS ultima_ejecucion,
    h.message
FROM msdb.dbo.sysjobs j
LEFT JOIN (
    SELECT job_id, run_status, run_date, message,
           ROW_NUMBER() OVER (PARTITION BY job_id ORDER BY instance_id DESC) AS rn
    FROM msdb.dbo.sysjobhistory
    WHERE step_id = 0
) h ON j.job_id = h.job_id AND h.rn = 1
ORDER BY j.name;

-- Jobs fallidos en los últimos 7 días
SELECT
    j.name      AS job,
    h.run_date,
    h.run_time,
    h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
WHERE h.run_status = 0
  AND h.step_id = 0
  AND h.run_date >= CONVERT(INT, CONVERT(NVARCHAR, DATEADD(DAY, -7, GETDATE()), 112))
ORDER BY h.run_date DESC;
```

---

## 8. Últimos backups por base

```sql
-- Verificar que todas las bases tienen backup reciente
SELECT
    d.name                          AS base_datos,
    MAX(CASE bs.type WHEN 'D' THEN bs.backup_finish_date END) AS ultimo_full,
    MAX(CASE bs.type WHEN 'I' THEN bs.backup_finish_date END) AS ultimo_diferencial,
    MAX(CASE bs.type WHEN 'L' THEN bs.backup_finish_date END) AS ultimo_log,
    DATEDIFF(DAY,
        MAX(CASE bs.type WHEN 'D' THEN bs.backup_finish_date END),
        GETDATE()
    )                               AS dias_sin_full
FROM sys.databases d
LEFT JOIN msdb.dbo.backupset bs ON d.name = bs.database_name
WHERE d.database_id > 4
  AND d.state_desc = 'ONLINE'
GROUP BY d.name
ORDER BY dias_sin_full DESC;
```

---

## 9. Shrink de archivos (usar con precaución)

```sql
-- Ver espacio usado vs asignado
EXEC sp_helpdb 'Ventas';

-- Shrink del log (solo si es necesario y hay backup de log hecho)
DBCC SHRINKFILE ('Ventas_log', 100);   -- intentar reducir a 100 MB

-- Shrink de la base
DBCC SHRINKDATABASE ('Ventas', 10);    -- 10% de espacio libre
```

> ⚠️ El shrink frecuente fragmenta los índices. No usarlo como rutina habitual.

---

## 10. Limpiar caché (solo para pruebas)

```sql
-- Limpiar caché de planes de ejecución
DBCC FREEPROCCACHE;

-- Limpiar caché de datos (buffer pool)
DBCC DROPCLEANBUFFERS;
```

> ⚠️ No ejecutar en producción salvo que sea estrictamente necesario.

---

## Frecuencias de mantenimiento recomendadas

| Tarea | Frecuencia |
|---|---|
| Backup Full | Diario o semanal |
| Backup Diferencial | Cada 4-12 horas |
| Backup de Log | Cada 15-60 minutos |
| DBCC CHECKDB | Semanal |
| Rebuild Index (>30% frag) | Semanal |
| Reorganize Index (10-30%) | Diario o semanal |
| Update Statistics | Semanal o post carga masiva |
| Cleanup History | Mensual |
