# Declaración de Uso de IA (DUIA) — Food Store

Este documento registra el uso de herramientas de IA (OpenCode) en cada ejercicio del TP de Concurrencia, siguiendo el flujo obligatorio de la cátedra: copia, transacción y respaldo antes de aplicar cualquier cambio generado por IA.

---

## Parte 1 — Restricción de integridad: validación de formato de email en usuario

### Herramienta
OpenCode (proveedor Google Gemini, modelo Gemini 2.5 Pro / Flash)

### Spec o prompt utilizado
> "En la tabla usuario, agregar una restricción CHECK que valide que el campo mail tenga formato de email válido (contiene @ y un dominio)."

### Contexto de la regla de negocio
Hasta el momento, la tabla `usuario` no contaba con una validación a nivel de motor para el formato de correo electrónico. Esto permitía la posibilidad de registrar direcciones sintácticamente inválidas (por ejemplo, sin `@` o sin un dominio/TLD correspondiente). Garantizar esta restricción directamente en PostgreSQL asegura la consistencia de los datos independientemente de las validaciones de la interfaz o aplicación.

### Qué generó
OpenCode propuso modificar la definición de la tabla `usuario` en `schema.sql`, agregando la restricción `CHECK` a nivel de tabla con una expresión regular (RegEx) para validar el formato de email:

```sql
CREATE TABLE usuario (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre       VARCHAR(100) NOT NULL,
    apellido     VARCHAR(100) NOT NULL,
    mail         VARCHAR(255) NOT NULL UNIQUE,
    telefono     VARCHAR(20),
    contrasena   VARCHAR(255) NOT NULL,
    rol          rol NOT NULL DEFAULT 'USUARIO',
    eliminado    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_usuario_mail_formato CHECK (
        mail ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
    )
);
```

### Diff aplicado (revisado línea por línea antes de aceptar)
```diff
@@ -47,7 +47,10 @@ CREATE TABLE usuario (
     contrasena   VARCHAR(255) NOT NULL,
     rol          rol NOT NULL DEFAULT 'USUARIO',
     eliminado    BOOLEAN NOT NULL DEFAULT FALSE,
-    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
+    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
+    CONSTRAINT chk_usuario_mail_formato CHECK (
+        mail ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
+    )
 );
```

**Qué hace este bloque:**
* **`CONSTRAINT chk_usuario_mail_formato`**: Otorga un nombre explícito a la regla de integridad.
* **`mail ~`**: Utiliza el operador de comparación con expresiones regulares en PostgreSQL.
* **`'^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'`**: Evalúa que el mail posea un usuario válido, el símbolo `@`, un nombre de dominio y una extensión TLD de al menos 2 letras.

### Qué se aceptó
Se aceptó la propuesta de OpenCode sin modificaciones, ya que la expresión regular sugerida es lo suficientemente estricta y resuelve completamente la necesidad planteada.

### Qué se modificó o descartó, y por qué
No se requirieron correcciones. Los correos de prueba ya cargados en `data.sql` (`ana.garcia@mail.com`, `juan.perez@mail.com`, etc.) cumplían con la expresión regular, por lo que no hubo inconsistencias ni fallos en la carga inicial.

### Verificación realizada

Siguiendo el protocolo de seguridad de la cátedra, la verificación se realizó reconstruyendo la base `plantilla_food_store` con el nuevo `schema.sql` y clonando la base de trabajo descartable **`copia_trabajo_c`** (`createdb -T plantilla_food_store copia_trabajo_c`). Se ejecutaron inserciones de prueba dentro de bloques `BEGIN...ROLLBACK`:

```sql
-- 1) Caso inválido: mail sin formato (debe fallar)
BEGIN;
INSERT INTO usuario (nombre, apellido, mail, contrasena)
VALUES ('Test', 'Invalido', 'mail_sin_formato', 'hash123');
-- ERROR: el nuevo registro para la relación «usuario» viola la restricción «check» «chk_usuario_mail_formato»
-- DETALLE: La fila que falla contiene (7, Test, Invalido, mail_sin_formato, null, hash123, USUARIO, f, 2026-08-30 22:38:38.336747-03).
ROLLBACK;

-- 2) Caso válido: mail correcto (debe pasar)
BEGIN;
INSERT INTO usuario (nombre, apellido, mail, contrasena)
VALUES ('Test', 'Valido', 'usuario.valido@dominio.com', 'hash123');
-- INSERT 0 1
ROLLBACK;
```

**Resultado:**

| Caso | Mail probado | Resultado esperado | Resultado real |
|---|---|---|---|
| Inválido | `'mail_sin_formato'` | Rechazado | ✅ Rechazado — error violando la restricción `chk_usuario_mail_formato` |
| Válido | `'usuario.valido@dominio.com'` | Aceptado | ✅ Aceptado — `INSERT 0 1` |

### Commit
```bash
git add schema.sql
git commit -m "Parte 1: CHECK de formato de email en usuario"
```

---

## Parte 2 — Escenarios de concurrencia: Bloqueo de Filas y Deadlock

### Herramienta
OpenCode (proveedor Google Gemini) — utilizado para el diseño y análisis de los escenarios experimentales de concurrencia.

---

### Experimento 1: Bloqueo de Filas (Row Exclusive Lock)

