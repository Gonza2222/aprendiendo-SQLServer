# Triggers

## ¿Qué es un trigger?

Un trigger es un **procedimiento que se ejecuta automáticamente** en respuesta a ciertos eventos sobre una tabla o la base de datos. No se llama explícitamente, sino que se dispara solo.

---

## Tipos de triggers

| Tipo | Se dispara ante |
|---|---|
| DML Trigger | `INSERT`, `UPDATE`, `DELETE` sobre tablas/vistas |
| DDL Trigger | `CREATE`, `ALTER`, `DROP` sobre objetos de la BD |
| Logon Trigger | Cuando un usuario inicia sesión |

---

## Tablas especiales: INSERTED y DELETED

Dentro de un trigger DML existen dos tablas virtuales con la estructura de la tabla afectada:

| Tabla | Disponible en | Contiene |
|---|---|---|
| `INSERTED` | INSERT, UPDATE | Las filas nuevas / modificadas |
| `DELETED` | DELETE, UPDATE | Las filas anteriores / eliminadas |

---

## 1. Triggers DML

### Sintaxis base

```sql
CREATE TRIGGER nombre_trigger
ON nombre_tabla
AFTER INSERT | UPDATE | DELETE  -- o FOR (son equivalentes)
AS
BEGIN
    -- lógica usando INSERTED y/o DELETED
END;
```

### Trigger AFTER INSERT — registrar auditoría

```sql
CREATE TRIGGER trg_AuditarInsertCliente
ON Clientes
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Auditoria (Tabla, Accion, Fecha)
    SELECT 'Clientes', 'INSERT', GETDATE()
    FROM INSERTED;
END;
```

### Trigger AFTER DELETE — registrar eliminaciones

```sql
CREATE TRIGGER trg_AuditarDeleteCliente
ON Clientes
AFTER DELETE
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO ClientesEliminados (ClienteID, Nombre, FechaEliminacion)
    SELECT ClienteID, Nombre, GETDATE()
    FROM DELETED;
END;
```

### Trigger AFTER UPDATE — detectar cambios

```sql
CREATE TRIGGER trg_AuditarUpdatePrecio
ON Productos
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    IF UPDATE(Precio)  -- solo si cambió la columna Precio
    BEGIN
        INSERT INTO AuditoriaPrecio (ProductoID, PrecioAnterior, PrecioNuevo, Fecha)
        SELECT d.ProductoID, d.Precio, i.Precio, GETDATE()
        FROM DELETED d
        JOIN INSERTED i ON d.ProductoID = i.ProductoID;
    END;
END;
```

### Trigger INSTEAD OF — interceptar la operación

Se ejecuta **en lugar de** la operación original. Útil en vistas o para validaciones complejas.

```sql
CREATE TRIGGER trg_NoEliminarCliente
ON Clientes
INSTEAD OF DELETE
AS
BEGIN
    -- En vez de eliminar, marcamos como inactivo
    UPDATE Clientes
    SET Activo = 0
    FROM Clientes c
    JOIN DELETED d ON c.ClienteID = d.ClienteID;
    PRINT 'Clientes marcados como inactivos (no eliminados).';
END;
```

---

## 2. Triggers DDL

Se disparan ante cambios en la estructura de la base de datos.

```sql
CREATE TRIGGER trg_ImpedirDropTable
ON DATABASE
FOR DROP_TABLE
AS
BEGIN
    PRINT 'No se permite eliminar tablas en esta base de datos.';
    ROLLBACK;
END;
```

```sql
-- Auditar creación de tablas
CREATE TRIGGER trg_LogCrearTabla
ON DATABASE
FOR CREATE_TABLE
AS
BEGIN
    INSERT INTO AuditoriaEsquema (Evento, Fecha, Usuario)
    VALUES ('CREATE_TABLE', GETDATE(), SUSER_NAME());
END;
```

---

## Modificar un trigger

```sql
ALTER TRIGGER trg_AuditarInsertCliente
ON Clientes
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    -- nueva lógica
END;
```

---

## Deshabilitar / Habilitar triggers

```sql
-- Deshabilitar un trigger específico
DISABLE TRIGGER trg_AuditarInsertCliente ON Clientes;

-- Habilitar un trigger
ENABLE TRIGGER trg_AuditarInsertCliente ON Clientes;

-- Deshabilitar todos los triggers de una tabla
DISABLE TRIGGER ALL ON Clientes;

-- Deshabilitar trigger DDL
DISABLE TRIGGER trg_ImpedirDropTable ON DATABASE;
```

---

## Eliminar un trigger

```sql
DROP TRIGGER trg_AuditarInsertCliente;           -- DML
DROP TRIGGER trg_ImpedirDropTable ON DATABASE;   -- DDL
```

---

## Ver triggers existentes

```sql
-- Triggers DML
SELECT name, parent_id, OBJECT_NAME(parent_id) AS tabla, create_date
FROM sys.triggers
WHERE parent_class = 1;

-- Triggers DDL (a nivel BD)
SELECT name, create_date
FROM sys.triggers
WHERE parent_class = 0;

-- Ver código de un trigger
EXEC sp_helptext 'trg_AuditarInsertCliente';
```

---

## Consideraciones importantes

- Los triggers se ejecutan **una vez por sentencia**, no una vez por fila → `INSERTED` y `DELETED` pueden tener múltiples filas
- Si el trigger hace `ROLLBACK`, la operación original también se revierte
- Los triggers anidados están permitidos (un trigger puede disparar otro), pero hay que tener cuidado con los ciclos infinitos
- Usar `SET NOCOUNT ON` para evitar mensajes innecesarios

---

## Cuándo usar triggers

| Caso de uso | Ejemplo |
|---|---|
| Auditoría | Registrar quién modificó qué y cuándo |
| Integridad de negocio | Validaciones complejas que no se pueden hacer con constraints |
| Sincronización | Mantener tablas de resumen actualizadas |
| Protección | Impedir operaciones peligrosas (DROP, DELETE masivo) |

---

## Cuándo NO usar triggers

- Para lógica de negocio que puede ir en la aplicación → dificulta el mantenimiento
- En tablas de alto volumen → pueden degradar el rendimiento
- Si hay alternativa con constraints o defaults
