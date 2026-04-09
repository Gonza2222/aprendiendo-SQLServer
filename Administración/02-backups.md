# Backups

## Tipos de backup

| Tipo | Qué respalda | Recovery model necesario |
|---|---|---|
| **Full** | Toda la base de datos | Cualquiera |
| **Diferencial** | Cambios desde el último Full | Cualquiera |
| **Transaction Log** | Cambios en el log desde el último log backup | Full o Bulk-Logged |
| **File/Filegroup** | Archivos individuales de la BD | Full o Bulk-Logged |
| **Copy-Only** | Full o log fuera de la cadena de backups | Cualquiera |

---

## 1. Backup Full

Respalda **toda la base de datos** incluyendo parte del transaction log necesario para dejarlo consistente.

```sql
BACKUP DATABASE Ventas
TO DISK = 'C:\Backups\Ventas_Full.bak'
WITH FORMAT,           -- sobreescribe el medio
     NAME = 'Backup Full - Ventas',
     STATS = 10;       -- muestra progreso cada 10%
```

### Opciones comunes

```sql
BACKUP DATABASE Ventas
TO DISK = 'C:\Backups\Ventas_Full.bak'
WITH FORMAT,
     COMPRESSION,      -- comprime el archivo
     CHECKSUM,         -- verifica integridad
     NAME = 'Ventas Full',
     DESCRIPTION = 'Backup completo base Ventas',
     STATS = 5;
```

---

## 2. Backup Diferencial

Respalda todos los **cambios desde el último backup full**. Más rápido que un full, más lento que un log backup.

```sql
BACKUP DATABASE Ventas
TO DISK = 'C:\Backups\Ventas_Diff.bak'
WITH DIFFERENTIAL,
     FORMAT,
     NAME = 'Backup Diferencial - Ventas',
     STATS = 10;
```

> Requiere que exista un backup full previo. Para restaurar necesitás: Full + el último Diferencial.

---

## 3. Backup de Transaction Log

Respalda el **log de transacciones** y lo trunca (libera espacio). Solo disponible con recovery model Full o Bulk-Logged.

```sql
BACKUP LOG Ventas
TO DISK = 'C:\Backups\Ventas_Log.bak'
WITH FORMAT,
     NAME = 'Backup Log - Ventas',
     STATS = 10;
```

> Para restaurar a un punto exacto necesitás: Full + (último Diferencial opcional) + todos los Log backups en orden.

---

## 4. Backup Copy-Only

Hace un backup full o log **sin romper la cadena de backups**. Útil para hacer una copia puntual sin afectar la estrategia de backup regular.

```sql
BACKUP DATABASE Ventas
TO DISK = 'C:\Backups\Ventas_CopyOnly.bak'
WITH COPY_ONLY,
     FORMAT,
     NAME = 'Copy-Only para pruebas';
```

---

## Hacer backup a múltiples archivos

SQL Server puede dividir el backup en varios archivos (striping) para mayor velocidad.

```sql
BACKUP DATABASE Ventas
TO DISK = 'C:\Backups\Ventas_1.bak',
   DISK = 'C:\Backups\Ventas_2.bak',
   DISK = 'C:\Backups\Ventas_3.bak'
WITH FORMAT, STATS = 10;
```

---

## Verificar un backup (RESTORE VERIFYONLY)

Comprueba que el backup es válido y legible sin restaurar nada.

```sql
RESTORE VERIFYONLY
FROM DISK = 'C:\Backups\Ventas_Full.bak';
```

---

## Ver el contenido de un backup

```sql
-- Info general del backup
RESTORE HEADERONLY
FROM DISK = 'C:\Backups\Ventas_Full.bak';

-- Info de archivos lógicos dentro del backup
RESTORE FILELISTONLY
FROM DISK = 'C:\Backups\Ventas_Full.bak';

-- Info del medio físico
RESTORE LABELONLY
FROM DISK = 'C:\Backups\Ventas_Full.bak';
```

---

## Historial de backups

```sql
-- Últimos backups realizados
SELECT bs.database_name,
       bs.backup_start_date,
       bs.backup_finish_date,
       bs.type,                          -- D=Full, I=Diferencial, L=Log
       CAST(bs.backup_size / 1048576.0 AS DECIMAL(10,2)) AS SizeMB,
       bmf.physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'Ventas'
ORDER BY bs.backup_start_date DESC;
```

---

## Estrategias de backup comunes

### Estrategia básica (Simple recovery)
```
Domingo:    Full backup
Lun - Sáb: Diferencial diario
```

### Estrategia completa (Full recovery)
```
Domingo:    Full backup
Lun - Sáb: Diferencial diario
Cada hora:  Transaction log backup
```

### Estrategia de alta disponibilidad
```
Cada día:      Full backup
Cada 4 horas:  Diferencial
Cada 15 min:   Transaction log backup
```

---

## Cadena de backups

```
[Full]──────────────────────────────────────────
        [Log1][Log2][Log3][Log4][Log5][Log6]...

Para recuperar al momento de Log4:
  → Restaurar Full (WITH NORECOVERY)
  → Restaurar Log1 (WITH NORECOVERY)
  → Restaurar Log2 (WITH NORECOVERY)
  → Restaurar Log3 (WITH NORECOVERY)
  → Restaurar Log4 (WITH RECOVERY o STOPAT)

Con diferencial:
[Full]──────[Diff]──────────────────────────────
                    [Log1][Log2][Log3]...

Para recuperar al momento de Log2:
  → Restaurar Full  (WITH NORECOVERY)
  → Restaurar Diff  (WITH NORECOVERY)
  → Restaurar Log1  (WITH NORECOVERY)
  → Restaurar Log2  (WITH RECOVERY)
```

---

## Buenas prácticas

- Guardar los backups en un **disco o ubicación diferente** a donde están los datos
- Usar `WITH CHECKSUM` para detectar corrupción
- Automatizar con **SQL Server Agent** (ver `05_jobs_y_mantenimiento.md`)
- Probar regularmente que los backups se pueden restaurar
- Usar `COMPRESSION` para ahorrar espacio (requiere edición Standard o superior)
- Mantener un historial en `msdb.dbo.backupset` y monitorearlo
