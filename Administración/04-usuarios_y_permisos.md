# Usuarios y Permisos

## Arquitectura de seguridad en SQL Server

```
Nivel de servidor:   LOGIN  (quién puede conectarse al servidor)
                        │
Nivel de base:       USER   (quién puede acceder a la base de datos)
                        │
Nivel de objeto:     PERMISOS (qué puede hacer sobre tablas, vistas, SPs, etc.)
```

Un **Login** existe a nivel servidor. Un **User** existe a nivel base de datos y está mapeado a un Login.

---

## Tipos de autenticación

| Tipo | Descripción |
|---|---|
| **Windows Authentication** | Usa cuentas de Windows/Active Directory. Más seguro. |
| **SQL Server Authentication** | Usuario y contraseña propios de SQL Server. |

---

## 1. Logins (nivel servidor)

### Crear un login SQL Server

```sql
CREATE LOGIN juan WITH PASSWORD = 'Passw0rd!';
```

### Crear un login Windows

```sql
CREATE LOGIN [DOMINIO\juan] FROM WINDOWS;
```

### Modificar un login

```sql
-- Cambiar contraseña
ALTER LOGIN juan WITH PASSWORD = 'NuevaClave123!';

-- Deshabilitar login
ALTER LOGIN juan DISABLE;

-- Habilitar login
ALTER LOGIN juan ENABLE;

-- Renombrar
ALTER LOGIN juan WITH NAME = juanperez;
```

### Eliminar un login

```sql
DROP LOGIN juan;
```

### Ver logins existentes

```sql
SELECT name, type_desc, is_disabled, create_date
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G')  -- S=SQL, U=Windows user, G=Windows group
ORDER BY name;
```

---

## 2. Users (nivel base de datos)

### Crear un user mapeado a un login

```sql
USE Ventas;

CREATE USER juan FOR LOGIN juan;
-- o con nombre diferente:
CREATE USER JuanVentas FOR LOGIN juan;
```

### Crear un user sin login (contained database)

```sql
CREATE USER juanlocal WITH PASSWORD = 'Passw0rd!';
```

### Eliminar un user

```sql
USE Ventas;
DROP USER juan;
```

### Ver users de la base actual

```sql
SELECT name, type_desc, create_date
FROM sys.database_principals
WHERE type NOT IN ('R')  -- excluye roles
ORDER BY name;
```

---

## 3. Roles

### Roles de servidor predefinidos

| Rol | Descripción |
|---|---|
| `sysadmin` | Control total del servidor |
| `serveradmin` | Configurar opciones del servidor |
| `securityadmin` | Gestionar logins y permisos |
| `dbcreator` | Crear, modificar, eliminar bases |
| `diskadmin` | Gestionar archivos de disco |
| `bulkadmin` | Ejecutar BULK INSERT |
| `public` | Todos los logins lo tienen por defecto |

```sql
-- Agregar login a un rol de servidor
ALTER SERVER ROLE sysadmin ADD MEMBER juan;

-- Ver miembros de un rol de servidor
SELECT r.name AS rol, m.name AS miembro
FROM sys.server_role_members rm
JOIN sys.server_principals r ON rm.role_principal_id = r.principal_id
JOIN sys.server_principals m ON rm.member_principal_id = m.principal_id;
```

### Roles de base de datos predefinidos

| Rol | Descripción |
|---|---|
| `db_owner` | Control total de la base |
| `db_datareader` | Leer todas las tablas |
| `db_datawriter` | Escribir en todas las tablas |
| `db_ddladmin` | Crear y modificar objetos |
| `db_securityadmin` | Gestionar roles y permisos |
| `db_backupoperator` | Hacer backups |
| `db_denydatareader` | No puede leer datos |
| `db_denydatawriter` | No puede modificar datos |
| `public` | Todos los users lo tienen |

```sql
-- Agregar user a un rol de base
ALTER ROLE db_datareader ADD MEMBER juan;
ALTER ROLE db_datawriter ADD MEMBER juan;

-- Quitar de un rol
ALTER ROLE db_datareader DROP MEMBER juan;

-- Ver roles de un user
SELECT r.name AS rol
FROM sys.database_role_members rm
JOIN sys.database_principals r ON rm.role_principal_id = r.principal_id
JOIN sys.database_principals m ON rm.member_principal_id = m.principal_id
WHERE m.name = 'juan';
```

