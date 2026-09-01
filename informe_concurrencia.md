# Informe de Concurrencia — Food Store

Este archivo reúne los escenarios de concurrencia reproducidos por el grupo, con la explicación de la IA verificada contra el motor real en cada caso.

---

## Escenario 1 — Lectura fantasma (Parte B)

| Campo | Contenido |
|---|---|
| **Escenario** | Lectura fantasma: un `COUNT` repetido dentro de la misma transacción cambia de resultado porque otra sesión inserta una fila nueva que cumple la condición del `WHERE`, y la confirma en el medio. |
| **Cómo se reprodujo** | **Sesión A:** `BEGIN;` → `SELECT COUNT(*) FROM producto WHERE categoria_id = 1;` (dio 5). **Sesión B:** `INSERT INTO producto (nombre, precio, stock, categoria_id) VALUES ('Pizza nueva', 5000, 10, 1);` → `COMMIT;`. **Sesión A (misma transacción):** se repite el `SELECT COUNT(*)` → `COMMIT;` |
| **Qué se observó** | El mismo `SELECT COUNT(*)` devolvió primero **5**, luego **6**, sin que Sesión A hiciera ningún cambio — la diferencia se debió a que Sesión B insertó y confirmó una fila nueva en el medio. |
| **Explicación de la IA** | *"Fenómeno: Lectura fantasma (Phantom Read)... El nivel de aislamiento por defecto en PostgreSQL es READ COMMITTED. En este nivel, cada instrucción SELECT toma una nueva instantánea al comenzar, viendo todos los cambios confirmados hasta ese instante... Qué nivel lo evitaría: REPEATABLE READ (o SERIALIZABLE), donde la instantánea se toma al inicio de la transacción."* |
| **Verificación en el motor** | Se repitió con `SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;`. Primer `COUNT`: **6**. Sesión B insertó y confirmó otra fila (`'Pizza test 2'`). Segundo `COUNT` en Sesión A: **6** — no cambió. |
| **Conclusión** | La explicación se confirmó en el motor real. Con `READ COMMITTED` el conteo cambió de 5 a 6; con `REPEATABLE READ` se mantuvo estable en 6. `REPEATABLE READ` evita la lectura fantasma en PostgreSQL. |

---

## Escenario 2 — Interbloqueo real / Deadlock, error 40P01 (Parte A — Andrés)

| Campo | Contenido |
|---|---|
| **Escenario** | Interbloqueo real (Deadlock) — Error 40P01. |
| **Cómo se reprodujo** | Comandos cruzados de `SELECT ... FOR UPDATE` entre Sesión A y Sesión B sobre `producto` (id=1 e id=2), en orden cruzado: A toma la fila 1 y pide la 2; B toma la fila 2 y pide la 1. |
| **Qué se observó** | La Sesión B fue abortada automáticamente por PostgreSQL con el error `40P01: deadlock detected`. |
| **Explicación de la IA** | *"Un interbloqueo ocurre cuando dos o más transacciones mantienen bloqueos sobre recursos que las otras necesitan, formando una espera circular. PostgreSQL cuenta con un Deadlock Detector que, tras el `deadlock_timeout` (por defecto 1 segundo), analiza el grafo de esperas (wait-for graph), detecta el ciclo, elige una sesión como víctima y aborta su transacción con `ERROR: deadlock detected (SQLSTATE 40P01)`."* La estrategia recomendada para evitarlo: mantener un **orden consistente de adquisición de bloqueos** (por ejemplo, ordenar siempre los IDs de menor a mayor antes de bloquear), mantener las transacciones cortas, usar reintentos con backoff ante el error 40P01, o alternativas como `NOWAIT` / `SKIP LOCKED` según el caso de uso. |
| **Verificación en el motor** | Se comprobó que, forzando un orden determinista (ordenar los IDs de menor a mayor en ambas transacciones antes de bloquear), la segunda sesión simplemente espera a que la primera termine, sin ciclo posible, previniendo el interbloqueo. |
| **Conclusión** | A diferencia de las anomalías de lectura, un interbloqueo no se soluciona subiendo el nivel de aislamiento (`REPEATABLE READ` o `SERIALIZABLE` lo detectan igual), sino garantizando un orden estricto en la adquisición de bloqueos. |

---

## Escenario 3 — Espera por bloqueo de filas (Parte C)

| Campo | Contenido |
|---|---|
| **Escenario** | Bloqueo de filas (`Row Exclusive Lock`): dos transacciones intentan actualizar concurrentemente el mismo registro (`id = 1`) en `producto`. |
| **Cómo se reprodujo** | **Sesión A:** `BEGIN; UPDATE producto SET precio = 1500.00 WHERE id = 1;` (adquiere el bloqueo, no confirma todavía). **Sesión B (en paralelo):** `BEGIN; UPDATE producto SET precio = 1800.00 WHERE id = 1;` — queda en espera. **Sesión A:** `COMMIT;` — libera el bloqueo. **Sesión B:** se destraba automáticamente y completa su `UPDATE`. |
| **Qué se observó** | La Sesión B quedó retenida apenas intentó actualizar la fila que Sesión A tenía modificada sin confirmar. En cuanto A hizo `COMMIT`, B continuó su ejecución de inmediato. |
| **Explicación de la IA** | *"PostgreSQL aplica READ COMMITTED por defecto. Cuando una transacción ejecuta un UPDATE, adquiere un bloqueo exclusivo sobre la fila (ROW EXCLUSIVE LOCK). Toda transacción concurrente que intente modificar la misma fila debe esperar a que la transacción poseedora del bloqueo haga COMMIT o ROLLBACK. Esto previene la pérdida de actualizaciones (Lost Update)."* |
| **Verificación en el motor** | Confirmado tal cual: la Sesión B esperó hasta el `COMMIT` de A, y recién ahí ejecutó su propio `UPDATE` sin error ni pérdida de datos. |
| **Conclusión** | El bloqueo `FOR UPDATE`/`ROW EXCLUSIVE LOCK` de PostgreSQL evita que dos transacciones concurrentes pisen la misma fila sin coordinación, garantizando que los cambios se apliquen de forma serializada sobre esa fila puntual. |

---

## Escenario 4 — Lectura no repetible (Parte C) — *pendiente*

*Falta completar: reproducir con Read Committed (mostrar que el mismo SELECT cambia de valor tras un UPDATE confirmado de otra sesión) y repetir con REPEATABLE READ (mostrar que ahí no cambia). Ver la guía ya compartida con la spec y los comandos exactos para producto id=7.*
