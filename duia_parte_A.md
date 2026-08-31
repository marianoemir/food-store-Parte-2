# Declaración de Uso de IA (DUIA) — Food Store

Este documento registra el uso de herramientas de IA (OpenCode) en cada ejercicio del TP de Concurrencia, siguiendo el flujo obligatorio de la cátedra: copia, transacción y respaldo antes de aplicar cualquier cambio generado por IA.

---

## Parte 1 — Restricción de integridad: fecha de pedido no puede ser futura

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

**Qué hace esta línea:** agrega una condición que PostgreSQL evalúa en cada `INSERT` o `UPDATE` sobre `pedido`. Si el valor de `fecha` es posterior a la fecha del día (`CURRENT_DATE`), la operación se rechaza automáticamente con un error de violación de `CHECK`, sin necesidad de que ninguna capa de la aplicación lo valide.

### Qué se aceptó
Se aceptó el cambio tal cual lo propuso OpenCode, sin modificaciones. Es una única línea, acotada y directamente verificable — no requirió ajustes.

### Qué se modificó o descartó, y por qué
No hubo correcciones manuales. El plan generado fue preciso, usó el nombre exacto de tabla y columna pedidos en la spec, y no tocó ninguna otra parte del esquema.

### Verificación realizada

Todo lo siguiente se ejecutó sobre la base **`copia_trabajo`** (copia de `food_store_parte2`), siguiendo el protocolo de seguridad de la cátedra: primero se aplicó el cambio equivalente con `ALTER TABLE`, y luego se probó dentro de transacciones que terminaron en `ROLLBACK`, para no dejar datos de prueba cargados.

```sql
-- 1) Aplicar el cambio sobre copia_trabajo
ALTER TABLE pedido ADD CONSTRAINT chk_fecha_no_futura CHECK (fecha <= CURRENT_DATE);
-- ALTER TABLE
-- Query returned successfully in 54 msec.

-- 2) Caso inválido: fecha futura
BEGIN;
INSERT INTO pedido (fecha, forma_pago, usuario_id) VALUES (CURRENT_DATE + 5, 'EFECTIVO', 1);
-- ERROR: el nuevo registro para la relación «pedido» viola la restricción «check» «chk_fecha_no_futura»
-- Detail: La fila que falla contiene (7, 2026-09-04, PENDIENTE, 0.00, EFECTIVO, 1, f, 2026-08-30 12:40:37.164684-03).
-- SQL state: 23514
ROLLBACK;
-- Query returned successfully in 54 msec.

-- 3) Caso válido: fecha de hoy
BEGIN;
INSERT INTO pedido (fecha, forma_pago, usuario_id) VALUES (CURRENT_DATE, 'EFECTIVO', 1);
-- INSERT 0 1
-- Query returned successfully in 55 msec.
ROLLBACK;
-- Query returned successfully in 44 msec.
```

**Resultado:**

| Caso | Fecha probada | Resultado esperado | Resultado real |
|---|---|---|---|
| Inválido | `2026-09-04` (futura) | Rechazado | ✅ Rechazado — error `23514`, violación de `chk_fecha_no_futura` |
| Válido | `CURRENT_DATE` (hoy) | Aceptado | ✅ Aceptado — `INSERT 0 1` |

La restricción se comporta exactamente como se esperaba en ambos casos. No hubo discrepancia entre lo propuesto por la IA y el comportamiento real del motor.

### Commit
```
git add schema.sql
git commit -m "CHECK: la fecha de un pedido no puede ser posterior a la fecha actual"
git push
```

---

## Parte 2 — Escenario de concurrencia: lectura fantasma

### Herramienta
OpenCode (proveedor Google Gemini) — usado únicamente para pedir la explicación del fenómeno, no para generar código.

### Escenario reproducido
Lectura fantasma: un `COUNT` repetido dentro de la misma transacción cambia de resultado porque otra sesión inserta una fila nueva que cumple la condición del `WHERE`, y la confirma en el medio.

