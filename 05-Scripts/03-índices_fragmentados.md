# Índices Fragmentados - Detección y Reparación

Script para detectar índices fragmentados y aplicar `REORGANIZE` o `REBUILD` automáticamente según el porcentaje de fragmentación.

---

## Configuración

| Variable | Descripción | Default |
|---|---|---|
| `@BaseDatos` | Base de datos a analizar | base actual |
| `@UmbralReorganize` | % mínimo para reorganizar | `10.0` |
| `@UmbralRebuild` | % mínimo para reconstruir | `30.0` |
| `@MinPaginas` | Ignorar índices con menos páginas | `100` |
| `@ModoEjecucion` | `0`=solo reporte, `1`=ejecutar cambios | `0` |

> Siempre ejecutar primero con `@ModoEjecucion = 0` para revisar el reporte antes de aplicar cambios.

---

## Paso 1: Reporte de fragmentación actual

```sql
DECLARE @BaseDatos          NVARCHAR(128) = DB_NAME();
DECLARE @UmbralReorganize   FLOAT         = 10.0;
DECLARE @UmbralRebuild      FLOAT         = 30.0;
DECLARE @MinPaginas         INT           = 100;

SELECT
    OBJECT_NAME(ips.object_id)              AS tabla,
    i.name                                  AS indice,
    i.type_desc                             AS tipo,
    CAST(ips.avg_fragmentation_in_percent AS DECIMAL(5,2)) AS fragmentacion_pct,
    ips.page_count                          AS paginas,
    CASE
        WHEN ips.avg_fragmentation_in_percent >= @UmbralRebuild    THEN 'REBUILD'
        WHEN ips.avg_fragmentation_in_percent >= @UmbralReorganize THEN 'REORGANIZE'
        ELSE 'OK'
    END                                     AS accion_recomendada
FROM sys.dm_db_index_physical_stats(
    DB_ID(@BaseDatos), NULL, NULL, NULL, 'LIMITED'
) ips
JOIN sys.indexes i
    ON ips.object_id = i.object_id
    AND ips.index_id = i.index_id
WHERE ips.page_count >= @MinPaginas
  AND ips.index_id > 0
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

---

## Paso 2: Reparación automática

Aplicar `REORGANIZE` o `REBUILD` según el umbral configurado. Cambiar `@ModoEjecucion = 1` para ejecutar.

```sql
DECLARE @BaseDatos          NVARCHAR(128) = DB_NAME();
DECLARE @UmbralReorganize   FLOAT         = 10.0;
DECLARE @UmbralRebuild      FLOAT         = 30.0;
DECLARE @MinPaginas         INT           = 100;
DECLARE @ModoEjecucion      BIT           = 0;   -- cambiar a 1 para ejecutar

DECLARE @Tabla      NVARCHAR(256);
DECLARE @Indice     NVARCHAR(256);
DECLARE @Frag       FLOAT;
DECLARE @Paginas    INT;
DECLARE @Accion     NVARCHAR(20);
DECLARE @SQL        NVARCHAR(MAX);
DECLARE @Contador   INT = 0;

SELECT
    OBJECT_NAME(ips.object_id)  AS tabla,
    i.name                      AS indice,
    ips.avg_fragmentation_in_percent AS fragmentacion,
    ips.page_count              AS paginas,
    CASE
        WHEN ips.avg_fragmentation_in_percent >= @UmbralRebuild    THEN 'REBUILD'
        WHEN ips.avg_fragmentation_in_percent >= @UmbralReorganize THEN 'REORGANIZE'
    END AS accion
INTO #IndicesProcesar
FROM sys.dm_db_index_physical_stats(
    DB_ID(@BaseDatos), NULL, NULL, NULL, 'LIMITED'
) ips
JOIN sys.indexes i
    ON ips.object_id = i.object_id
    AND ips.index_id = i.index_id
WHERE ips.page_count >= @MinPaginas
  AND ips.index_id > 0
  AND ips.avg_fragmentation_in_percent >= @UmbralReorganize
ORDER BY ips.avg_fragmentation_in_percent DESC;

IF @ModoEjecucion = 0
BEGIN
    PRINT 'MODO SOLO REPORTE. Cambiar @ModoEjecucion = 1 para ejecutar.';
    SELECT * FROM #IndicesProcesar;
    DROP TABLE #IndicesProcesar;
    RETURN;
END

DECLARE cur CURSOR FOR
    SELECT tabla, indice, fragmentacion, paginas, accion FROM #IndicesProcesar;

OPEN cur;
FETCH NEXT FROM cur INTO @Tabla, @Indice, @Frag, @Paginas, @Accion;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @Contador = @Contador + 1;
    PRINT CAST(@Contador AS NVARCHAR(5)) + '. [' + @Tabla + '].[' + @Indice + '] — '
        + CAST(CAST(@Frag AS DECIMAL(5,2)) AS NVARCHAR(10)) + '% — ' + @Accion;

    BEGIN TRY
        IF @Accion = 'REBUILD'
            SET @SQL = 'ALTER INDEX [' + @Indice + '] ON [' + @Tabla + '] REBUILD;';
        ELSE
            SET @SQL = 'ALTER INDEX [' + @Indice + '] ON [' + @Tabla + '] REORGANIZE;';

        EXEC sp_executesql @SQL;
        PRINT '   OK';
    END TRY
    BEGIN CATCH
        PRINT '   ERROR: ' + ERROR_MESSAGE();
    END CATCH

    FETCH NEXT FROM cur INTO @Tabla, @Indice, @Frag, @Paginas, @Accion;
END

CLOSE cur; DEALLOCATE cur;
DROP TABLE IF EXISTS #IndicesProcesar;
PRINT 'Índices procesados: ' + CAST(@Contador AS NVARCHAR(5));
```

---

## Extras útiles

### Ver fragmentación de una tabla específica

```sql
SELECT i.name AS indice,
       ips.avg_fragmentation_in_percent,
       ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('Ventas'), NULL, NULL, 'DETAILED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

### Reconstruir manualmente un índice específico

```sql
ALTER INDEX IX_Clientes_Email ON Clientes REBUILD;
ALTER INDEX ALL ON Clientes REBUILD;          -- todos los índices de la tabla
ALTER INDEX ALL ON Clientes REORGANIZE;       -- reorganizar (más liviano)
```

### Actualizar estadísticas después del mantenimiento

```sql
EXEC sp_updatestats;                          -- toda la base
UPDATE STATISTICS Clientes WITH FULLSCAN;     -- tabla específica
```

---

## Referencia de umbrales

| Fragmentación | Acción recomendada |
|---|---|
| < 10% | No hacer nada |
| 10% - 30% | `REORGANIZE` |
| > 30% | `REBUILD` |
