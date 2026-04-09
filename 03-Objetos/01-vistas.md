# Vistas (Views)

## ¿Qué es una vista?

Una vista es una **consulta almacenada** que se comporta como una tabla virtual. No almacena datos por sí misma (salvo las indexadas), sino que ejecuta la consulta subyacente cada vez que se la llama.

---

## Crear una vista

```sql
CREATE VIEW nombre_vista AS
SELECT columna1, columna2
FROM tabla
WHERE condicion;
```

### Ejemplo

```sql
CREATE VIEW vw_ClientesActivos AS
SELECT ClienteID, Nombre, Email
FROM Clientes
WHERE Activo = 1;
```

---

## Consultar una vista

```sql
SELECT * FROM vw_ClientesActivos;
SELECT Nombre FROM vw_ClientesActivos WHERE ClienteID = 5;
```

---

## Modificar una vista

```sql
ALTER VIEW vw_ClientesActivos AS
SELECT ClienteID, Nombre, Email, Telefono
FROM Clientes
WHERE Activo = 1;
```

---

## Eliminar una vista

```sql
DROP VIEW vw_ClientesActivos;
```

---

## Opciones importantes

### WITH SCHEMABINDING
Vincula la vista al esquema de las tablas base. Impide que se modifiquen o eliminen las columnas que usa la vista.

```sql
CREATE VIEW vw_Productos
WITH SCHEMABINDING AS
SELECT ProductoID, Nombre, Precio
FROM dbo.Productos;
```

### WITH CHECK OPTION
Garantiza que las filas insertadas o modificadas a través de la vista sigan cumpliendo la condición del WHERE.

```sql
CREATE VIEW vw_ProductosCaros
WITH CHECK OPTION AS
SELECT * FROM Productos
WHERE Precio > 1000;
-- Un INSERT a través de esta vista fallará si Precio <= 1000
```

---

## Vistas indexadas (Materialized Views)

A diferencia de las vistas normales, **almacenan físicamente los datos**. Requieren `WITH SCHEMABINDING` y un índice clustered único.

```sql
CREATE VIEW vw_ResumenVentas
WITH SCHEMABINDING AS
SELECT VendedorID, COUNT_BIG(*) AS CantVentas, SUM(Total) AS TotalVentas
FROM dbo.Ventas
GROUP BY VendedorID;

-- Crear el índice que la materializa
CREATE UNIQUE CLUSTERED INDEX IX_vw_ResumenVentas
ON vw_ResumenVentas(VendedorID);
```

---

## Vistas actualizables

Se puede hacer `INSERT`, `UPDATE` o `DELETE` a través de una vista **si**:
- No usa `DISTINCT`, `GROUP BY`, `HAVING`, `UNION`
- No usa funciones de agregación
- No tiene columnas calculadas
- Referencia una sola tabla base

```sql
UPDATE vw_ClientesActivos
SET Email = 'nuevo@mail.com'
WHERE ClienteID = 3;
```

---

## Ver las vistas existentes

```sql
-- Listar todas las vistas
SELECT name, create_date, modify_date
FROM sys.views;

-- Ver el código de una vista
EXEC sp_helptext 'vw_ClientesActivos';

-- O desde catálogos
SELECT definition
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('vw_ClientesActivos');
```

---

## Cuándo usar vistas

| Caso de uso | Por qué |
|---|---|
| Simplificar consultas complejas | Encapsulás JOINs o lógica repetida |
| Seguridad | Exponés solo las columnas necesarias |
| Compatibilidad | Mantenés la interfaz aunque cambie la tabla |
| Reportes | Preparás datos listos para consumir |

---

## Limitaciones

- No pueden tener parámetros (para eso usar funciones de tabla)
- No almacenan datos (salvo indexadas)
- Pueden afectar performance si la consulta base es pesada
