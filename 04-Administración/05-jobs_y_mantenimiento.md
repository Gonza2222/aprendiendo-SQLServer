# Jobs y Mantenimiento

## SQL Server Agent

El **SQL Server Agent** es un servicio de Windows que permite automatizar tareas mediante **Jobs** (trabajos programados). Debe estar corriendo para que los jobs funcionen.

```sql
-- Ver si el Agent está activo
SELECT servicename, status_desc
FROM sys.dm_server_services
WHERE servicename LIKE '%Agent%';
```

---

## Estructura de un Job

```
Job
 ├── Steps (pasos: T-SQL, SSIS, PowerShell, etc.)
 │     └── Cada paso define qué hacer si tiene éxito o falla
 └── Schedules (cuándo ejecutarse)
       └── Una vez, diario, semanal, cada N minutos, etc.
```

---

## Tablas del sistema en msdb

| Tabla | Contiene |
|---|---|
| `msdb.dbo.sysjobs` | Todos los jobs definidos |
| `msdb.dbo.sysjobsteps` | Los pasos de cada job |
| `msdb.dbo.sysjobschedules` | Asociación job ↔ schedule |
| `msdb.dbo.sysschedules` | Definición de los horarios |
| `msdb.dbo.sysjobhistory` | Historial de ejecuciones |
| `msdb.dbo.syscategories` | Categorías de jobs |

---

## Consultar jobs existentes

```sql
-- Todos los jobs con estado
SELECT j.job_id,
       j.name AS job_name,
       j.enabled,
       c.name AS categoria,
       j.date_created,
       j.date_modified
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.syscategories c ON j.category_id = c.category_id
ORDER BY j.name;

-- Jobs con su próxima ejecución
SELECT j.name,
       j.enabled,
       s.next_run_date,
       s.next_run_time
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobschedules js ON j.job_id = js.job_id
JOIN msdb.dbo.sysschedules s ON js.schedule_id = s.schedule_id;

-- Ver pasos de un job
SELECT step_id, step_name, subsystem, command, on_success_action, on_fail_action
FROM msdb.dbo.sysjobsteps
WHERE job_id = (SELECT job_id FROM msdb.dbo.sysjobs WHERE name = 'NombreDelJob');
```

---

## Historial de ejecuciones

```sql
-- Últimas ejecuciones de todos los jobs
SELECT j.name AS job_name,
       h.run_date,
       h.run_time,
       h.run_duration,
       CASE h.run_status
           WHEN 0 THEN 'Fallido'
           WHEN 1 THEN 'Exitoso'
           WHEN 2 THEN 'Reintentando'
           WHEN 3 THEN 'Cancelado'
           WHEN 4 THEN 'En progreso'
       END AS estado,
       h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
ORDER BY h.run_date DESC, h.run_time DESC;

-- Usar sp_help_job para info completa de un job
EXEC msdb.dbo.sp_help_job @job_name = 'NombreDelJob';
```

---

## Crear un Job con T-SQL

```sql
USE msdb;

-- 1. Crear el job
EXEC sp_add_job
    @job_name = 'BackupDiario_Ventas',
    @enabled = 1,
    @description = 'Backup full de la base Ventas';

-- 2. Agregar un paso T-SQL
EXEC sp_add_jobstep
    @job_name = 'BackupDiario_Ventas',
    @step_name = 'Hacer backup full',
    @subsystem = 'TSQL',
    @command = 'BACKUP DATABASE Ventas TO DISK = ''C:\Backups\Ventas_Full.bak'' WITH FORMAT, STATS = 10;',
    @on_success_action = 1,  -- 1=Quit with success, 2=Quit with failure, 3=Go to next step
    @on_fail_action = 2;

-- 3. Crear un schedule (diario a las 23:00)
EXEC sp_add_schedule
    @schedule_name = 'DiarioNoche',
    @freq_type = 4,           -- 4=Daily
    @freq_interval = 1,       -- cada 1 día
    @active_start_time = 230000;  -- 23:00:00

-- 4. Asociar el schedule al job
EXEC sp_attach_schedule
    @job_name = 'BackupDiario_Ventas',
    @schedule_name = 'DiarioNoche';

-- 5. Asociar el job al servidor
EXEC sp_add_jobserver
    @job_name = 'BackupDiario_Ventas',
    @server_name = '(LOCAL)';
```

