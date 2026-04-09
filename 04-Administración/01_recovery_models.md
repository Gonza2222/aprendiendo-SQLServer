# Recovery Models (Modelos de Recuperación)

## ¿Qué es un recovery model?

Es una configuración de la base de datos que controla **cómo se registran las operaciones en el transaction log**, qué tipos de backup están disponibles y hasta qué punto se puede recuperar la base ante un fallo.

---

## Los tres modelos

| Modelo | Log | Backup de log | Recuperación punto exacto |
|---|---|---|---|
| **Simple** | Mínimo, se trunca automáticamente | ❌ | ❌ |
| **Bulk-Logged** | Mínimo para operaciones masivas | ✅ | ❌ (en período bulk) |
| **Full** | Completo, nunca se trunca solo | ✅ | ✅ |

---

## 1. Simple

- El transaction log **se trunca automáticamente** al hacer un checkpoint
- No permite backup de log → no hay recuperación a un punto exacto
- El log no crece indefinidamente
- Ideal para entornos de desarrollo, bases de datos de prueba o donde la pérdida de datos no es crítica

```sql
ALTER DATABASE Ventas SET RECOVERY SIMPLE;
```

**Riesgo:** si la base falla, se pierden todos los cambios desde el último backup.

---

## 2. Full

- **Todo** queda registrado en el log
- El log NO se trunca automáticamente → necesita backups de log periódicos para liberar espacio
- Permite restaurar a un **punto exacto en el tiempo** (point-in-time recovery)
- Requiere al menos un backup full antes de poder hacer backups de log
- Ideal para producción donde la pérdida de datos es inaceptable

```sql
ALTER DATABASE Ventas SET RECOVERY FULL;
```

**Importante:** si no se hacen backups de log regularmente, el archivo `.ldf` crece sin límite.

---

## 3. Bulk-Logged

- Similar a Full, pero las **operaciones masivas** (BULK INSERT, SELECT INTO, CREATE INDEX, etc.) se loguean mínimamente
- Permite backups de log, pero **no** point-in-time recovery durante el período en que se ejecutaron operaciones bulk
- Reduce el tamaño del log durante cargas masivas
- Diseñado para usarse **temporalmente** durante ETLs o cargas grandes, luego volver a Full

```sql
ALTER DATABASE Ventas SET RECOVERY BULK_LOGGED;
```

**Uso típico:**
1. Cambiar a Bulk-Logged antes de la carga masiva
2. Ejecutar la operación bulk
3. Hacer backup de log
4. Volver a Full

---

## Ver el recovery model actual

```sql
-- Una base específica
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = 'Ventas';

-- Todas las bases
SELECT name, recovery_model_desc, log_reuse_wait_desc
FROM sys.databases;
```

---

## Transaction Log: conceptos clave

| Concepto | Descripción |
|---|---|
| **VLF** (Virtual Log File) | Unidades internas del log; el log se divide en VLFs |
| **Checkpoint** | Punto donde SQL Server escribe páginas sucias a disco y puede truncar el log (en Simple) |
| **Truncamiento del log** | Libera espacio lógico en el log (no necesariamente en disco) |
| **Shrink del log** | Reduce el tamaño físico del archivo `.ldf` |
| **log_reuse_wait_desc** | Indica por qué el log no puede truncarse |

### Valores comunes de log_reuse_wait_desc

| Valor | Causa |
|---|---|
| `NOTHING` | El log puede truncarse libremente |
| `LOG_BACKUP` | Esperando un backup de log (modelo Full/Bulk-Logged) |
| `ACTIVE_TRANSACTION` | Hay una transacción abierta |
| `REPLICATION` | La replicación no procesó los cambios aún |
| `CHECKPOINT` | Esperando un checkpoint |

---

## Cambiar el recovery model

```sql
-- A Full
ALTER DATABASE MiBase SET RECOVERY FULL;

-- A Simple
ALTER DATABASE MiBase SET RECOVERY SIMPLE;

-- A Bulk-Logged
ALTER DATABASE MiBase SET RECOVERY BULK_LOGGED;
```

> Después de cambiar de Simple a Full, hay que hacer un **backup full** antes de que los backups de log sean posibles.

---

## Resumen de decisión

```
¿Necesito recuperar hasta el último minuto antes del fallo?
    └── SÍ → FULL (con backups de log regulares)
    └── NO → ¿Tenés cargas masivas frecuentes?
                └── SÍ → Alternar BULK-LOGGED / FULL
                └── NO → SIMPLE
```
