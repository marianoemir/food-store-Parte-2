###Escenario: Interbloqueo real (Deadlock) - Error 40P01.
###Cómo se reprodujo: Comandos cruzados de SELECT ... FOR UPDATE entre Sesión A y Sesión B sobre producto (id=1 e id=2).
###Qué se observó: La Sesión B fue abortada automáticamente por PostgreSQL arrojando el error 40P01: deadlock detected

###Explicación de la IA

1. ¿Por qué ocurrió el interbloqueo (Deadlock — error 40P01)?
Un interbloqueo (deadlock) ocurre cuando dos o más transacciones mantienen bloqueos sobre recursos que las otras necesitan y ninguna puede continuar hasta que la otra libere su bloqueo, formando una espera circular.

Secuencia cronológica del fenómeno:
Tiempo	Sesión A	Sesión B	Estado en PostgreSQL
$t_1$	BEGIN;	BEGIN;	Ambas transacciones abiertas.
$t_2$	SELECT ... WHERE id = 1 FOR UPDATE; (adquiere bloqueo exclusivo sobre id=1)	—	Sesión A retiene la fila 1.
$t_3$	—	SELECT ... WHERE id = 2 FOR UPDATE; (adquiere bloqueo exclusivo sobre id=2)	Sesión B retiene la fila 2.
$t_4$	SELECT ... WHERE id = 2 FOR UPDATE; (intenta bloquear id=2)	—	Sesión A se bloquea en espera de que B libere la fila 2.
$t_5$	—	SELECT ... WHERE id = 1 FOR UPDATE; (intenta bloquear id=1)	Sesión B se bloquea en espera de que A libere la fila 1.
$t_6$	⏳ En espera...	⏳ En espera...	Ciclo de dependencia: A espera a B y B espera a A.
Al llegar a $t_6$, ninguna sesión puede avanzar por sí misma. PostgreSQL cuenta con un proceso en segundo plano (Deadlock Detector) que, tras cumplirse el tiempo límite configurado en deadlock_timeout (por defecto 1 segundo), analiza el grafo de esperas (wait-for graph), detecta el ciclo, elige a una de las sesiones como víctima y aborta su transacción lanzando la excepción:

ERROR: deadlock detected (SQLSTATE 40P01)
2. Estrategia recomendada a nivel de código de aplicación
La solución definitiva consiste en eliminar la condición de espera circular. Las mejores prácticas para lograrlo son:

A. Orden consistente de adquisición de bloqueos (Consistent Locking Order — Recomendada)
Si todas las transacciones bloquean las filas en el mismo orden determinístico (por ejemplo, ordenando los IDs de menor a mayor), el ciclo es matemáticamente imposible.

A nivel de aplicación (Backend): Ordenar la lista de productos por id antes de enviar la consulta o llamar al procedimiento:
// Ordenar siempre por ID antes de bloquear
items.sort((a, b) => a.producto_id - b.producto_id);
A nivel SQL: Si se realiza un SELECT ... FOR UPDATE multi-fila, incluir siempre ORDER BY id:
SELECT id, stock 
FROM producto 
WHERE id IN (1, 2) 
ORDER BY id 
FOR UPDATE;
B. Mantener transacciones cortas y atómicas
Adquirir los bloqueos lo más tarde posible dentro de la transacción.
Realizar las validaciones y cálculos antes de abrir la transacción o ejecutar el FOR UPDATE, minimizando el tiempo que se retiene el bloqueo.
C. Política de reintentos (Retry Mechanism con Backoff)
A nivel de aplicación, atrapar el código de error 40P01 y reintentar la transacción tras un breve retardo aleatorio (jitter), ya que los deadlocks bajo alta concurrencia pueden ser esporádicos.

D. Alternativas según el caso de uso
NOWAIT: SELECT ... FOR UPDATE NOWAIT falla inmediatamente en lugar de quedar esperando si la fila está bloqueada.
SKIP LOCKED: Útil para colas de trabajo donde no importa procesar exactamente esa fila si ya está ocupada por otra sesión.

###Verificación en el motor: Se comprobó que al forzar el orden determinista (ordenar los IDs a bloquear de menor a mayor en ambas transacciones), la segunda sesión simplemente espera a que la primera termine, eliminando la posibilidad de ciclo y previniendo el interbloqueo.

###Conclusión: A diferencia de las anomalías de lectura, un interbloqueo no se soluciona subiendo el nivel de aislamiento (de hecho, REPEATABLE READ o SERIALIZABLE lo detectan igual), sino garantizando un orden estricto en la adquisición de bloqueos.