# 📊 Agrupamiento

## Funciones de agregado

Operan sobre un conjunto de filas y devuelven un único valor.

| Función | Descripción |
|---|---|
| `COUNT()` | Cuenta filas |
| `SUM()` | Suma valores |
| `AVG()` | Promedio |
| `MAX()` | Valor máximo |
| `MIN()` | Valor mínimo |

```sql
-- Total de clientes
SELECT COUNT(*) AS total_clientes FROM Clientes;

-- Total de ventas
SELECT SUM(total) AS total_ventas FROM Pedidos;

-- Ticket promedio
SELECT AVG(total) AS ticket_promedio FROM Pedidos;

-- Pedido más caro y más barato
SELECT MAX(total) AS mayor, MIN(total) AS menor FROM Pedidos;
```

---

## GROUP BY — Agrupar resultados

Agrupa filas que tienen el mismo valor en una o más columnas.

```sql
-- Cantidad de clientes por ciudad
SELECT ciudad, COUNT(*) AS cantidad
FROM Clientes
GROUP BY ciudad;

-- Total de ventas por cliente
SELECT id_cliente, SUM(total) AS total_vendido
FROM Pedidos
GROUP BY id_cliente;

-- Ventas por mes y año
SELECT YEAR(fecha) AS anio, MONTH(fecha) AS mes, SUM(total) AS total
FROM Pedidos
GROUP BY YEAR(fecha), MONTH(fecha)
ORDER BY anio, mes;
```

> ⚠️ Todo lo que va en el `SELECT` que no sea una función de agregado, **debe estar en el `GROUP BY`**.

---

## HAVING — Filtrar grupos

`HAVING` es el equivalente del `WHERE` pero para grupos.

```sql
-- Ciudades con más de 10 clientes
SELECT ciudad, COUNT(*) AS cantidad
FROM Clientes
GROUP BY ciudad
HAVING COUNT(*) > 10;

-- Clientes con más de $5000 en pedidos
SELECT id_cliente, SUM(total) AS total
FROM Pedidos
GROUP BY id_cliente
HAVING SUM(total) > 5000;
```

### WHERE vs HAVING

| | `WHERE` | `HAVING` |
|---|---|---|
| Filtra | Filas individuales | Grupos |
| Se ejecuta | Antes del GROUP BY | Después del GROUP BY |
| Puede usar funciones de agregado | ❌ No | ✅ Sí |

```sql
-- Combinando WHERE y HAVING
SELECT ciudad, COUNT(*) AS cantidad
FROM Clientes
WHERE fecha_alt >= '2023-01-01'   -- filtra filas primero
GROUP BY ciudad
HAVING COUNT(*) > 5;               -- filtra grupos después
```

---

## Orden de ejecución de una consulta

Aunque se escribe en este orden:
```sql
SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY
```

SQL Server lo ejecuta en este orden:
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```

---

## Ejemplos combinados

```sql
-- Top 3 productos más vendidos
SELECT TOP 3 id_producto, SUM(cantidad) AS total_vendido
FROM DetallePedidos
GROUP BY id_producto
ORDER BY total_vendido DESC;

-- Promedio de ventas por vendedor, solo los que superan el promedio general
SELECT id_vendedor, AVG(total) AS promedio
FROM Pedidos
GROUP BY id_vendedor
HAVING AVG(total) > (SELECT AVG(total) FROM Pedidos);
```
