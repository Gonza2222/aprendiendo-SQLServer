# 🔍 Filtros y ordenamiento

## WHERE — Filtrar registros

```sql
SELECT * FROM Clientes
WHERE ciudad = 'Córdoba';
```

### Operadores de comparación

| Operador | Descripción | Ejemplo |
|---|---|---|
| `=` | Igual | `WHERE edad = 30` |
| `<>` o `!=` | Distinto | `WHERE ciudad <> 'Buenos Aires'` |
| `>` | Mayor que | `WHERE precio > 1000` |
| `<` | Menor que | `WHERE precio < 1000` |
| `>=` | Mayor o igual | `WHERE edad >= 18` |
| `<=` | Menor o igual | `WHERE stock <= 10` |

### Operadores lógicos

```sql
-- AND: ambas condiciones deben cumplirse
SELECT * FROM Productos
WHERE precio > 100 AND stock > 0;

-- OR: al menos una condición debe cumplirse
SELECT * FROM Clientes
WHERE ciudad = 'Córdoba' OR ciudad = 'Rosario';

-- NOT: niega la condición
SELECT * FROM Clientes
WHERE NOT ciudad = 'Buenos Aires';
```

---

## BETWEEN — Rango de valores

```sql
-- Valores entre 100 y 500 (inclusive)
SELECT * FROM Productos
WHERE precio BETWEEN 100 AND 500;

-- Fechas en un rango
SELECT * FROM Pedidos
WHERE fecha BETWEEN '2024-01-01' AND '2024-12-31';
```

---

## IN — Lista de valores

```sql
-- En lugar de varios OR
SELECT * FROM Clientes
WHERE ciudad IN ('Córdoba', 'Rosario', 'Mendoza');

-- Negación
SELECT * FROM Clientes
WHERE ciudad NOT IN ('Buenos Aires', 'La Plata');
```

---

## LIKE — Búsqueda por patrón

| Comodín | Descripción | Ejemplo |
|---|---|---|
| `%` | Cualquier cantidad de caracteres | `'Juan%'` → empieza con Juan |
| `_` | Un solo carácter | `'_uan'` → Juan, Buan, etc. |

```sql
-- Nombres que empiezan con 'Mar'
SELECT * FROM Clientes WHERE nombre LIKE 'Mar%';

-- Nombres que terminan con 'ez'
SELECT * FROM Clientes WHERE nombre LIKE '%ez';

-- Nombres que contienen 'arc'
SELECT * FROM Clientes WHERE nombre LIKE '%arc%';

-- Emails que no son de Gmail
SELECT * FROM Clientes WHERE email NOT LIKE '%@gmail.com';
```

---

## IS NULL / IS NOT NULL

```sql
-- Registros sin teléfono
SELECT * FROM Clientes WHERE telefono IS NULL;

-- Registros con teléfono cargado
SELECT * FROM Clientes WHERE telefono IS NOT NULL;
```

---

## ORDER BY — Ordenar resultados

```sql
-- Ascendente (por defecto)
SELECT * FROM Clientes ORDER BY nombre ASC;

-- Descendente
SELECT * FROM Productos ORDER BY precio DESC;

-- Por múltiples columnas
SELECT * FROM Clientes ORDER BY ciudad ASC, nombre ASC;
```

---

## TOP + ORDER BY — Primeros/últimos registros

```sql
-- Los 5 productos más caros
SELECT TOP 5 * FROM Productos ORDER BY precio DESC;

-- Los 3 clientes más recientes
SELECT TOP 3 * FROM Clientes ORDER BY fecha_alt DESC;
```

---

## Funciones útiles en filtros

```sql
-- Convertir a mayúsculas para comparar sin importar el case
SELECT * FROM Clientes
WHERE UPPER(nombre) LIKE 'JUAN%';

-- Largo de un texto
SELECT * FROM Clientes
WHERE LEN(nombre) > 20;

-- Año de una fecha
SELECT * FROM Pedidos
WHERE YEAR(fecha) = 2024;

-- Mes de una fecha
SELECT * FROM Pedidos
WHERE MONTH(fecha) = 3;
```
