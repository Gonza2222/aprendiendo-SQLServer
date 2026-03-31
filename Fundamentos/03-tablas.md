# 📋 Tablas

## Crear una tabla

```sql
CREATE TABLE Clientes (
    id        INT PRIMARY KEY IDENTITY(1,1),
    nombre    VARCHAR(100) NOT NULL,
    email     VARCHAR(150) UNIQUE,
    telefono  VARCHAR(20),
    fecha_alt DATE DEFAULT GETDATE()
);
```

### Conceptos clave al crear tablas

| Concepto | Descripción |
|---|---|
| `PRIMARY KEY` | Identificador único de cada fila |
| `IDENTITY(1,1)` | Autoincremento: empieza en 1, incrementa de a 1 |
| `NOT NULL` | El campo no puede estar vacío |
| `UNIQUE` | No permite valores repetidos |
| `DEFAULT` | Valor por defecto si no se especifica uno |
| `FOREIGN KEY` | Relación con otra tabla |

---

## Ver tablas de la base actual

```sql
-- Nombres de todas las tablas
SELECT name FROM sys.tables;

-- Con más detalle
SELECT name, create_date, modify_date
FROM sys.tables
ORDER BY name;
```

---

## Ver columnas de una tabla

```sql
-- Opción 1
SP_HELP Clientes;

-- Opción 2
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Clientes';
```

---

## Modificar una tabla

### Agregar una columna
```sql
ALTER TABLE Clientes ADD direccion VARCHAR(200);
```

### Eliminar una columna
```sql
ALTER TABLE Clientes DROP COLUMN direccion;
```

### Modificar el tipo de dato de una columna
```sql
ALTER TABLE Clientes ALTER COLUMN telefono VARCHAR(30);
```

### Agregar una restricción
```sql
-- Clave foránea
ALTER TABLE Pedidos
ADD CONSTRAINT FK_Pedidos_Clientes
FOREIGN KEY (id_cliente) REFERENCES Clientes(id);

-- Valor por defecto
ALTER TABLE Clientes
ADD CONSTRAINT DF_fecha_alt DEFAULT GETDATE() FOR fecha_alt;
```

### Eliminar una restricción
```sql
ALTER TABLE Clientes DROP CONSTRAINT DF_fecha_alt;
```

---

## Renombrar una tabla

```sql
EXEC sp_rename 'Clientes', 'ClientesNuevo';
```

---

## Eliminar una tabla

```sql
DROP TABLE Clientes;
```

> ⚠️ Elimina la tabla y todos sus datos. Si tiene dependencias (claves foráneas) hay que eliminarlas primero.

### Eliminar solo si existe
```sql
IF OBJECT_ID('Clientes', 'U') IS NOT NULL
    DROP TABLE Clientes;
```

---

## Vaciar una tabla (sin eliminarla)

```sql
-- Elimina todas las filas y resetea el IDENTITY
TRUNCATE TABLE Clientes;

-- Elimina filas con condición (no resetea IDENTITY)
DELETE FROM Clientes WHERE id = 1;
```

| | `TRUNCATE` | `DELETE` |
|---|---|---|
| Velocidad | Más rápido | Más lento |
| Resetea IDENTITY | ✅ Sí | ❌ No |
| Permite WHERE | ❌ No | ✅ Sí |
| Se puede revertir | ✅ Sí (con transacción) | ✅ Sí |
