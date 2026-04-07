# Procedimientos Almacenados (Stored Procedures)

## ¿Qué es un stored procedure?

Un procedimiento almacenado es un **bloque de código T-SQL guardado en la base de datos** que puede ejecutarse por nombre. Puede recibir parámetros, devolver resultados y ejecutar lógica compleja.

---

## Crear un procedimiento

```sql
CREATE PROCEDURE nombre_procedure
AS
BEGIN
    -- código T-SQL
END;
```

### Ejemplo básico

```sql
CREATE PROCEDURE sp_ListarClientes
AS
BEGIN
    SELECT ClienteID, Nombre, Email
    FROM Clientes
    ORDER BY Nombre;
END;
```

---

## Ejecutar un procedimiento

```sql
EXEC sp_ListarClientes;
-- o
EXECUTE sp_ListarClientes;
```

---

## Procedimientos con parámetros

### Parámetros de entrada

```sql
CREATE PROCEDURE sp_BuscarCliente
    @ClienteID INT
AS
BEGIN
    SELECT * FROM Clientes
    WHERE ClienteID = @ClienteID;
END;

-- Ejecución
EXEC sp_BuscarCliente @ClienteID = 5;
EXEC sp_BuscarCliente 5;  -- también válido (por posición)
```

### Parámetros con valor por defecto

```sql
CREATE PROCEDURE sp_ListarPedidos
    @Estado VARCHAR(20) = 'Pendiente'
AS
BEGIN
    SELECT * FROM Pedidos
    WHERE Estado = @Estado;
END;

-- Ejecución
EXEC sp_ListarPedidos;                        -- usa 'Pendiente'
EXEC sp_ListarPedidos @Estado = 'Entregado';  -- sobreescribe el default
```

### Parámetros de salida (OUTPUT)

```sql
CREATE PROCEDURE sp_ContarClientes
    @Total INT OUTPUT
AS
BEGIN
    SELECT @Total = COUNT(*) FROM Clientes;
END;

-- Ejecución
DECLARE @Resultado INT;
EXEC sp_ContarClientes @Total = @Resultado OUTPUT;
SELECT @Resultado AS TotalClientes;
```

---

## Modificar un procedimiento

```sql
ALTER PROCEDURE sp_BuscarCliente
    @ClienteID INT
AS
BEGIN
    SELECT ClienteID, Nombre, Email, Telefono
    FROM Clientes
    WHERE ClienteID = @ClienteID;
END;
```

---

## Eliminar un procedimiento

```sql
DROP PROCEDURE sp_BuscarCliente;
```

---

## Manejo de errores con TRY/CATCH

```sql
CREATE PROCEDURE sp_InsertarCliente
    @Nombre VARCHAR(100),
    @Email  VARCHAR(100)
AS
BEGIN
    BEGIN TRY
        INSERT INTO Clientes (Nombre, Email)
        VALUES (@Nombre, @Email);
        PRINT 'Cliente insertado correctamente.';
    END TRY
    BEGIN CATCH
        PRINT 'Error: ' + ERROR_MESSAGE();
        -- También se puede registrar en una tabla de logs
    END CATCH
END;
```

---

## Uso de transacciones dentro de un SP

```sql
CREATE PROCEDURE sp_TransferirSaldo
    @CuentaOrigen INT,
    @CuentaDestino INT,
    @Monto DECIMAL(10,2)
AS
BEGIN
    BEGIN TRANSACTION;
    BEGIN TRY
        UPDATE Cuentas SET Saldo = Saldo - @Monto WHERE CuentaID = @CuentaOrigen;
        UPDATE Cuentas SET Saldo = Saldo + @Monto WHERE CuentaID = @CuentaDestino;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        PRINT 'Transacción revertida: ' + ERROR_MESSAGE();
    END CATCH
END;
```

---

## Recompilación con WITH RECOMPILE

Hace que el SP genere un nuevo plan de ejecución cada vez que se llama. Útil cuando los datos cambian mucho.

```sql
CREATE PROCEDURE sp_ReporteVentas
WITH RECOMPILE
AS
BEGIN
    SELECT * FROM Ventas ORDER BY Fecha DESC;
END;
```

---

## Ver procedimientos existentes

```sql
-- Listar todos los SPs de usuario
SELECT name, create_date, modify_date
FROM sys.procedures;

-- Ver el código de un SP
EXEC sp_helptext 'sp_BuscarCliente';

-- O desde catálogos
SELECT definition
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('sp_BuscarCliente');
```

---

## Procedimientos del sistema útiles

| Procedimiento | Para qué sirve |
|---|---|
| `sp_helpdb` | Info de bases de datos |
| `sp_help 'tabla'` | Estructura de una tabla |
| `sp_helptext 'objeto'` | Código de un objeto |
| `sp_who` / `sp_who2` | Conexiones activas |
| `sp_rename` | Renombrar objetos |

---

## Buenas prácticas

- Prefijo `sp_` está reservado para SPs del sistema → usá `usp_` o un prefijo propio
- Siempre usar `SET NOCOUNT ON` al inicio para evitar mensajes de "N rows affected"
- Documentar parámetros con comentarios
- Manejar errores con TRY/CATCH

```sql
CREATE PROCEDURE usp_MiProcedimiento
AS
BEGIN
    SET NOCOUNT ON;
    -- lógica acá
END;
```

---

## Ventajas vs. consultas ad-hoc

| Ventaja | Descripción |
|---|---|
| Rendimiento | El plan de ejecución se cachea |
| Seguridad | Se da permiso al SP, no a la tabla |
| Mantenimiento | Lógica centralizada en la BD |
| Reutilización | Se llama desde cualquier app o script |
