# Monitoreo de Sesiones y Queries

Queries listas para monitorear en tiempo real: sesiones activas, bloqueos, queries lentas, esperas y uso de recursos. Ejecutar la sección según el problema a diagnosticar.

---

## 1. Sesiones activas

```sql
-- Vista rápida clásica
EXEC sp_who2;

-- Vista detallada con DMVs
SELECT
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    DB_NAME(s.database_id)      AS base_datos,
    s.status,
    s.cpu_time                  AS cpu_ms,
    s.memory_usage * 8          AS memoria_kb,
    s.total_elapsed_time / 1000 AS elapsed_seg,
    s.last_request_start_time,
    s.reads,
    s.writes,
    s.logical_reads
FROM sys.dm_exec_sessions s
WHERE s.is_user_process = 1
ORDER BY s.cpu_time DESC;
```

---

## 2. Queries en ejecución ahora mismo

```sql
SELECT
    r.session_id,
    r.status,
    r.blocking_session_id,
    r.wait_type,
    r.wait_time / 1000              AS espera_seg,
    r.total_elapsed_time / 1000     AS elapsed_seg,
    r.cpu_time                      AS cpu_ms,
    r.logical_reads,
    DB_NAME(r.database_id)          AS base_datos,
    SUBSTRING(t.text, 1, 500)       AS query_texto,
    r.percent_complete,
    r.estimated_completion_time / 1000 AS fin_estimado_seg
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id > 50
ORDER BY r.total_elapsed_time DESC;
```

---

## 3. Bloqueos activos

```sql
SELECT
    bloqueada.session_id            AS sesion_bloqueada,
    bloqueante.session_id           AS sesion_bloqueante,
    bloqueada.wait_type,
    bloqueada.wait_time / 1000      AS espera_seg,
    DB_NAME(bloqueada.database_id)  AS base_datos,
    s_bloqueada.login_name          AS usuario_bloqueado,
    s_bloqueante.login_name         AS usuario_bloqueante,
    s_bloqueante.host_name          AS host_bloqueante,
    SUBSTRING(t.text, 1, 300)       AS query_bloqueada
FROM sys.dm_exec_requests bloqueada
JOIN sys.dm_exec_sessions s_bloqueada  ON bloqueada.session_id = s_bloqueada.session_id
JOIN sys.dm_exec_sessions s_bloqueante ON bloqueada.blocking_session_id = s_bloqueante.session_id
CROSS APPLY sys.dm_exec_sql_text(bloqueada.sql_handle) t
WHERE bloqueada.blocking_session_id > 0
ORDER BY bloqueada.wait_time DESC;

-- Ver qué ejecuta el bloqueante (reemplazar 55 con el session_id)
DBCC INPUTBUFFER(55);

-- Terminar una sesión (usar con cuidado)
-- KILL 55;
```

---

## 4. Historial de deadlocks

```sql
SELECT
    xdr.value('@timestamp', 'datetime2')    AS fecha,
    xdr.query('.')                          AS deadlock_xml
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets t
    JOIN sys.dm_xe_sessions s ON t.event_session_address = s.address
    WHERE s.name = 'system_health'
      AND t.target_name = 'ring_buffer'
) AS data
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS xdt(xdr)
ORDER BY fecha DESC;
```

---

## 5. Queries más costosas (caché)

### Por CPU promedio

```sql
SELECT TOP 15
    qs.execution_count,
    qs.total_worker_time / qs.execution_count   AS avg_cpu_us,
    qs.total_elapsed_time / qs.execution_count  AS avg_elapsed_us,
    qs.total_logical_reads / qs.execution_count AS avg_lecturas,
    SUBSTRING(t.text, 1, 400)                   AS query_texto
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY avg_cpu_us DESC;
```

### Por lecturas lógicas

```sql
SELECT TOP 15
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_lecturas,
    qs.total_worker_time / qs.execution_count   AS avg_cpu_us,
    SUBSTRING(t.text, 1, 400)                   AS query_texto
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY avg_lecturas DESC;
```

### Por tiempo de ejecución

