# 🔄 Subconsultas

Una subconsulta es una consulta dentro de otra consulta. Se escribe entre paréntesis y se ejecuta primero.

---

## Subconsulta en WHERE

```sql
-- Clientes que hicieron algún pedido
SELECT nombre FROM Clientes
WHERE id IN (SELECT id_cliente FROM Pedidos);

-- Productos más caros que el promedio
SELECT nombre, precio FROM Productos
WHERE precio > (SELECT AVG(precio) FROM Productos);

-- Clientes que nunca hicieron un pedido
SELECT nombre FROM Clientes
WHERE id NOT IN (SELECT id_cliente FROM Pedidos);
```

---

## Subconsulta en SELECT

Devuelve un valor calculado por cada fila del resultado principal.

```sql
-- Para cada cliente, mostrar cuántos pedidos tiene
SELECT
    nombre,
    (SELECT COUNT(*) FROM Pedidos WHERE id_cliente = c.id) AS total_pedidos
FROM Clientes c;
```

---

## Subconsulta en FROM

Se usa como si fuera una tabla (se le debe dar un alias).

```sql
-- Promedio de ventas por cliente, y luego el promedio de esos promedios
SELECT AVG(total_por_cliente) AS promedio_general
FROM (
    SELECT id_cliente, SUM(total) AS total_por_cliente
    FROM Pedidos
    GROUP BY id_cliente
) AS resumen;
```

---

## EXISTS / NOT EXISTS

Verifica si la subconsulta devuelve al menos una fila. Más eficiente que `IN` en tablas grandes.

```sql
-- Clientes que tienen al menos un pedido
SELECT nombre FROM Clientes c
WHERE EXISTS (
    SELECT 1 FROM Pedidos p WHERE p.id_cliente = c.id
);

-- Clientes sin pedidos
SELECT nombre FROM Clientes c
WHERE NOT EXISTS (
    SELECT 1 FROM Pedidos p WHERE p.id_cliente = c.id
);
```

---

## ANY / ALL

```sql
-- Productos más caros que ALGÚN producto de la categoría 1
SELECT nombre, precio FROM Productos
WHERE precio > ANY (
    SELECT precio FROM Productos WHERE id_categoria = 1
);

-- Productos más caros que TODOS los productos de la categoría 1
SELECT nombre, precio FROM Productos
WHERE precio > ALL (
    SELECT precio FROM Productos WHERE id_categoria = 1
);
```

---

## Subconsulta vs JOIN

En general, lo que se puede resolver con subconsulta también se puede con JOIN y viceversa.

```sql
-- Con subconsulta
SELECT nombre FROM Clientes
WHERE id IN (SELECT id_cliente FROM Pedidos WHERE total > 1000);

-- Con JOIN (equivalente)
SELECT DISTINCT c.nombre
FROM Clientes c
INNER JOIN Pedidos p ON c.id = p.id_cliente
WHERE p.total > 1000;
```

> En tablas grandes, el JOIN suele ser más eficiente. `EXISTS` también suele ser más rápido que `IN` con subconsultas que devuelven muchas filas.

---

## CTE — Common Table Expression

Una forma más legible de escribir subconsultas complejas, usando `WITH`.

```sql
-- Clientes con más de 5 pedidos
WITH ClientesFrecuentes AS (
    SELECT id_cliente, COUNT(*) AS total_pedidos
    FROM Pedidos
    GROUP BY id_cliente
    HAVING COUNT(*) > 5
)
SELECT c.nombre, cf.total_pedidos
FROM Clientes c
INNER JOIN ClientesFrecuentes cf ON c.id = cf.id_cliente
ORDER BY cf.total_pedidos DESC;
```