### Roles de base de datos personalizados

```sql
-- Crear un rol propio
CREATE ROLE rol_ventas;

-- Darle permisos al rol
GRANT SELECT, INSERT ON Pedidos TO rol_ventas;

-- Asignar users al rol
ALTER ROLE rol_ventas ADD MEMBER juan;
ALTER ROLE rol_ventas ADD MEMBER maria;
```

---

## 4. Permisos (GRANT / DENY / REVOKE)

### Sintaxis general

```sql
GRANT  permiso ON objeto TO usuario_o_rol;
DENY   permiso ON objeto TO usuario_o_rol;
REVOKE permiso ON objeto FROM usuario_o_rol;
```

### Permisos sobre tablas/vistas

```sql
-- Dar permiso de lectura
GRANT SELECT ON Clientes TO juan;

-- Dar múltiples permisos
GRANT SELECT, INSERT, UPDATE ON Pedidos TO juan;

-- Dar todos los permisos DML
GRANT SELECT, INSERT, UPDATE, DELETE ON Productos TO rol_ventas;

-- Permiso sobre una columna específica
GRANT SELECT (ClienteID, Nombre) ON Clientes TO juan;

-- Denegar explícitamente (tiene prioridad sobre GRANT)
DENY DELETE ON Clientes TO juan;

-- Quitar un permiso
REVOKE INSERT ON Pedidos FROM juan;
```

### Permisos sobre stored procedures

```sql
GRANT EXECUTE ON sp_BuscarCliente TO juan;
REVOKE EXECUTE ON sp_BuscarCliente FROM juan;
```

### Permisos a nivel base de datos

```sql
GRANT CREATE TABLE TO juan;
GRANT CREATE VIEW TO juan;
GRANT BACKUP DATABASE TO juan;
```

### Permisos a nivel servidor

```sql
GRANT VIEW SERVER STATE TO juan;
GRANT ALTER ANY LOGIN TO juan;
```

---

## 5. Ver permisos efectivos

```sql
-- Permisos de un user sobre objetos de la base
SELECT class_desc, OBJECT_NAME(major_id) AS objeto,
       permission_name, state_desc
FROM sys.database_permissions
WHERE grantee_principal_id = USER_ID('juan');

-- Permisos efectivos del usuario actual
EXECUTE AS USER = 'juan';
SELECT * FROM fn_my_permissions(NULL, 'DATABASE');
REVERT;

-- Verificar si un user puede hacer algo
HAS_PERMS_BY_NAME('Clientes', 'OBJECT', 'SELECT')
-- Devuelve 1 (sí) o 0 (no)
```

---

## 6. Schema (esquema)

Los esquemas agrupan objetos y simplifican la gestión de permisos.

```sql
-- Crear un esquema
CREATE SCHEMA ventas;

-- Crear objeto en ese esquema
CREATE TABLE ventas.Facturas (FacturaID INT, Total DECIMAL(10,2));

-- Dar permisos sobre todo el esquema
GRANT SELECT ON SCHEMA::ventas TO juan;
GRANT EXECUTE ON SCHEMA::ventas TO rol_ventas;
```

---

## Flujo típico para un nuevo usuario

```sql
-- 1. Crear el login (nivel servidor)
CREATE LOGIN maria WITH PASSWORD = 'MiClave2024!';

-- 2. Crear el user en la base (nivel base)
USE Ventas;
CREATE USER maria FOR LOGIN maria;

-- 3. Asignar a un rol o dar permisos
ALTER ROLE db_datareader ADD MEMBER maria;
GRANT EXECUTE ON usp_GenerarFactura TO maria;
DENY DELETE ON Clientes TO maria;
```

---

## Usuario especial: sa

- Login `sa` = System Administrator, existe por defecto
- Es miembro de `sysadmin`, tiene control total
- **Buena práctica:** deshabilitarlo y usar otro login con `sysadmin` con nombre no obvio

```sql
ALTER LOGIN sa DISABLE;
```
