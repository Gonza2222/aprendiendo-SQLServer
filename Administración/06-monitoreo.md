# Monitoreo

## ¿Qué monitorear?

- Conexiones y sesiones activas
- Queries lentas o bloqueadas
- Uso de CPU, memoria y disco
- Estado de los jobs
- Tamaño y crecimiento de bases y logs
- Fragmentación de índices
- Deadlocks

---

## 1. Sesiones y conexiones activas

### sp_who y sp_who2

```sql
-- Info básica de conexiones
EXEC sp_who;

-- Info más detallada (incluye CPU, I/O, memoria, estado)
EXEC sp_who2;

-- Filtrar por base de datos
EXEC sp_who2 'active';  -- solo sesiones activas
```

### sys.dm_exec_sessions (DMV)

```sql
SELECT session_id,
       login_name,
       status,          -- running, sleeping, blocked
       host_name,
       program_name,
       database_id,
       DB_NAME(database_id) AS base_datos,
       cpu_time,
       memory_usage,
       total_elapsed_time,
       last_request_start_time
FROM sys.dm_exec_sessions
WHERE is_user_process = 1   -- excluir procesos del sistema
ORDER BY cpu_time DESC;
```

---

## 2. Requests activos (queries en ejecución)

```sql
SELECT r.session_id,
       r.status,
       r.blocking_session_id,
       r.wait_type,
       r.wait_time,
       r.total_elapsed_time,
       r.cpu_time,
       r.logical_reads,
       SUBSTRING(t.text, 1, 200) AS query_texto
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50   -- excluir procesos del sistema
ORDER BY r.total_elapsed_time DESC;
```

---

## 3. Detectar bloqueos

```sql
-- Sesiones bloqueadas y quién las bloquea
SELECT blocked.session_id AS sesion_bloqueada,
       blocking.session_id AS sesion_bloqueante,
       blocked.wait_type,
       blocked.wait_time / 1000.0 AS espera_segundos,
       SUBSTRING(t.text, 1, 200) AS query_bloqueada
FROM sys.dm_exec_requests blocked
JOIN sys.dm_exec_sessions blocking ON blocked.blocking_session_id = blocking.session_id
CROSS APPLY sys.dm_exec_sql_text(blocked.sql_handle) t
WHERE blocked.blocking_session_id > 0;
```

### Terminar una sesión bloqueada

```sql
KILL 55;  -- reemplazar 55 con el session_id
```

---

## 4. Queries más costosas

### Por CPU

```sql
SELECT TOP 10
    qs.total_worker_time / qs.execution_count AS avg_cpu,
    qs.execution_count,
    qs.total_worker_time,
    SUBSTRING(t.text, 1, 300) AS query_texto
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY avg_cpu DESC;
```

### Por lecturas lógicas

```sql
SELECT TOP 10
    qs.total_logical_reads / qs.execution_count AS avg_lecturas,
    qs.execution_count,
    SUBSTRING(t.text, 1, 300) AS query_texto
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY avg_lecturas DESC;
```

### Por tiempo de ejecución

```sql
SELECT TOP 10
    qs.total_elapsed_time / qs.execution_count AS avg_tiempo_us,
    qs.execution_count,
    SUBSTRING(t.text, 1, 300) AS query_texto
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY avg_tiempo_us DESC;
```

---

## 5. Uso de memoria

```sql
-- Memoria total del servidor y uso de SQL Server
SELECT physical_memory_in_use_kb / 1024 AS memoria_usada_MB,
       page_fault_count,
       memory_utilization_percentage
FROM sys.dm_os_process_memory;

-- Buffer pool (páginas cacheadas por base)
SELECT DB_NAME(database_id) AS base_datos,
       COUNT(*) * 8 / 1024 AS memoria_MB
FROM sys.dm_os_buffer_descriptors
WHERE database_id > 4  -- excluir bases del sistema
GROUP BY database_id
ORDER BY memoria_MB DESC;
```

---

## 6. Esperas del servidor (Wait Stats)

Las wait stats indican dónde pasa el tiempo SQL Server cuando no está ejecutando queries.