#### Escenario reproducido
Simulación de concurrencia donde dos transacciones intentan actualizar concurrentemente el precio del mismo registro (`id = 1`) en la tabla `producto`.

#### Procedimiento y comandos ejecutados

* **Sesión A (Terminal 1):**
  ```sql
  BEGIN;
  SELECT id, nombre, precio FROM producto WHERE id = 1;
  -- id: 1 | nombre: Muzzarella | precio: 3500.00

  UPDATE producto SET precio = 1500.00 WHERE id = 1;
  -- UPDATE 1 (bloqueo exclusivo sobre la tupla)
  ```

* **Sesión B (Terminal 2, en paralelo):**
  ```sql
  BEGIN;
  SELECT id, nombre, precio FROM producto WHERE id = 1;
  -- id: 1 | nombre: Muzzarella | precio: 3500.00

  UPDATE producto SET precio = 1800.00 WHERE id = 1;
  -- (Sesión B queda en estado de espera / bloqueada)
  ```

* **Sesión A (Terminal 1):**
  ```sql
  COMMIT;
  -- Al confirmar A, se libera el candado sobre la fila id = 1.
  ```

* **Sesión B (Terminal 2):**
  ```sql
  -- Inmediatamente se destraba y completa la operación:
  -- UPDATE 1
  COMMIT;
  ```

#### Qué se observó
Al ejecutar el `UPDATE` en la Sesión B sobre una fila modificada en la Sesión A (aún no confirmada), la Sesión B quedó automáticamente retenida. En cuanto la Sesión A ejecutó `COMMIT`, la Sesión B continuó su ejecución actualizando la tupla.

#### Explicación técnica
PostgreSQL aplica un nivel de aislamiento `READ COMMITTED` por defecto. Cuando una transacción ejecuta un `UPDATE`, adquiere un bloqueo exclusivo sobre la fila (`ROW EXCLUSIVE LOCK`). Toda transacción concurrente que intente actualizar o eliminar esa misma fila debe esperar a que la transacción poseedora del bloqueo realice `COMMIT` o `ROLLBACK`. Esto previene la pérdida de actualizaciones (*Lost Update*).

---

### Experimento 2: Interbloqueo (Deadlock)

#### Escenario reproducido
Simulación de dependencia circular donde la Sesión A retiene la fila 1 y solicita la fila 2, mientras la Sesión B retiene la fila 2 y solicita la fila 1.

#### Procedimiento y comandos ejecutados

* **Sesión A (Terminal 1):**
  ```sql
  BEGIN;
  UPDATE producto SET precio = 2000.00 WHERE id = 1;
  -- UPDATE 1
  ```

* **Sesión B (Terminal 2):**
  ```sql
  BEGIN;
  UPDATE producto SET precio = 3000.00 WHERE id = 2;
  -- UPDATE 1
  ```

* **Sesión A (Terminal 1):**
  ```sql
  UPDATE producto SET precio = 2500.00 WHERE id = 2;
  -- (Sesión A queda bloqueada esperando que B libere id = 2)
  ```

* **Sesión B (Terminal 2):**
  ```sql
  UPDATE producto SET precio = 3500.00 WHERE id = 1;
  -- Provoca Deadlock al intentar acceder a id = 1 en posesión de A
  ```

#### Resultado arrojado por el motor (Sesión B):
```text
ERROR:  se ha detectado un deadlock
DETALLE:  El proceso 10400 espera ShareLock en transacción 927; bloqueado por proceso 2524.
El proceso 2524 espera ShareLock en transacción 928; bloqueado por proceso 10400.
SUGERENCIA:  Vea el registro del servidor para obtener detalles de las consultas.
CONTEXTO:  mientras se actualizaba la tupla (0,32) en la relación «producto»
copia_trabajo_c=!# ROLLBACK;
```

#### Qué se observó
El motor de PostgreSQL detectó automáticamente la espera circular activa entre el `proceso 10400` y el `proceso 2524`, abortando de manera inmediata la transacción de la Sesión B e indicando `ERROR: se ha detectado un deadlock`. Esto obligó a la Sesión B a ejecutar un `ROLLBACK`.

#### Explicación técnica
El mecanismo de detección de *Deadlocks* de PostgreSQL analiza de forma periódica el grafo de esperas entre procesos. Al detectar un ciclo donde ninguna transacción puede avanzar sin que otra finalice, aborta una de las transacciones involucradas para romper la condición de bloqueo mutuo y permitir que las demás transacciones del sistema puedan continuar.

---

### Resumen de los Experimentos de Concurrencia

| Experimento | Comportamiento Observado | Explicación Técnica |
| :--- | :--- | :--- |
| **1. Bloqueo de Filas (Row Lock)** | La Sesión B aguarda hasta que la Sesión A ejecuta `COMMIT`. | **Row Exclusive Lock:** Evita la pérdida de actualizaciones (*Lost Update*) garantizando el nivel de aislamiento *Read Committed*. |
| **2. Interbloqueo (Deadlock)** | Sesión B es abortada con `ERROR: se ha detectado un deadlock`. | **Deadlock Detection:** El motor detecta la dependencia circular entre transacciones y aborta una mediante `ROLLBACK` forzado. |

### Commit
```bash
git add duia_companero_c.md
git commit -m "Parte 2: Verificación de experimentos de concurrencia y deadlocks"
```
