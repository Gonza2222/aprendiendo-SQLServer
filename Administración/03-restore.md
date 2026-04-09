# Restore (Restauración)

## Conceptos clave antes de restaurar

| Concepto | Descripción |
|---|---|
| `WITH RECOVERY` | Finaliza la restauración, la BD queda online. Usar en el **último paso**. |
| `WITH NORECOVERY` | La BD queda en estado "Restoring", lista para recibir más backups. Usar en **pasos intermedios**. |
| `WITH STANDBY` | La BD queda en modo solo lectura entre restauraciones de log. |
| `WITH MOVE` | Cambia la ubicación física de los archivos al restaurar. |
| `WITH REPLACE` | Sobreescribe una base existente aunque no coincidan los nombres. |
| `WITH STOPAT` | Restaura hasta un punto exacto en el tiempo. |

---

## 1. Restore de backup Full

```sql
-- Restauración simple (única operación, BD queda online)
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH RECOVERY, STATS = 10;
```

### Si hay más backups para aplicar después (diferencial o log)

```sql
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY, STATS = 10;
```

---

## 2. Restore Full + Diferencial

```sql
-- Paso 1: restaurar el full
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY, STATS = 10;

-- Paso 2: restaurar el diferencial (último paso → WITH RECOVERY)
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Diff.bak'
WITH RECOVERY, STATS = 10;
```

---

## 3. Restore Full + Logs (point-in-time recovery)

```sql
-- Paso 1: full
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY;

-- Paso 2: log 1
RESTORE LOG Ventas
FROM DISK = 'C:\Backups\Ventas_Log1.bak'
WITH NORECOVERY;

-- Paso 3: log 2
RESTORE LOG Ventas
FROM DISK = 'C:\Backups\Ventas_Log2.bak'
WITH NORECOVERY;

-- Paso 4: último log → WITH RECOVERY
RESTORE LOG Ventas
FROM DISK = 'C:\Backups\Ventas_Log3.bak'
WITH RECOVERY;
```

---

## 4. Restore a un punto exacto en el tiempo (STOPAT)

```sql
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH NORECOVERY;

RESTORE LOG Ventas
FROM DISK = 'C:\Backups\Ventas_Log1.bak'
WITH NORECOVERY;

RESTORE LOG Ventas
FROM DISK = 'C:\Backups\Ventas_Log2.bak'
WITH RECOVERY,
     STOPAT = '2024-11-15 14:30:00';  -- restaura solo hasta ese momento
```

---

## 5. Restore con WITH MOVE

Necesario cuando:
- Los archivos originales tienen una ruta que no existe en el servidor destino
- Querés restaurar la base con un nombre diferente
- Hay conflicto con archivos existentes

```sql
-- Ver los nombres lógicos del backup
RESTORE FILELISTONLY
FROM DISK = 'C:\Backups\Ventas_Full.bak';
-- Devuelve: LogicalName | PhysicalName | Type (D=data, L=log)

-- Restaurar con rutas nuevas
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH MOVE 'Ventas'     TO 'C:\SQLData\Ventas.mdf',
     MOVE 'Ventas_log' TO 'C:\SQLLogs\Ventas_log.ldf',
     RECOVERY, STATS = 10;
```

> El nombre entre comillas en `MOVE` es el **nombre lógico** del archivo (columna `LogicalName` del FILELISTONLY).

---

## 6. Restaurar con otro nombre (base diferente)

```sql
RESTORE DATABASE VentasRestaurada
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH MOVE 'Ventas'     TO 'C:\SQLData\VentasRestaurada.mdf',
     MOVE 'Ventas_log' TO 'C:\SQLLogs\VentasRestaurada_log.ldf',
     RECOVERY;
```

---

## 7. Restore con WITH REPLACE

Fuerza la restauración sobreescribiendo una base existente aunque no coincidan los metadatos.

```sql
RESTORE DATABASE Ventas
FROM DISK = 'C:\Backups\Ventas_Full.bak'
WITH REPLACE, RECOVERY;
```

