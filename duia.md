# Declaración de Uso de IA (DUIA) — Food Store

Este documento registra el uso de OpenCode para generar las restricciones de integridad del TP de Concurrencia, siguiendo el flujo obligatorio de la cátedra: copia, transacción y respaldo antes de aplicar cualquier cambio generado por IA.

---

# Parte A (Andrés) — Trigger: transición de estado de pedido

### Herramienta
OpenCode

### Spec o prompt utilizado
> "En la tabla pedido, agregar un trigger BEFORE UPDATE que impida que el campo estado pase de 'CONFIRMADO' a 'PENDIENTE'. Si se intenta, debe lanzar una excepción con RAISE EXCEPTION."

### Contexto de la regla de negocio
Un `CHECK` declarativo no alcanza para esta regla porque necesita comparar el valor viejo de `estado` contra el nuevo — por eso se resuelve con un trigger `BEFORE UPDATE`, no con una restricción simple.

### Qué generó
```sql
-- Valida que un pedido CONFIRMADO no pueda volver a estado PENDIENTE
CREATE OR REPLACE FUNCTION fn_validar_estado_pedido()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.estado = 'CONFIRMADO' AND NEW.estado = 'PENDIENTE' THEN
        RAISE EXCEPTION 'No se permite cambiar el estado de un pedido de CONFIRMADO a PENDIENTE (Pedido ID: %)', OLD.id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_validar_estado_pedido
BEFORE UPDATE OF estado ON pedido
FOR EACH ROW EXECUTE FUNCTION fn_validar_estado_pedido();
```

### Qué se aceptó
Se aceptó tal cual lo propuso OpenCode, sin modificaciones.

### Verificación realizada

```sql
-- Caso inválido (debe lanzar excepción)
BEGIN;
UPDATE pedido SET estado = 'CONFIRMADO' WHERE id = 1;
UPDATE pedido SET estado = 'PENDIENTE' WHERE id = 1;
-- ERROR: No se permite cambiar el estado de un pedido de CONFIRMADO a PENDIENTE (Pedido ID: 1)
-- CONTEXTO: función PL/pgSQL fn_validar_estado_pedido() en la línea 4 en RAISE
ROLLBACK;

-- Caso válido (transición permitida)
BEGIN;
UPDATE pedido SET estado = 'CONFIRMADO' WHERE id = 1;
UPDATE pedido SET estado = 'TERMINADO' WHERE id = 1;
-- se ejecuta correctamente
ROLLBACK;
```

**Resultado:**

| Caso | Transición probada | Resultado esperado | Resultado real |
|---|---|---|---|
| Inválido | CONFIRMADO → PENDIENTE | Rechazado | ✅ Rechazado — `RAISE EXCEPTION` |
| Válido | CONFIRMADO → TERMINADO | Aceptado | ✅ Aceptado |

---

# Parte B — Restricción de integridad: fecha de pedido no puede ser futura

### Herramienta
OpenCode (proveedor Google Gemini, modelo Gemini 3.5 Flash Lite)

### Spec o prompt utilizado
> "En la tabla pedido, agregar una restricción CHECK que impida que el campo fecha sea posterior a la fecha actual (CURRENT_DATE)."

### Contexto de la regla de negocio
Hoy, nada impide que se cargue un pedido con una fecha futura (por ejemplo, por un error de carga manual o un bug en la aplicación). Esta regla debería garantizarse en el motor y no depender de que la aplicación la valide correctamente en todos los casos.

### Qué generó
OpenCode propuso, en modo Plan, modificar la definición de la tabla `pedido` en `schema.sql`, agregando la restricción `CHECK` directamente sobre la columna `fecha`:

