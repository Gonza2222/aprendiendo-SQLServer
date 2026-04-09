# Funciones (Functions)

## Tipos de funciones en SQL Server

| Tipo | Devuelve | Se usa en SELECT |
|---|---|---|
| Escalar | Un valor único | ✅ |
| Tabla (inline) | Una tabla | ✅ |
| Tabla (multi-statement) | Una tabla | ✅ |

---

## 1. Funciones Escalares

Devuelven **un solo valor** de cualquier tipo de dato.

```sql
CREATE FUNCTION nombre_funcion (@parametro tipo)
RETURNS tipo_retorno
AS
BEGIN
    DECLARE @resultado tipo_retorno;
    -- lógica
    RETURN @resultado;
END;
```

### Ejemplo

```sql
CREATE FUNCTION fn_CalcularIVA (@Precio DECIMAL(10,2))
RETURNS DECIMAL(10,2)
AS
BEGIN
    RETURN @Precio * 1.21;
END;

-- Uso (siempre con esquema dbo.)
SELECT dbo.fn_CalcularIVA(1000) AS PrecioConIVA;

SELECT ProductoID, Nombre, dbo.fn_CalcularIVA(Precio) AS PrecioFinal
FROM Productos;
```

---

## 2. Funciones de Tabla Inline (TVF)

Devuelven una tabla. Contienen **una sola sentencia SELECT**. Son más eficientes que las multi-statement porque SQL Server las trata como una vista parametrizada.

```sql
CREATE FUNCTION fn_nombre (@param tipo)
RETURNS TABLE
AS
RETURN (
    SELECT columnas
    FROM tabla
    WHERE condicion = @param
);
```

### Ejemplo

```sql
CREATE FUNCTION fn_PedidosPorCliente (@ClienteID INT)
RETURNS TABLE
AS
RETURN (
    SELECT PedidoID, Fecha, Total
    FROM Pedidos
    WHERE ClienteID = @ClienteID
);

-- Uso
SELECT * FROM dbo.fn_PedidosPorCliente(3);

-- Se puede hacer JOIN con ellas
SELECT c.Nombre, p.PedidoID, p.Total
FROM Clientes c
CROSS APPLY dbo.fn_PedidosPorCliente(c.ClienteID) p;
```

---

## 3. Funciones de Tabla Multi-Statement (MSTVF)

Devuelven una tabla definida explícitamente. Permiten lógica más compleja (variables, bucles, condicionales).

```sql
CREATE FUNCTION fn_nombre (@param tipo)
RETURNS @resultado TABLE (
    col1 tipo,
    col2 tipo
)
AS
BEGIN
    INSERT INTO @resultado
    SELECT col1, col2 FROM tabla WHERE condicion = @param;

    -- más lógica si hace falta
    UPDATE @resultado SET col1 = col1 * 2 WHERE col2 > 100;

    RETURN;
END;
```

### Ejemplo

```sql
CREATE FUNCTION fn_ResumenCliente (@ClienteID INT)
RETURNS @resumen TABLE (
    Nombre      VARCHAR(100),
    TotalGastado DECIMAL(10,2),
    CantPedidos INT
)
AS
BEGIN
    INSERT INTO @resumen
    SELECT c.Nombre, SUM(p.Total), COUNT(p.PedidoID)
    FROM Clientes c
    JOIN Pedidos p ON c.ClienteID = p.ClienteID
    WHERE c.ClienteID = @ClienteID
    GROUP BY c.Nombre;

    RETURN;
END;

-- Uso
SELECT * FROM dbo.fn_ResumenCliente(5);
```

---

## Modificar una función

```sql
ALTER FUNCTION fn_CalcularIVA (@Precio DECIMAL(10,2))
RETURNS DECIMAL(10,2)
AS
BEGIN
    RETURN @Precio * 1.105;  -- cambio de tasa
END;
```

---

## Eliminar una función

```sql
DROP FUNCTION fn_CalcularIVA;
DROP FUNCTION fn_PedidosPorCliente;
```

---

## Ver funciones existentes

```sql
-- Listar funciones de usuario
SELECT name, type_desc, create_date
FROM sys.objects
WHERE type IN ('FN', 'IF', 'TF');  -- FN=escalar, IF=inline, TF=multi-statement

-- Ver código
EXEC sp_helptext 'fn_CalcularIVA';
```

---

## APPLY: usar funciones de tabla por fila

`CROSS APPLY` y `OUTER APPLY` permiten llamar a una función por cada fila de otra tabla.

```sql
-- CROSS APPLY: solo filas donde la función devuelve resultados
SELECT c.Nombre, p.PedidoID
FROM Clientes c
CROSS APPLY dbo.fn_PedidosPorCliente(c.ClienteID) p;

-- OUTER APPLY: incluye filas aunque la función no devuelva nada (como LEFT JOIN)
SELECT c.Nombre, p.PedidoID
FROM Clientes c
OUTER APPLY dbo.fn_PedidosPorCliente(c.ClienteID) p;
```

---

## Diferencias clave: Función vs. Stored Procedure

| Característica | Función | Stored Procedure |
|---|---|---|
| Devuelve valor | Obligatorio | Opcional |
| Se usa en SELECT | ✅ | ❌ |
| Puede modificar datos | ❌ (solo lectura) | ✅ |
| Manejo de transacciones | ❌ | ✅ |
| TRY/CATCH | ❌ | ✅ |

---

## Buenas prácticas

- Preferir funciones **inline** sobre multi-statement cuando sea posible (mejor performance)
- Siempre llamar funciones con el esquema: `dbo.fn_nombre()`
- Evitar funciones escalares en columnas de tablas grandes → pueden ser lentas (se ejecutan fila por fila)
