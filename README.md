# 🗄️ SQL Server - Notas y Cheatsheets

Repositorio con apuntes, cheatsheets y scripts de SQL Server que fui armando mientras estudio la **Tecnicatura en Análisis de Sistemas** y me especializo en administración de bases de datos.  
La idea es que sirva tanto como referencia personal como para cualquiera que esté aprendiendo SQL Server desde cero.

---

## 📂 Contenido

### 📁 fundamentos/

| Archivo | Descripción |
|---|---|
| `01-conceptos-basicos.md` | Qué es SQL Server, ediciones, tipos de datos, objetos |
| `02-bases-de-datos.md` | CREATE, USE, ALTER, DROP DATABASE |
| `03-tablas.md` | CREATE TABLE, ALTER TABLE, TRUNCATE, DROP |

### 📁 consultas/

| Archivo | Descripción |
|---|---|
| `01-crud.md` | INSERT, SELECT, UPDATE, DELETE |
| `02-filtros-y-ordenamiento.md` | WHERE, LIKE, BETWEEN, IN, ORDER BY |
| `03-agrupamiento.md` | GROUP BY, HAVING, funciones de agregado |
| `04-joins.md` | INNER, LEFT, RIGHT, FULL, CROSS JOIN |
| `05-subconsultas.md` | Subconsultas, EXISTS, CTE |

### 📁 objetos/

| Archivo | Descripción |
|---|---|
| `01_vistas.md` | CREATE/ALTER/DROP VIEW, SCHEMABINDING, CHECK OPTION, vistas indexadas |
| `02_procedimientos_almacenados.md` | CREATE PROCEDURE, parámetros, OUTPUT, TRY/CATCH, transacciones |
| `03_funciones.md` | Funciones escalares, inline TVF, multi-statement, APPLY |
| `04_triggers.md` | Triggers DML y DDL, AFTER vs INSTEAD OF, tablas INSERTED/DELETED |
| `05_indices.md` | Clustered, non-clustered, únicos, filtrados, columnstore, fragmentación |

### 📁 administracion/

| Archivo | Descripción |
|---|---|
| `01_recovery_models.md` | Modelos Simple, Full y Bulk-Logged, transaction log, log_reuse_wait_desc |
| `02_backups.md` | Full, diferencial, log, copy-only, verificación, historial, estrategias |
| `03_restore.md` | WITH RECOVERY/NORECOVERY/MOVE/STOPAT, errores comunes, tail-log backup |
| `04_usuarios_y_permisos.md` | Logins, users, roles de servidor y base, GRANT/DENY/REVOKE, schemas |
| `05_jobs_y_mantenimiento.md` | SQL Server Agent, sysjobs, crear jobs con T-SQL, DBCC, planes de mantenimiento |
| `06_monitoreo.md` | sp_who2, DMVs, bloqueos, queries costosas, wait stats, fragmentación |

### 📁 scripts/

> En construcción

---

## 🗺️ Roadmap

- [x] Fundamentos (bases de datos, tablas, tipos de datos)
- [x] CRUD (INSERT, SELECT, UPDATE, DELETE)
- [x] Filtros, ordenamiento y agrupamiento
- [x] JOINs
- [x] Subconsultas y CTEs
- [x] Vistas
- [x] Procedimientos almacenados y funciones
- [x] Triggers
- [x] Índices
- [x] Usuarios y permisos
- [x] Backups y restauración
- [x] Modelos de recuperación (Simple, Full, Bulk-Logged)
- [x] Jobs y mantenimiento
- [x] Monitoreo y DMVs
- [ ] Scripts de utilidad

---

## 👤 Autor

**Gonzalo** — Estudiante de Tecnicatura en Análisis de Sistemas  
📍 Argentina
