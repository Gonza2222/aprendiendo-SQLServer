# Restore Rápido - Plantillas Completas

Plantillas listas para los escenarios más comunes de restauración. Ejecutar sección por sección según el caso.

---

## Paso 0: Inspeccionar el backup antes de restaurar

```sql
-- Info general del backup (fecha, tipo, base, etc.)
RESTORE HEADERONLY
FROM DISK = 'C:\Backups\Ventas_Full.bak';

-- Nombres lógicos de los archivos (necesario para WITH MOVE)
RESTORE FILELISTONLY
FROM DISK = 'C:\Backups\Ventas_Full.bak';
-- Columnas clave: LogicalName | PhysicalName | Type (D=data, L=log)

-- Verificar integridad sin restaurar
RESTORE VERIFYONLY
FROM DISK = 'C:\Backups\Ventas_Full.bak';
```

---

## Paso 1: Desconectar usuarios antes de restaurar

```sql
USE master;
ALTER DATABASE Ventas SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
-- Cierra todas las conexiones activas. Volver a MULTI_USER al final.
```

---

## Escenario 1: Solo backup Full

```sql
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH RECOVERY, STATS = 10;

ALTER DATABASE Ventas SET MULTI_USER;
```

---

## Escenario 2: Full + Diferencial

```sql
-- Paso 1: Full (NORECOVERY porque viene el diff después)
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY, STATS = 10;

-- Paso 2: Diferencial (último → WITH RECOVERY)
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Diff.bak'
WITH RECOVERY, STATS = 10;

ALTER DATABASE Ventas SET MULTI_USER;
```

---

## Escenario 3: Full + Transaction Logs

```sql
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY, STATS = 10;

RESTORE LOG Ventas FROM DISK = 'C:\Backups\Ventas_Log1.bak' WITH NORECOVERY;
RESTORE LOG Ventas FROM DISK = 'C:\Backups\Ventas_Log2.bak' WITH NORECOVERY;

-- Último log → WITH RECOVERY
RESTORE LOG Ventas FROM DISK = 'C:\Backups\Ventas_Log3.bak' WITH RECOVERY;

ALTER DATABASE Ventas SET MULTI_USER;
```

---

## Escenario 4: Full + Diferencial + Logs

```sql
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY, STATS = 10;

RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Diff.bak'
WITH NORECOVERY, STATS = 10;

RESTORE LOG Ventas FROM DISK = 'C:\Backups\Ventas_Log1.bak' WITH NORECOVERY;
RESTORE LOG Ventas FROM DISK = 'C:\Backups\Ventas_Log2.bak' WITH NORECOVERY;

RESTORE LOG Ventas FROM DISK = 'C:\Backups\Ventas_Log3.bak' WITH RECOVERY;

ALTER DATABASE Ventas SET MULTI_USER;
```

---

## Escenario 5: Restore a un punto exacto en el tiempo (STOPAT)

```sql
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY, STATS = 10;

RESTORE LOG Ventas
FROM DISK = 'C:\Backups\Ventas_Log1.bak'
WITH NORECOVERY;

-- Indicar hasta qué momento restaurar en el último log
RESTORE LOG Ventas
FROM DISK = 'C:\Backups\Ventas_Log2.bak'
WITH RECOVERY,
     STOPAT = '2024-11-15 14:30:00';

ALTER DATABASE Ventas SET MULTI_USER;
```

---

## Escenario 6: Restore con WITH MOVE (rutas diferentes)

Necesario cuando los archivos no existen en la ruta original o al restaurar en otro servidor.

```sql
-- Primero ver los nombres lógicos reales del backup
RESTORE FILELISTONLY FROM DISK = 'C:\Backups\Ventas_Full.bak';

-- Restaurar indicando las nuevas rutas
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH MOVE 'Ventas'     TO 'C:\SQLData\Ventas.mdf',
     MOVE 'Ventas_log' TO 'C:\SQLLogs\Ventas_log.ldf',
     RECOVERY, STATS = 10;

ALTER DATABASE Ventas SET MULTI_USER;
```

> El nombre entre comillas en `MOVE` es el **LogicalName** que devuelve `FILELISTONLY`.

---

## Escenario 7: Restaurar con otro nombre de base

```sql
RESTORE DATABASE VentasCopia
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH MOVE 'Ventas'     TO 'C:\SQLData\VentasCopia.mdf',
     MOVE 'Ventas_log' TO 'C:\SQLLogs\VentasCopia_log.ldf',
     RECOVERY, REPLACE;
```

---

## Escenario 8: Tail-Log Backup + Restore (máxima recuperación)

Usar cuando la base todavía está accesible y querés recuperar hasta el último segundo antes del fallo.

```sql
-- Paso 1: Tail-log backup (deja la BD en modo Restoring)
BACKUP LOG Ventas
TO DISK = 'C:\Backups\Ventas_TailLog.bak'
WITH NORECOVERY, NO_TRUNCATE;

-- Paso 2: Restaurar la cadena completa
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY, STATS = 10;

RESTORE LOG Ventas FROM DISK = 'C:\Backups\Ventas_Log1.bak' WITH NORECOVERY;
RESTORE LOG Ventas FROM DISK = 'C:\Backups\Ventas_Log2.bak' WITH NORECOVERY;

-- Paso 3: Aplicar el tail-log como último paso
RESTORE LOG Ventas
FROM DISK = 'C:\Backups\Ventas_TailLog.bak'
WITH RECOVERY;

ALTER DATABASE Ventas SET MULTI_USER;
```

---

## Utilidades post-restore

```sql
-- Ver estado de todas las bases
SELECT name, state_desc, recovery_model_desc
FROM sys.databases
ORDER BY name;

-- Ver progreso de un restore en curso
SELECT session_id,
       percent_complete,
       estimated_completion_time / 1000 / 60 AS minutos_restantes,
       command,
       status
FROM sys.dm_exec_requests
WHERE command LIKE '%RESTORE%';

-- Ver historial de restauraciones
SELECT rh.destination_database_name,
       rh.restore_date,
       CASE rh.restore_type
           WHEN 'D' THEN 'Full'
           WHEN 'I' THEN 'Diferencial'
           WHEN 'L' THEN 'Log'
       END AS tipo,
       bs.backup_start_date
FROM msdb.dbo.restorehistory rh
JOIN msdb.dbo.backupset bs ON rh.backup_set_id = bs.backup_set_id
ORDER BY rh.restore_date DESC;

-- Finalizar una base atascada en RESTORING
-- RESTORE DATABASE Ventas WITH RECOVERY;

-- Volver a multi-user si quedó en single-user
-- ALTER DATABASE Ventas SET MULTI_USER;
```

---

## Referencia rápida de opciones

| Opción | Cuándo usarla |
|---|---|
| `WITH RECOVERY` | Último paso: deja la BD online |
| `WITH NORECOVERY` | Pasos intermedios: quedan más backups por aplicar |
| `WITH MOVE` | Las rutas de los archivos cambian |
| `WITH REPLACE` | Sobreescribir una base existente |
| `WITH STOPAT` | Recuperar hasta un punto exacto en el tiempo |
| `WITH STANDBY` | Dejar la BD en solo lectura entre logs |