```sql
CREATE TABLE pedido (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    fecha        DATE NOT NULL DEFAULT CURRENT_DATE CHECK (fecha <= CURRENT_DATE),
    estado       estado_pedido NOT NULL DEFAULT 'PENDIENTE',
    total        NUMERIC(12,2) NOT NULL DEFAULT 0 CHECK (total >= 0),
    forma_pago   forma_pago NOT NULL,
    usuario_id   BIGINT NOT NULL REFERENCES usuario(id),
    eliminado    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Diff aplicado (revisado línea por línea antes de aceptar)
```diff
-    fecha        DATE NOT NULL DEFAULT CURRENT_DATE,
+    fecha        DATE NOT NULL DEFAULT CURRENT_DATE CHECK (fecha <= CURRENT_DATE),
```

**Qué hace esta línea:** agrega una condición que PostgreSQL evalúa en cada `INSERT` o `UPDATE` sobre `pedido`. Si el valor de `fecha` es posterior a la fecha del día (`CURRENT_DATE`), la operación se rechaza automáticamente con un error de violación de `CHECK`.

### Qué se aceptó
Se aceptó el cambio tal cual lo propuso OpenCode, sin modificaciones.

### Verificación realizada

```sql
ALTER TABLE pedido ADD CONSTRAINT chk_fecha_no_futura CHECK (fecha <= CURRENT_DATE);
-- ALTER TABLE / Query returned successfully in 54 msec.

BEGIN;
INSERT INTO pedido (fecha, forma_pago, usuario_id) VALUES (CURRENT_DATE + 5, 'EFECTIVO', 1);
-- ERROR: el nuevo registro para la relación «pedido» viola la restricción «check» «chk_fecha_no_futura»
-- SQL state: 23514
ROLLBACK;

BEGIN;
INSERT INTO pedido (fecha, forma_pago, usuario_id) VALUES (CURRENT_DATE, 'EFECTIVO', 1);
-- INSERT 0 1
ROLLBACK;
```

**Resultado:**

| Caso | Fecha probada | Resultado esperado | Resultado real |
|---|---|---|---|
| Inválido | `2026-09-04` (futura) | Rechazado | ✅ Rechazado — error `23514` |
| Válido | `CURRENT_DATE` (hoy) | Aceptado | ✅ Aceptado — `INSERT 0 1` |

### Commit
```bash
git add schema.sql
git commit -m "CHECK: la fecha de un pedido no puede ser posterior a la fecha actual"
git push
```

---

# Parte C — Restricción de integridad: formato de email en usuario

### Herramienta
OpenCode (proveedor Google Gemini, modelo Gemini 2.5 Pro / Flash)

### Spec o prompt utilizado
> "En la tabla usuario, agregar una restricción CHECK que valide que el campo mail tenga formato de email válido (contiene @ y un dominio)."

### Contexto de la regla de negocio
Hasta el momento, la tabla `usuario` no contaba con una validación a nivel de motor para el formato de correo electrónico. Garantizar esta restricción directamente en PostgreSQL asegura la consistencia de los datos independientemente de las validaciones de la aplicación.

### Qué generó
```sql
CONSTRAINT chk_usuario_mail_formato CHECK (
    mail ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
)
```

**Qué hace este bloque:** usa el operador `~` de PostgreSQL (comparación con expresión regular) para validar que el mail tenga un usuario válido, el símbolo `@`, un dominio y una extensión de al menos 2 letras.

### Qué se aceptó
Se aceptó la propuesta sin modificaciones — la expresión regular es suficientemente estricta y resuelve la necesidad planteada. Los correos de prueba de `data.sql` cumplían la condición sin conflictos.

### Verificación realizada

```sql
BEGIN;
INSERT INTO usuario (nombre, apellido, mail, contrasena)
VALUES ('Test', 'Invalido', 'mail_sin_formato', 'hash123');
-- ERROR: el nuevo registro para la relación «usuario» viola la restricción «check» «chk_usuario_mail_formato»
ROLLBACK;

BEGIN;
INSERT INTO usuario (nombre, apellido, mail, contrasena)
VALUES ('Test', 'Valido', 'usuario.valido@dominio.com', 'hash123');
-- INSERT 0 1
ROLLBACK;
```

**Resultado:**

| Caso | Mail probado | Resultado esperado | Resultado real |
|---|---|---|---|
| Inválido | `'mail_sin_formato'` | Rechazado | ✅ Rechazado |
| Válido | `'usuario.valido@dominio.com'` | Aceptado | ✅ Aceptado — `INSERT 0 1` |

### Commit
```bash
git add schema.sql
git commit -m "CHECK: formato de email en usuario"
```
