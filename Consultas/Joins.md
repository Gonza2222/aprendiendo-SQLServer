# 🔗 JOINs

Los JOINs permiten combinar filas de dos o más tablas basándose en una columna relacionada.

---

## Tipos de JOIN

### INNER JOIN
Devuelve solo las filas que tienen coincidencia en **ambas** tablas.

```sql
SELECT p.id, c.nombre, p.total
FROM Pedidos p
INNER JOIN Clientes c ON p.id_cliente = c.id;
```

### LEFT JOIN (LEFT OUTER JOIN)
Devuelve **todas las filas de la tabla izquierda** y las coincidencias de la derecha. Si no hay coincidencia, completa con `NULL`.

```sql
-- Todos los clientes, hayan hecho pedidos o no
SELECT c.nombre, p.total
FROM Clientes c
LEFT JOIN Pedidos p ON c.id = p.id_cliente;
```

### RIGHT JOIN (RIGHT OUTER JOIN)
Devuelve **todas las filas de la tabla derecha** y las coincidencias de la izquierda.

```sql
SELECT c.nombre, p.total
FROM Clientes c
RIGHT JOIN Pedidos p ON c.id = p.id_cliente;
```

### FULL JOIN (FULL OUTER JOIN)
Devuelve **todas las filas de ambas tablas**. Completa con `NULL` donde no hay coincidencia.

```sql
SELECT c.nombre, p.total
FROM Clientes c
FULL JOIN Pedidos p ON c.id = p.id_cliente;
```

### CROSS JOIN
Devuelve el **producto cartesiano**: todas las combinaciones posibles entre las dos tablas.

```sql
SELECT c.nombre, p.nombre
FROM Colores c
CROSS JOIN Productos p;
```

---

## Diagrama visual

```
Tabla A    Tabla B

INNER JOIN        → solo la intersección
LEFT JOIN         → todo A + intersección
RIGHT JOIN        → intersección + todo B
FULL JOIN         → todo A + todo B
```

---

## JOIN con alias

```sql
-- Usar alias para abreviar nombres de tabla
SELECT c.nombre, p.total, pr.nombre AS producto
FROM Pedidos p
INNER JOIN Clientes c ON p.id_cliente = c.id
INNER JOIN Productos pr ON p.id_producto = pr.id;
```

---

## JOIN con filtros

```sql
-- Pedidos de clientes de Córdoba en 2024
SELECT c.nombre, p.total, p.fecha
FROM Pedidos p
INNER JOIN Clientes c ON p.id_cliente = c.id
WHERE c.ciudad = 'Córdoba'
  AND YEAR(p.fecha) = 2024
ORDER BY p.fecha DESC;
```

---

## Encontrar registros sin coincidencia

```sql
-- Clientes que nunca hicieron un pedido
SELECT c.nombre
FROM Clientes c
LEFT JOIN Pedidos p ON c.id = p.id_cliente
WHERE p.id IS NULL;
```

---

## Self JOIN — unir una tabla consigo misma

Útil cuando una tabla tiene una relación consigo misma (ej: empleados con jefes).

```sql
-- Empleados con el nombre de su jefe
SELECT e.nombre AS empleado, j.nombre AS jefe
FROM Empleados e
INNER JOIN Empleados j ON e.id_jefe = j.id;
```
