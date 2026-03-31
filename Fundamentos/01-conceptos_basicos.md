# 📌 Conceptos básicos de SQL Server

## ¿Qué es SQL Server?

SQL Server es un **Sistema Gestor de Bases de Datos Relacional (RDBMS)** desarrollado por Microsoft.
Permite crear, administrar y consultar bases de datos relacionales usando el lenguaje **T-SQL** (Transact-SQL), que es la versión extendida de SQL estándar que usa SQL Server.

---

## Ediciones principales

| Edición | Para qué sirve |
|---|---|
| Express | Gratuita, limitada, para aprender o apps pequeñas |
| Developer | Gratuita, todas las funciones, solo para desarrollo |
| Standard | Uso empresarial medio |
| Enterprise | Uso empresarial a gran escala |

---

## Componentes principales

| Componente | Descripción |
|---|---|
| **SSMS** | SQL Server Management Studio — interfaz gráfica para administrar SQL Server |
| **Motor de BD** | Servicio principal que procesa las consultas y gestiona los datos |
| **T-SQL** | Lenguaje de consulta extendido que usa SQL Server |
| **SQL Server Agent** | Servicio para automatizar tareas (backups, jobs, alertas) |

---

## Tipos de datos comunes

### Numéricos
| Tipo | Descripción |
|---|---|
| `INT` | Entero (-2.147.483.648 a 2.147.483.647) |
| `BIGINT` | Entero grande |
| `SMALLINT` | Entero pequeño |
| `DECIMAL(p,s)` | Número exacto con decimales. `p` = dígitos totales, `s` = decimales |
| `FLOAT` | Número con decimales aproximado |
| `BIT` | Booleano (0 o 1) |

### Texto
| Tipo | Descripción |
|---|---|
| `CHAR(n)` | Texto de longitud fija (rellena con espacios) |
| `VARCHAR(n)` | Texto de longitud variable hasta n caracteres |
| `VARCHAR(MAX)` | Texto de longitud variable hasta 2GB |
| `NVARCHAR(n)` | Igual que VARCHAR pero soporta Unicode |
| `TEXT` | Texto largo (obsoleto, usar VARCHAR(MAX)) |

### Fecha y hora
| Tipo | Descripción |
|---|---|
| `DATE` | Solo fecha (YYYY-MM-DD) |
| `TIME` | Solo hora |
| `DATETIME` | Fecha y hora |
| `DATETIME2` | Fecha y hora con mayor precisión |

---

## Objetos de una base de datos

| Objeto | Descripción |
|---|---|
| **Tabla** | Estructura principal que almacena datos en filas y columnas |
| **Vista** | Consulta guardada que se comporta como una tabla virtual |
| **Procedimiento almacenado** | Bloque de código T-SQL reutilizable |
| **Función** | Similar al procedimiento pero retorna un valor |
| **Índice** | Estructura que acelera las búsquedas |
| **Trigger** | Código que se ejecuta automáticamente ante un evento (INSERT, UPDATE, DELETE) |