---

## Ejecutar un job manualmente

```sql
EXEC msdb.dbo.sp_start_job @job_name = 'BackupDiario_Ventas';
```

---

## Deshabilitar / Habilitar / Eliminar un job

```sql
-- Deshabilitar
EXEC msdb.dbo.sp_update_job @job_name = 'BackupDiario_Ventas', @enabled = 0;

-- Habilitar
EXEC msdb.dbo.sp_update_job @job_name = 'BackupDiario_Ventas', @enabled = 1;

-- Eliminar
EXEC msdb.dbo.sp_delete_job @job_name = 'BackupDiario_Ventas';
```

---

## Maintenance Plans (Planes de mantenimiento)

Los planes de mantenimiento son una alternativa visual (desde SSMS) para automatizar tareas comunes. Internamente también usan SQL Server Agent.

### Tareas típicas de mantenimiento

| Tarea | Para qué |
|---|---|
| **Backup** | Full, diferencial o log periódicos |
| **Rebuild Index** | Reconstruir índices fragmentados |
| **Reorganize Index** | Desfragmentación ligera |
| **Update Statistics** | Actualizar estadísticas del optimizador |
| **Check Database Integrity** | Detectar corrupción (DBCC CHECKDB) |
| **Cleanup History** | Eliminar historial viejo de msdb |
| **Shrink Database** | Reducir tamaño de archivos (usar con cuidado) |

---

## DBCC: comandos de mantenimiento manual

### Verificar integridad

```sql
-- Verificar integridad completa de la base
DBCC CHECKDB ('Ventas');

-- Solo verificar la estructura (más rápido)
DBCC CHECKDB ('Ventas') WITH PHYSICAL_ONLY;

-- Verificar una tabla específica
DBCC CHECKTABLE ('Clientes');
```

### Actualizar estadísticas

```sql
-- Actualizar estadísticas de toda la base
EXEC sp_updatestats;

-- De una tabla específica
UPDATE STATISTICS Clientes;
UPDATE STATISTICS Pedidos WITH FULLSCAN;  -- escaneo completo (más preciso)
```

### Reconstruir índices manualmente

```sql
-- Reconstruir todos los índices de una tabla
ALTER INDEX ALL ON Clientes REBUILD;

-- Reorganizar (más liviano, online)
ALTER INDEX ALL ON Clientes REORGANIZE;
```

### Limpiar caché (para pruebas)

```sql
-- Limpiar caché de planes de ejecución
DBCC FREEPROCCACHE;

-- Limpiar caché de datos
DBCC DROPCLEANBUFFERS;
```

> No usar en producción salvo que sea necesario.

---

## Shrink (reducir tamaño de archivos)

```sql
-- Shrink del archivo de log
DBCC SHRINKFILE ('Ventas_log', 100);  -- intentar reducir a 100 MB

-- Shrink de toda la base
DBCC SHRINKDATABASE ('Ventas', 10);   -- 10% de espacio libre

-- Ver tamaño actual de archivos
EXEC sp_helpdb 'Ventas';
```

> **Advertencia:** hacer shrink frecuente fragmenta los índices. No es una práctica recomendada de rutina.

---

## Frecuencias de mantenimiento recomendadas

| Tarea | Frecuencia sugerida |
|---|---|
| Backup Full | Diario o semanal |
| Backup Diferencial | Cada 4-12 horas |
| Backup de Log | Cada 15-60 minutos |
| DBCC CHECKDB | Semanal |
| Rebuild Index | Semanal (índices con > 30% fragmentación) |
| Reorganize Index | Diario o semanal (10-30% fragmentación) |
| Update Statistics | Semanal o después de cargas masivas |
| Cleanup History | Mensual |
