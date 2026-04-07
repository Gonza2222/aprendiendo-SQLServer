# Índices (Indexes)

## ¿Qué es un índice?

Un índice es una **estructura de datos auxiliar** que SQL Server mantiene para acelerar las búsquedas. Funciona como el índice de un libro: en lugar de recorrer todas las páginas, vas directo a la ubicación.

> Mejoran el rendimiento de las consultas a cambio de más espacio en disco y mayor costo en INSERT/UPDATE/DELETE.

---

## Tipos principales

| Tipo | Descripción |
|---|---|
| **Clustered** | Ordena físicamente los datos de la tabla. Solo puede haber uno por tabla. |
| **Non-Clustered** | Estructura separada con punteros a los datos. Puede haber muchos. |
| **Único (UNIQUE)** | Garantiza que no haya valores duplicados en la columna indexada. |
| **Filtrado** | Non-clustered sobre un subconjunto de filas (con WHERE). |
| **Columnstore** | Optimizado para consultas analíticas sobre grandes volúmenes. |

---

## 1. Índice Clustered

```sql
CREATE CLUSTERED INDEX IX_nombre
ON tabla (columna ASC);
```

> La PRIMARY KEY crea automáticamente un índice clustered (salvo que se especifique lo contrario).

```sql
-- Ejemplo explícito
CREATE CLUSTERED INDEX IX_Clientes_ID
ON Clientes (ClienteID ASC);
```

- Solo **uno por tabla**
- Los datos de la tabla se ordenan físicamente según este índice
- Ideal para columnas por las que se busca o ordena frecuentemente (IDs, fechas)

---

## 2. Índice Non-Clustered

```sql
CREATE NONCLUSTERED INDEX IX_nombre
ON tabla (columna1 ASC, columna2 DESC);
```

```sql
-- Ejemplo
CREATE NONCLUSTERED INDEX IX_Clientes_Email
ON Clientes (Email);
```

- Puede haber hasta **999 por tabla**
- Tiene su propia estructura con punteros a las filas de la tabla
- Ideal para columnas que aparecen en WHERE, JOIN o ORDER BY frecuentemente

---

## 3. Índice Único

```sql
CREATE UNIQUE INDEX IX_nombre
ON tabla (columna);
```

```sql
CREATE UNIQUE NONCLUSTERED INDEX IX_Clientes_EmailUnico
ON Clientes (Email);
```

- Garantiza unicidad (como un UNIQUE CONSTRAINT)
- Puede ser clustered o non-clustered

---

## 4. Índice con columnas incluidas (INCLUDE)

Agrega columnas extra al índice sin ordenarlas. Útil para cubrir una consulta completamente sin tocar la tabla base (índice cubriente).

```sql
CREATE NONCLUSTERED INDEX IX_Pedidos_ClienteID
ON Pedidos (ClienteID)
INCLUDE (Fecha, Total);
```

Si la consulta pide solo `ClienteID`, `Fecha` y `Total`, SQL Server puede resolverla solo con el índice → menos I/O.

---

## 5. Índice Filtrado

Solo indexa un subconjunto de filas definido por un WHERE.

```sql
CREATE NONCLUSTERED INDEX IX_Pedidos_Pendientes
ON Pedidos (Fecha)
WHERE Estado = 'Pendiente';
```

Ideal cuando solo consultás frecuentemente una parte de los datos.

---

## 6. Índice Columnstore

Almacena los datos por columna en lugar de por fila. Óptimo para queries analíticas (agregaciones sobre millones de filas).

```sql
-- Non-clustered columnstore (puede coexistir con row-store)
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_Ventas_Columnstore
ON Ventas (Fecha, VendedorID, Total);
```

---

## Modificar un índice

No existe `ALTER INDEX` para cambiar la definición. Hay que DROP + CREATE. Sí se puede reorganizar o reconstruir:

```sql
-- Reorganizar (desfragmentación ligera, online)
ALTER INDEX IX_Clientes_Email ON Clientes REORGANIZE;

-- Reconstruir (desfragmentación completa, más pesado)
ALTER INDEX IX_Clientes_Email ON Clientes REBUILD;

-- Reconstruir todos los índices de una tabla
ALTER INDEX ALL ON Clientes REBUILD;
```

---

## Deshabilitar / Habilitar un índice

```sql
ALTER INDEX IX_Clientes_Email ON Clientes DISABLE;
ALTER INDEX IX_Clientes_Email ON Clientes REBUILD;  -- para volver a habilitarlo
```

---

## Eliminar un índice

```sql
DROP INDEX IX_Clientes_Email ON Clientes;
```

---

## Ver índices existentes

```sql
-- Índices de una tabla
EXEC sp_helpindex 'Clientes';

-- Desde catálogos del sistema
SELECT i.name, i.type_desc, i.is_unique, i.is_primary_key,
       COL_NAME(ic.object_id, ic.column_id) AS columna
FROM sys.indexes i
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
WHERE i.object_id = OBJECT_ID('Clientes');
```

---

## Fragmentación de índices

Con el tiempo los índices se fragmentan y pierden eficiencia.

```sql
SELECT index_id,
       avg_fragmentation_in_percent,
       page_count
FROM sys.dm_db_index_physical_stats(
    DB_ID(),             -- base de datos actual
    OBJECT_ID('Ventas'), -- tabla
    NULL, NULL, 'LIMITED'
);
```

| Fragmentación | Acción recomendada |
|---|---|
| < 10% | No hacer nada |
| 10% - 30% | `REORGANIZE` |
| > 30% | `REBUILD` |

---

## Cuándo crear un índice

✅ Crear índice si:
- La columna aparece frecuentemente en `WHERE`, `JOIN ON`, `ORDER BY`
- La tabla tiene muchas filas y las queries son lentas
- La columna tiene alta selectividad (muchos valores distintos)

❌ No crear índice si:
- La tabla es pequeña (un full scan es más rápido)
- La columna tiene baja cardinalidad (ej: campo Sexo con M/F)
- La tabla recibe muchos INSERT/UPDATE/DELETE (los índices ralentizan escrituras)

---

## Resumen visual

```
Tabla sin índice:    [fila1][fila2][fila3]...[filaN]  → SQL escanea todo (Table Scan)

Clustered Index:     Los datos YA están ordenados físicamente por esa columna

Non-Clustered:       Estructura separada: [valor → puntero a fila]
                     SQL va al índice, obtiene la ubicación, luego busca en la tabla
```