### Cómo se reprodujo (nivel por defecto: Read Committed)

```sql
-- Sesión A
BEGIN;
SELECT COUNT(*) FROM producto WHERE categoria_id = 1;
-- count: 5

-- (mientras tanto, en Sesión B)
INSERT INTO producto (nombre, precio, stock, categoria_id) VALUES ('Pizza nueva', 5000, 10, 1);
COMMIT;

-- de nuevo en Sesión A, misma transacción, se repite la consulta
SELECT COUNT(*) FROM producto WHERE categoria_id = 1;
-- count: 6

COMMIT;
```

### Qué se observó
El mismo `SELECT COUNT(*)`, ejecutado dos veces dentro de la misma transacción de Sesión A, devolvió resultados distintos (5 y luego 6) sin que Sesión A hiciera ningún cambio — la diferencia se debió exclusivamente a que Sesión B insertó y confirmó una fila nueva en el medio.

### Explicación de la IA (copiada tal cual, herramienta: OpenCode / Gemini)

> "Fenómeno: Lectura fantasma (Phantom Read), donde una consulta ejecutada dos veces dentro de la misma transacción devuelve un conjunto diferente de filas debido a inserciones (o eliminaciones) confirmadas por otra transacción concurrente.
>
> Por qué pasa en PostgreSQL (Read Committed): El nivel de aislamiento por defecto en PostgreSQL es READ COMMITTED. En este nivel, cada instrucción (SELECT) individual dentro de una transacción toma una nueva instantánea (snapshot) al momento de comenzar su ejecución, viendo todos los cambios confirmados (committed) hasta ese instante, incluso si la transacción general sigue abierta.
>
> Qué nivel lo evitaría: REPEATABLE READ (o SERIALIZABLE). En el nivel REPEATABLE READ, la instantánea se toma al inicio de la primera consulta de toda la transacción (en lugar de cada instrucción), garantizando que todas las lecturas posteriores dentro de la misma transacción vean exactamente el mismo estado y no perciban filas insertadas por otras transacciones concurrentes."

### Verificación en el motor (con REPEATABLE READ)

```sql
-- Sesión A
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT COUNT(*) FROM producto WHERE categoria_id = 1;
-- count: 6

-- (mientras tanto, en Sesión B, otra fila nueva)
INSERT INTO producto (nombre, precio, stock, categoria_id) VALUES ('Pizza test 2', 4000, 5, 1);
COMMIT;

-- de nuevo en Sesión A, misma transacción
SELECT COUNT(*) FROM producto WHERE categoria_id = 1;
-- count: 6  (no cambió, a pesar del INSERT confirmado por Sesión B)

COMMIT;
```

### Conclusión
La explicación de la IA se confirmó exactamente en el motor real. Con el nivel por defecto (`READ COMMITTED`), el conteo cambió de 5 a 6 dentro de la misma transacción. Al repetir el experimento bajo `REPEATABLE READ`, el conteo se mantuvo estable en 6 las dos veces, a pesar de que Sesión B insertó y confirmó una fila nueva en el medio. Esto confirma que `REPEATABLE READ` (tomando el snapshot al inicio de la transacción, no en cada sentencia) evita la lectura fantasma en PostgreSQL.

---

## Parte 3 — Ejercicio de lectura crítica: Script 1

### Script analizado (adaptado al esquema de Food Store)

El script original del TP usa una tabla `funcion` que no existe en nuestro esquema (es del ejemplo genérico de la cátedra). Se adaptó a una tabla real del proyecto, conservando el mismo error (falta de `WHERE`):

```sql
-- Generado para: dar de baja los productos sin stock
UPDATE producto SET disponible = FALSE;
```

### Qué haría realmente tal como está escrito
Al no tener `WHERE`, el `UPDATE` afecta a **todas** las filas de la tabla `producto`, sin importar si tienen stock o no — no solo a los productos sin stock, como dice el comentario que dice cumplir.