> Usar con cuidado: sobreescribe sin pedir confirmación.

---

## Errores comunes y soluciones

### Error: sesiones activas en la base

```
Cannot exclusive access to perform the operation.
```

**Solución:** desconectar todas las sesiones antes de restaurar.

```sql
USE master;
ALTER DATABASE Ventas SET SINGLE_USER WITH ROLLBACK IMMEDIATE;

-- Restaurar acá...

ALTER DATABASE Ventas SET MULTI_USER;
```

---

### Error: sector size mismatch

```
Operating system error 1784: The disk sector size is not supported.
```

**Causa:** el backup fue hecho en un disco con sector de 512 bytes y se intenta restaurar en uno de 4096 bytes (o viceversa).

**Solución:** restaurar en un disco compatible, o usar una herramienta de conversión.

---

### Error: logical file name mismatch (WITH MOVE)

```
Logical file 'X' is not part of database 'Y'.
```

**Causa:** el nombre lógico en el WITH MOVE no coincide con el del backup.

**Solución:**

```sql
-- Ver nombres lógicos reales del backup
RESTORE FILELISTONLY FROM DISK = 'C:\Backups\Ventas_Full.bak';

-- Usar exactamente esos nombres en el WITH MOVE
```

---

### Error: base en estado RESTORING atascada

```sql
-- Finalizar la restauración sin más backups
RESTORE DATABASE Ventas WITH RECOVERY;

-- O si querés dejarla sin usar y borrarla
DROP DATABASE Ventas;
```

---

### Error: el log backup no pertenece a la cadena

```
This backup cannot be restored because it is more recent than the last full database backup.
```

**Causa:** se saltó un backup en la cadena o se restauraron en orden incorrecto.

**Solución:** respetar el orden cronológico estricto de los backups de log.

---

## Verificar el estado de la restauración

```sql
-- Ver progreso de un restore en curso
SELECT percent_complete, estimated_completion_time, status, command
FROM sys.dm_exec_requests
WHERE command LIKE '%RESTORE%';

-- Ver estado de las bases
SELECT name, state_desc
FROM sys.databases;
-- state_desc: ONLINE, RESTORING, RECOVERING, SUSPECT, OFFLINE
```

---

## Historial de restauraciones

```sql
SELECT rh.destination_database_name,
       rh.restore_date,
       rh.restore_type,   -- D=Database, L=Log, I=Diferencial
       bs.backup_start_date,
       bs.backup_finish_date
FROM msdb.dbo.restorehistory rh
JOIN msdb.dbo.backupset bs ON rh.backup_set_id = bs.backup_set_id
ORDER BY rh.restore_date DESC;
```

---

## Flujo completo de decisión para restaurar

```
¿Tenés backup de log del momento del fallo?
    └── SÍ → Hacer TAIL-LOG backup primero (si la BD está accesible)
    └── NO → Restaurar hasta el último backup disponible

¿Qué backups tenés?
    └── Solo Full       → RESTORE DATABASE ... WITH RECOVERY
    └── Full + Diff     → Full (NORECOVERY) → Diff (RECOVERY)
    └── Full + Logs     → Full (NORECOVERY) → Logs en orden (NORECOVERY) → último (RECOVERY)
    └── Full+Diff+Logs  → Full (NORECOVERY) → Diff (NORECOVERY) → Logs (NORECOVERY) → último (RECOVERY)

¿Los archivos están en rutas diferentes?
    └── SÍ → Usar RESTORE FILELISTONLY primero, luego WITH MOVE
```

---

## Tail-Log Backup (antes de restaurar en Full recovery)

Si la base todavía está accesible y querés recuperar hasta el último instante:

```sql
-- Backup del "tail" del log (lo que aún no fue respaldado)
BACKUP LOG Ventas
TO DISK = 'C:\Backups\Ventas_TailLog.bak'
WITH NORECOVERY,  -- deja la BD en estado Restoring
     NO_TRUNCATE; -- no trunca el log aunque haya errores de I/O
```

Luego restaurás la cadena completa incluyendo este tail-log como último paso.
