# ✏️ CRUD

CRUD son las cuatro operaciones básicas sobre los datos: **Create, Read, Update, Delete**.

---

## INSERT — Insertar datos

### Una sola fila
```sql
INSERT INTO Clientes (nombre, email, telefono)
VALUES ('Juan Pérez', 'juan@email.com', '351-1234567');
```

### Varias filas a la vez
```sql
INSERT INTO Clientes (nombre, email, telefono)
VALUES
    ('Ana García', 'ana@email.com', '351-1111111'),
    ('Carlos López', 'carlos@email.com', '351-2222222'),
    ('María Torres', 'maria@email.com', '351-3333333');
```

### Insertar desde otra tabla
```sql
INSERT INTO ClientesArchivo (nombre, email)
SELECT nombre, email
FROM Clientes
WHERE fecha_alt < '2023-01-01';
```

---

## SELECT — Consultar datos

### Todos los registros
```sql
SELECT * FROM Clientes;
```

### Columnas específicas
```sql
SELECT nombre, email FROM Clientes;
```

### Con alias
```sql
SELECT nombre AS 'Nombre completo', email AS 'Correo'
FROM Clientes;
```

### Sin duplicados
```sql
SELECT DISTINCT ciudad FROM Clientes;
```

### Top N registros
```sql
SELECT TOP 10 * FROM Clientes;

-- Top por porcentaje
SELECT TOP 10 PERCENT * FROM Clientes;
```

---

## UPDATE — Modificar datos

### Modificar un registro
```sql
UPDATE Clientes
SET email = 'nuevo@email.com'
WHERE id = 1;
```

### Modificar varios campos a la vez
```sql
UPDATE Clientes
SET email = 'nuevo@email.com',
    telefono = '351-9999999'
WHERE id = 1;
```

### Modificar varios registros
```sql
UPDATE Clientes
SET ciudad = 'Córdoba'
WHERE ciudad IS NULL;
```

> ⚠️ Siempre usar `WHERE` en un `UPDATE`. Sin él se modifican **todos** los registros.

---

## DELETE — Eliminar datos

### Eliminar un registro
```sql
DELETE FROM Clientes
WHERE id = 1;
```

### Eliminar varios registros
```sql
DELETE FROM Clientes
WHERE fecha_alt < '2020-01-01';
```

> ⚠️ Siempre usar `WHERE` en un `DELETE`. Sin él se eliminan **todos** los registros.

---

## TRUNCATE vs DELETE

| | `TRUNCATE` | `DELETE` |
|---|---|---|
| Elimina todas las filas | ✅ Sí | ✅ Sí (sin WHERE) |
| Permite WHERE | ❌ No | ✅ Sí |
| Resetea IDENTITY | ✅ Sí | ❌ No |
| Velocidad | Más rápido | Más lento |

```sql
-- Vacía la tabla completa y resetea el autoincremento
TRUNCATE TABLE Clientes;
```

---

## Funciones útiles al insertar/consultar

```sql
-- Fecha y hora actual
SELECT GETDATE();

-- ID del último registro insertado
SELECT SCOPE_IDENTITY();

-- Verificar si un valor es NULL
SELECT ISNULL(telefono, 'Sin teléfono') FROM Clientes;
```