```sql
SELECT TOP 15
    wait_type,
    waiting_tasks_count,
    wait_time_ms / 1000.0 AS wait_time_seg,
    max_wait_time_ms / 1000.0 AS max_wait_seg,
    signal_wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','BROKER_TO_FLUSH','BROKER_TASK_STOP',
    'CLR_AUTO_EVENT','DISPATCHER_QUEUE_SEMAPHORE',
    'FT_IFTS_SCHEDULER_IDLE_WAIT','HADR_WORK_QUEUE',
    'LAZYWRITER_SLEEP','LOGMGR_QUEUE','ONDEMAND_TASK_QUEUE',
    'REQUEST_FOR_DEADLOCK_SEARCH','RESOURCE_QUEUE',
    'SERVER_IDLE_CHECK','SLEEP_DBSTARTUP','SLEEP_DCOMSTARTUP',
    'SLEEP_MASTERDBREADY','SLEEP_MASTERMDREADY',
    'SLEEP_MASTERUPGRADED','SLEEP_MSDBSTARTUP',
    'SLEEP_TEMPDBSTARTUP','SNI_HTTP_ACCEPT',
    'SP_SERVER_DIAGNOSTICS_SLEEP','SQLTRACE_BUFFER_FLUSH',
    'WAITFOR','XE_DISPATCHER_WAIT','XE_TIMER_EVENT'
)
ORDER BY wait_time_ms DESC;
```

### Tipos de espera comunes

| Wait type | Posible causa |
|---|---|
| `LCK_M_X` | Bloqueos de escritura exclusiva |
| `PAGEIOLATCH_SH/EX` | I/O de disco lento |
| `CXPACKET` | Paralelismo excesivo |
| `SOS_SCHEDULER_YIELD` | Alta presión de CPU |
| `WRITELOG` | I/O del log lento |
| `ASYNC_NETWORK_IO` | Red o cliente lento |

---

## 7. Tamaño de bases de datos y archivos

```sql
-- Tamaño de todas las bases
SELECT name,
       CAST(size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_MB,
       state_desc
FROM sys.master_files
ORDER BY size DESC;

-- Espacio usado/libre dentro de cada base
EXEC sp_helpdb;

-- Detalle por base
SELECT DB_NAME() AS base,
       name AS archivo,
       physical_name,
       CAST(size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_MB,
       CAST(FILEPROPERTY(name, 'SpaceUsed') * 8.0 / 1024 AS DECIMAL(10,2)) AS usado_MB
FROM sys.database_files;
```

---

## 8. Monitorear jobs fallidos

```sql
SELECT j.name AS job_name,
       h.run_date,
       h.run_time,
       h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
WHERE h.run_status = 0   -- 0 = fallido
ORDER BY h.run_date DESC, h.run_time DESC;
```

---

## 9. Fragmentación de índices

```sql
SELECT OBJECT_NAME(ips.object_id) AS tabla,
       i.name AS indice,
       ips.avg_fragmentation_in_percent,
       ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 10
  AND ips.page_count > 100
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

---

## 10. Activity Monitor (GUI)

En SSMS: clic derecho sobre el servidor → **Activity Monitor**

Muestra en tiempo real:
- Procesos activos
- Esperas de recursos
- Accesos a disco recientes
- Queries costosas recientes

---

## 11. Extended Events (eventos extendidos)

Alternativa moderna y liviana al SQL Profiler para capturar eventos específicos.

```sql
-- Ver sesiones de Extended Events activas
SELECT name, state_desc
FROM sys.dm_xe_sessions;
```

Desde SSMS: Management → Extended Events → Sessions → New Session (wizard)

---

## DMVs de referencia rápida

| DMV | Para qué |
|---|---|
| `sys.dm_exec_sessions` | Sesiones activas |
| `sys.dm_exec_requests` | Queries en ejecución |
| `sys.dm_exec_query_stats` | Estadísticas de queries ejecutadas |
| `sys.dm_exec_sql_text` | Texto de una query (por handle) |
| `sys.dm_os_wait_stats` | Estadísticas de esperas |
| `sys.dm_os_buffer_descriptors` | Uso del buffer pool |
| `sys.dm_db_index_physical_stats` | Fragmentación de índices |
| `sys.dm_io_virtual_file_stats` | I/O por archivo |
| `sys.dm_os_process_memory` | Uso de memoria del proceso |
| `sys.dm_server_services` | Estado de los servicios SQL Server |
