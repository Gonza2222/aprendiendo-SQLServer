# Auditoría de Usuarios y Permisos

Reporte completo de logins, users, roles y permisos de toda la instancia. Solo lectura, no modifica nada. Ejecutar la sección según lo que necesitás auditar.

---

## 1. Logins del servidor

```sql
SELECT
    name                    AS login,
    type_desc               AS tipo,
    is_disabled             AS deshabilitado,
    is_policy_checked       AS verifica_politica,
    is_expiration_checked   AS verifica_expiracion,
    create_date,
    modify_date,
    default_database_name   AS base_default
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G')  -- S=SQL login, U=Windows user, G=Windows group
ORDER BY type_desc, name;
```

---

## 2. Roles de servidor y sus miembros

```sql
SELECT
    r.name      AS rol_servidor,
    m.name      AS miembro,
    m.type_desc AS tipo_miembro
FROM sys.server_role_members rm
JOIN sys.server_principals r ON rm.role_principal_id = r.principal_id
JOIN sys.server_principals m ON rm.member_principal_id = m.principal_id
ORDER BY r.name, m.name;
```

---

## 3. Permisos a nivel de servidor

```sql
SELECT
    p.class_desc,
    p.permission_name,
    p.state_desc        AS estado,
    sp.name             AS login_o_rol
FROM sys.server_permissions p
JOIN sys.server_principals sp ON p.grantee_principal_id = sp.principal_id
WHERE sp.type NOT IN ('R')
ORDER BY sp.name, p.permission_name;
```

---

## 4. Usuarios por base de datos

```sql
SELECT
    DB_NAME()               AS base_datos,
    name                    AS usuario,
    type_desc               AS tipo,
    authentication_type_desc AS autenticacion,
    create_date,
    modify_date,
    default_schema_name     AS esquema_default
FROM sys.database_principals
WHERE type IN ('S', 'U', 'G', 'E', 'X')
  AND name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
ORDER BY name;
```

---

## 5. Roles de base de datos y sus miembros

```sql
SELECT
    DB_NAME()       AS base_datos,
    r.name          AS rol,
    m.name          AS miembro,
    m.type_desc     AS tipo_miembro
FROM sys.database_role_members rm
JOIN sys.database_principals r ON rm.role_principal_id = r.principal_id
JOIN sys.database_principals m ON rm.member_principal_id = m.principal_id
ORDER BY r.name, m.name;
```

---

## 6. Permisos explícitos sobre objetos

```sql
SELECT
    DB_NAME()                       AS base_datos,
    p.class_desc,
    OBJECT_NAME(p.major_id)         AS objeto,
    p.permission_name,
    p.state_desc                    AS estado,  -- GRANT, DENY, REVOKE
    dp.name                         AS usuario_o_rol,
    dp.type_desc                    AS tipo
FROM sys.database_permissions p
JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
WHERE p.class = 1
  AND dp.name NOT IN ('dbo', 'public')
ORDER BY dp.name, OBJECT_NAME(p.major_id), p.permission_name;
```

---

## 7. Permisos a nivel de base (no sobre objetos)

```sql
SELECT
    DB_NAME()               AS base_datos,
    p.permission_name,
    p.state_desc            AS estado,
    dp.name                 AS usuario_o_rol
FROM sys.database_permissions p
JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
WHERE p.class = 0           -- CREATE TABLE, BACKUP DATABASE, etc.
  AND dp.name NOT IN ('dbo', 'public')
ORDER BY dp.name, p.permission_name;
```

---

## 8. Mapeo Login → User (todas las bases)

```sql
EXEC sp_msforeachdb '
USE [?];
SELECT
    ''?''           AS base_datos,
    sp.name         AS login,
    dp.name         AS usuario_bd,
    dp.type_desc    AS tipo
FROM sys.database_principals dp
JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN (''S'', ''U'', ''G'')
  AND dp.name NOT IN (''dbo'', ''guest'')
ORDER BY sp.name;
';
```

---

## 9. Usuarios huérfanos (sin login en el servidor)

```sql
SELECT
    dp.name         AS usuario_sin_login,
    dp.type_desc,
    dp.create_date
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN ('S', 'U')
  AND sp.sid IS NULL
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys');
```

---

## 10. Permisos efectivos de un usuario específico

```sql
-- Cambiar 'juan' por el usuario a auditar
EXECUTE AS USER = 'juan';

SELECT * FROM fn_my_permissions(NULL, 'DATABASE')
ORDER BY permission_name;

REVERT;

-- Verificar permisos sobre una tabla específica
SELECT
    HAS_PERMS_BY_NAME('Clientes', 'OBJECT', 'SELECT')  AS puede_SELECT,
    HAS_PERMS_BY_NAME('Clientes', 'OBJECT', 'INSERT')  AS puede_INSERT,
    HAS_PERMS_BY_NAME('Clientes', 'OBJECT', 'UPDATE')  AS puede_UPDATE,
    HAS_PERMS_BY_NAME('Clientes', 'OBJECT', 'DELETE')  AS puede_DELETE;
```

---

## 11. Reporte completo: Usuario → Roles → Permisos

```sql
SELECT DISTINCT
    DB_NAME()                   AS base_datos,
    m.name                      AS usuario,
    r.name                      AS rol_asignado,
    p.permission_name,
    p.state_desc                AS estado,
    OBJECT_NAME(p.major_id)     AS objeto
FROM sys.database_role_members rm
JOIN sys.database_principals r  ON rm.role_principal_id   = r.principal_id
JOIN sys.database_principals m  ON rm.member_principal_id = m.principal_id
JOIN sys.database_permissions p ON p.grantee_principal_id = r.principal_id
WHERE p.class = 1
  AND m.name NOT IN ('dbo', 'public')
ORDER BY m.name, r.name, p.permission_name;
```

---

## Roles de servidor predefinidos

| Rol | Descripción |
|---|---|
| `sysadmin` | Control total del servidor |
| `serveradmin` | Configurar opciones del servidor |
| `securityadmin` | Gestionar logins y permisos |
| `dbcreator` | Crear, modificar, eliminar bases |
| `bulkadmin` | Ejecutar BULK INSERT |

## Roles de base de datos predefinidos

| Rol | Descripción |
|---|---|
| `db_owner` | Control total de la base |
| `db_datareader` | Leer todas las tablas |
| `db_datawriter` | Escribir en todas las tablas |
| `db_ddladmin` | Crear y modificar objetos |
| `db_securityadmin` | Gestionar roles y permisos |
| `db_backupoperator` | Hacer backups |
