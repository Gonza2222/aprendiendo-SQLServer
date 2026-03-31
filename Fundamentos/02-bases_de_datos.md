# 🗄️ Bases de datos

## Crear una base de datos

```sql
CREATE DATABASE NombreDB;
```

### Con opciones avanzadas
```sql
CREATE DATABASE NombreDB
ON PRIMARY (
    NAME = 'NombreDB_Data',
    FILENAME = 'C:\Data\NombreDB.mdf',
    SIZE = 10MB,
    MAXSIZE = 100MB,
    FILEGROWTH = 5MB
)
LOG ON (
    NAME = 'NombreDB_Log',
    FILENAME = 'C:\Data\NombreDB.ldf',
    SIZE = 5MB,
    MAXSIZE = 50MB,
    FILEGROWTH = 2MB
);
```

---

## Seleccionar una base de datos

```sql
USE NombreDB;
```

> Siempre ejecutar `USE` antes de trabajar para asegurarse de estar en la base correcta.

---

## Ver bases de datos existentes

```sql
-- Todas las bases del servidor
SELECT name, state_desc, recovery_model_desc
FROM sys.databases;

-- Solo los nombres
SELECT name FROM sys.databases;
```

---

## Modificar una base de datos

### Cambiar el modelo de recuperación
```sql
-- Modelo completo (Full)
ALTER DATABASE NombreDB SET RECOVERY FULL;

-- Modelo simple
ALTER DATABASE NombreDB SET RECOVERY SIMPLE;

-- Registro masivo
ALTER DATABASE NombreDB SET RECOVERY BULK_LOGGED;
```

### Forzar una sola conexión (útil antes de restaurar)
```sql
ALTER DATABASE NombreDB SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
```

### Volver a multiusuario
```sql
ALTER DATABASE NombreDB SET MULTI_USER;
```

---

## Eliminar una base de datos

```sql
DROP DATABASE NombreDB;
```

> ⚠️ Esta operación es irreversible. Asegurarse de tener backup antes de ejecutar.

---

## Ver archivos físicos de una base de datos

```sql
SELECT name, physical_name, state_desc
FROM sys.master_files
WHERE database_id = DB_ID('NombreDB');
```