### Por qué no coincide con la consigna
La consigna es "dar de baja los productos sin stock" (o sea, marcar `disponible = FALSE` solo donde `stock = 0`). Tal como está escrito, el script ignora por completo la condición de stock y desactiva incluso productos con stock disponible, lo cual rompería la operatoria normal de la tienda (ningún producto podría venderse, tenga stock o no).

### Verificación en el motor (sobre `copia_trabajo`)

```sql
-- Total de productos en la base
SELECT COUNT(*) FROM producto;
-- count: 18

-- Productos que SÍ tienen stock (no deberían marcarse como no disponibles)
SELECT COUNT(*) FROM producto WHERE stock > 0;
-- count: 16

-- Prueba del script original (mal), dentro de transacción
BEGIN;
UPDATE producto SET disponible = FALSE;
SELECT COUNT(*) FROM producto WHERE disponible = FALSE;
-- count: 18   (afectó a TODOS los productos, incluidos los 16 con stock)
ROLLBACK;

-- Prueba del script corregido, dentro de transacción
BEGIN;
UPDATE producto SET disponible = FALSE WHERE stock = 0;
SELECT COUNT(*) FROM producto WHERE disponible = FALSE;
-- count: 2    (solo afectó a los productos que realmente no tienen stock)
ROLLBACK;
```

### Conclusión
El script original, al carecer de `WHERE`, marca los 18 productos de la tabla como no disponibles, cuando en realidad solo 2 productos tienen `stock = 0` y deberían quedar marcados así. Los 16 productos restantes (con stock > 0) quedarían incorrectamente deshabilitados para la venta. La versión corregida agrega la condición faltante:

```sql
UPDATE producto SET disponible = FALSE WHERE stock = 0;
```

Con esta corrección, el efecto verificado en el motor coincide exactamente con lo que la consigna dice cumplir: solo se desactivan los 2 productos sin stock, dejando intactos los 16 restantes.

### Principio general
En cualquier tabla transaccional (inventarios, productos, pedidos), omitir el `WHERE` en un `UPDATE` o `DELETE` es un error crítico que modifica masivamente todos los registros de la tabla. Toda sentencia de este tipo debe incluir siempre un filtro preciso (habitualmente por `id`, o por la condición de negocio exacta que se quiere cumplir).

###Parte A (Andrés)
Se le dio el siguiente Promp a openCode: "En la tabla pedido, agregar un trigger BEFORE UPDATE que impida que el campo estado pase de 'CONFIRMADO' a 'PENDIENTE'. Si se intenta, debe lanzar una excepción con RAISE EXCEPTION."
El resultado fue el siguiente: 

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

--Realizamos la siguiente prueba

Caso Inválido (debe lanzar excepción):
BEGIN;
-- Tomar un pedido en estado CONFIRMADO o actualizar uno a CONFIRMADO
UPDATE pedido SET estado = 'CONFIRMADO' WHERE id = 1;
-- Intentar cambiarlo a PENDIENTE
UPDATE pedido SET estado = 'PENDIENTE' WHERE id = 1; -- Debe fallar con RAISE EXCEPTION
ROLLBACK;

El resultado fue el esperado, lanzando el siguiente error: 

ERROR:  No se permite cambiar el estado de un pedido de CONFIRMADO a PENDIENTE (Pedido ID: 1)
CONTEXTO:  función PL/pgSQL fn_validar_estado_pedido() en la línea 4 en RAISE

--Realiazamos la siguiente prueba

Caso Válido (transición permitida):
BEGIN;
UPDATE pedido SET estado = 'CONFIRMADO' WHERE id = 1;
UPDATE pedido SET estado = 'TERMINADO' WHERE id = 1;  -- Debe ejecutarse correctamente
ROLLBACK;

El resultado fue el esperado, ejecutándose correctamente

