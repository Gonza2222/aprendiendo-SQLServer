# Backup Automático Configurable

Script para realizar backup full, diferencial o de log con variables configurables. Listo para usar directamente o programar como paso de un SQL Server Agent Job.

---

## Configuración

Modificar estas variables antes de ejecutar:

| Variable | Descripción | Valores posibles |
|---|---|---|
| `@BaseDatos` | Nombre de la base a respaldar | cualquier base existente |
| `@TipoBackup` | Tipo de backup | `F`=Full, `D`=Diferencial, `L`=Log |
| `@RutaBackup` | Carpeta destino del archivo | ruta con `\` al final |
| `@Compresion` | Activar compresión | `1`=sí, `0`=no |
| `@Checksum` | Verificar integridad | `1`=sí, `0`=no |
| `@Stats` | Mostrar progreso cada N% | número entero |

---

## Script

```sql
-- ============================================================
-- CONFIGURACIÓN (modificar según necesidad)
-- ============================================================

DECLARE @BaseDatos      NVARCHAR(128) = 'Ventas';
DECLARE @TipoBackup     CHAR(1)       = 'F';           -- F=Full, D=Diferencial, L=Log
DECLARE @RutaBackup     NVARCHAR(256) = 'C:\Backups\';
DECLARE @Compresion     BIT           = 1;
DECLARE @Checksum       BIT           = 1;
DECLARE @Stats          INT           = 10;

-- ============================================================
-- NO MODIFICAR A PARTIR DE ACÁ
-- ============================================================

DECLARE @NombreArchivo  NVARCHAR(512);
DECLARE @NombreBackup   NVARCHAR(256);
DECLARE @Timestamp      NVARCHAR(20);
DECLARE @SQL            NVARCHAR(MAX);
DECLARE @TipoDesc       NVARCHAR(20);

SET @Timestamp = CONVERT(NVARCHAR(20), GETDATE(), 112)
               + '_'
               + REPLACE(CONVERT(NVARCHAR(8), GETDATE(), 108), ':', '');

SELECT @TipoDesc = CASE @TipoBackup
    WHEN 'F' THEN 'Full'
    WHEN 'D' THEN 'Diferencial'
    WHEN 'L' THEN 'Log'
    ELSE 'Desconocido'
END;

SET @NombreArchivo = @RutaBackup + @BaseDatos + '_' + @TipoDesc + '_' + @Timestamp + '.bak';
SET @NombreBackup  = @BaseDatos + ' - Backup ' + @TipoDesc + ' - ' + @Timestamp;

-- Verificar que la base existe
IF NOT EXISTS (SELECT 1 FROM sys.databases WHERE name = @BaseDatos)
BEGIN
    RAISERROR('ERROR: La base de datos "%s" no existe en esta instancia.', 16, 1, @BaseDatos);
    RETURN;
END

-- Verificar recovery model para backup de log
IF @TipoBackup = 'L'
BEGIN
    DECLARE @RecoveryModel NVARCHAR(20);
    SELECT @RecoveryModel = recovery_model_desc
    FROM sys.databases WHERE name = @BaseDatos;

    IF @RecoveryModel = 'SIMPLE'
    BEGIN
        RAISERROR('ERROR: La base "%s" usa recovery model SIMPLE. No se puede hacer backup de log.', 16, 1, @BaseDatos);
        RETURN;
    END
END

-- Construir el comando
IF @TipoBackup IN ('F', 'D')
BEGIN
    SET @SQL = 'BACKUP DATABASE [' + @BaseDatos + ']
TO DISK = ''' + @NombreArchivo + '''
WITH FORMAT,
     NAME = ''' + @NombreBackup + ''',
     STATS = ' + CAST(@Stats AS NVARCHAR(3));

    IF @TipoBackup = 'D'  SET @SQL = @SQL + ', DIFFERENTIAL';
    IF @Compresion = 1     SET @SQL = @SQL + ', COMPRESSION';
    IF @Checksum = 1       SET @SQL = @SQL + ', CHECKSUM';
END
ELSE IF @TipoBackup = 'L'
BEGIN
    SET @SQL = 'BACKUP LOG [' + @BaseDatos + ']
TO DISK = ''' + @NombreArchivo + '''
WITH FORMAT,
     NAME = ''' + @NombreBackup + ''',
     STATS = ' + CAST(@Stats AS NVARCHAR(3));

    IF @Compresion = 1 SET @SQL = @SQL + ', COMPRESSION';
    IF @Checksum = 1   SET @SQL = @SQL + ', CHECKSUM';
END

PRINT '============================================================';
PRINT 'Iniciando backup...';
PRINT '  Base de datos : ' + @BaseDatos;
PRINT '  Tipo          : ' + @TipoDesc;
PRINT '  Destino       : ' + @NombreArchivo;
PRINT '  Compresión    : ' + CASE @Compresion WHEN 1 THEN 'Sí' ELSE 'No' END;
PRINT '  Checksum      : ' + CASE @Checksum   WHEN 1 THEN 'Sí' ELSE 'No' END;
PRINT '  Inicio        : ' + CONVERT(NVARCHAR(20), GETDATE(), 120);
PRINT '============================================================';

BEGIN TRY
    EXEC sp_executesql @SQL;
    PRINT 'Backup completado exitosamente.';
    PRINT 'Fin: ' + CONVERT(NVARCHAR(20), GETDATE(), 120);
END TRY
BEGIN CATCH
    PRINT 'ERROR durante el backup: ' + ERROR_MESSAGE();
    RAISERROR('Backup fallido.', 16, 1);
END CATCH
```

---

## Verificar el backup generado

```sql
RESTORE VERIFYONLY
FROM DISK = 'C:\Backups\Ventas_Full_20241115_230000.bak';
```

---

## Hacer los tres tipos en secuencia

```sql
-- Full
BACKUP DATABASE [Ventas] TO DISK = 'C:\Backups\Ventas_Full.bak'
WITH FORMAT, COMPRESSION, CHECKSUM, NAME = 'Full', STATS = 10;

-- Diferencial
BACKUP DATABASE [Ventas] TO DISK = 'C:\Backups\Ventas_Diff.bak'
WITH DIFFERENTIAL, FORMAT, COMPRESSION, CHECKSUM, NAME = 'Diff', STATS = 10;

-- Log
BACKUP LOG [Ventas] TO DISK = 'C:\Backups\Ventas_Log.bak'
WITH FORMAT, COMPRESSION, CHECKSUM, NAME = 'Log', STATS = 10;
```

---

## Ver historial de backups

```sql
SELECT bs.database_name,
       bs.backup_start_date,
       bs.backup_finish_date,
       CASE bs.type
           WHEN 'D' THEN 'Full'
           WHEN 'I' THEN 'Diferencial'
           WHEN 'L' THEN 'Log'
       END AS tipo,
       CAST(bs.backup_size / 1048576.0 AS DECIMAL(10,2)) AS size_MB,
       bmf.physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'Ventas'
ORDER BY bs.backup_start_date DESC;
```