```sql
SELECT TOP 15
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count  AS avg_elapsed_us,
    qs.total_worker_time / qs.execution_count   AS avg_cpu_us,
    SUBSTRING(t.text, 1, 400)                   AS query_texto
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY avg_elapsed_us DESC;
```

---

## 6. Esperas del servidor (Wait Stats)

```sql
SELECT TOP 20
    wait_type,
    waiting_tasks_count,
    CAST(wait_time_ms / 1000.0 AS DECIMAL(10,2))     AS wait_seg,
    CAST(max_wait_time_ms / 1000.0 AS DECIMAL(10,2)) AS max_wait_seg,
    CAST(signal_wait_time_ms * 100.0 / NULLIF(wait_time_ms, 0) AS DECIMAL(5,2)) AS pct_cpu_wait
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','BROKER_TO_FLUSH','BROKER_TASK_STOP','CLR_AUTO_EVENT',
    'DISPATCHER_QUEUE_SEMAPHORE','FT_IFTS_SCHEDULER_IDLE_WAIT',
    'HADR_WORK_QUEUE','LAZYWRITER_SLEEP','LOGMGR_QUEUE',
    'ONDEMAND_TASK_QUEUE','REQUEST_FOR_DEADLOCK_SEARCH','RESOURCE_QUEUE',
    'SERVER_IDLE_CHECK','SLEEP_DBSTARTUP','SLEEP_DCOMSTARTUP',
    'SLEEP_MASTERDBREADY','SLEEP_MASTERMDREADY','SLEEP_MASTERUPGRADED',
    'SLEEP_MSDBSTARTUP','SLEEP_TEMPDBSTARTUP','SNI_HTTP_ACCEPT',
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

## 7. Uso de memoria

```sql
-- Memoria usada por SQL Server
SELECT
    physical_memory_in_use_kb / 1024    AS memoria_usada_MB,
    memory_utilization_percentage
FROM sys.dm_os_process_memory;

-- Buffer pool por base de datos
SELECT
    DB_NAME(database_id)                AS base_datos,
    COUNT(*) * 8 / 1024                 AS buffer_MB
FROM sys.dm_os_buffer_descriptors
WHERE database_id > 4
GROUP BY database_id
ORDER BY buffer_MB DESC;

-- RAM disponible del sistema
SELECT
    total_physical_memory_kb / 1024     AS ram_total_MB,
    available_physical_memory_kb / 1024 AS ram_disponible_MB,
    system_memory_state_desc            AS estado_memoria
FROM sys.dm_os_sys_memory;
```

---

## 8. I/O por archivo

```sql
SELECT
    DB_NAME(fs.database_id)                             AS base_datos,
    mf.physical_name                                    AS archivo,
    mf.type_desc,
    fs.io_stall_read_ms  / NULLIF(fs.num_of_reads, 0)  AS avg_read_ms,
    fs.io_stall_write_ms / NULLIF(fs.num_of_writes, 0) AS avg_write_ms,
    fs.num_of_reads,
    fs.num_of_writes,
    fs.io_stall
FROM sys.dm_io_virtual_file_stats(NULL, NULL) fs
JOIN sys.master_files mf
    ON fs.database_id = mf.database_id
    AND fs.file_id = mf.file_id
ORDER BY fs.io_stall DESC;
```

---

## 9. Conexiones por base de datos

```sql
SELECT
    DB_NAME(database_id)    AS base_datos,
    COUNT(*)                AS conexiones_activas
FROM sys.dm_exec_sessions
WHERE is_user_process = 1
GROUP BY database_id
ORDER BY conexiones_activas DESC;
```

---

## DMVs de referencia rápida

| DMV | Para qué |
|---|---|
| `sys.dm_exec_sessions` | Sesiones activas |
| `sys.dm_exec_requests` | Queries en ejecución |
| `sys.dm_exec_query_stats` | Estadísticas históricas de queries |
| `sys.dm_exec_sql_text` | Texto de una query por handle |
| `sys.dm_os_wait_stats` | Estadísticas de esperas |
| `sys.dm_os_buffer_descriptors` | Uso del buffer pool |
| `sys.dm_io_virtual_file_stats` | I/O por archivo |
| `sys.dm_os_process_memory` | Uso de memoria del proceso |
