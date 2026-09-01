# Ejercicio de Lectura Crítica — Food Store

Análisis de los scripts generados "para dar de baja registros vencidos", identificando qué harían realmente antes de ejecutarlos, y su corrección.

---

## Script 1 — UPDATE sin WHERE (Parte B)

### Script analizado (adaptado al esquema de Food Store)

El script original del TP usa una tabla `funcion` que no existe en nuestro esquema. Se adaptó a una tabla real del proyecto, conservando el mismo error (falta de `WHERE`):

```sql
-- Generado para: dar de baja los productos sin stock
UPDATE producto SET disponible = FALSE;
```

### Qué haría realmente tal como está escrito
Al no tener `WHERE`, el `UPDATE` afecta a **todas** las filas de `producto`, sin importar si tienen stock o no — no solo a los productos sin stock, como dice el comentario que dice cumplir.

### Por qué no coincide con la consigna
La consigna es "dar de baja los productos sin stock" (marcar `disponible = FALSE` solo donde `stock = 0`). Tal como está escrito, desactiva incluso productos con stock disponible, rompiendo la operatoria normal de la tienda.

### Verificación en el motor (sobre `copia_trabajo`)

```sql
SELECT COUNT(*) FROM producto;
-- count: 18

SELECT COUNT(*) FROM producto WHERE stock > 0;
-- count: 16

-- Script original (mal)
BEGIN;
UPDATE producto SET disponible = FALSE;
SELECT COUNT(*) FROM producto WHERE disponible = FALSE;
-- count: 18   (afectó a TODOS, incluidos los 16 con stock)
ROLLBACK;

-- Script corregido
BEGIN;
UPDATE producto SET disponible = FALSE WHERE stock = 0;
SELECT COUNT(*) FROM producto WHERE disponible = FALSE;
-- count: 2    (solo los que realmente no tienen stock)
ROLLBACK;
```

### Conclusión
El script original marca los 18 productos como no disponibles, cuando solo 2 tienen `stock = 0`. La versión corregida agrega la condición faltante:

```sql
UPDATE producto SET disponible = FALSE WHERE stock = 0;
```

### Principio general
En cualquier tabla transaccional, omitir el `WHERE` en un `UPDATE` o `DELETE` es un error crítico que modifica masivamente todos los registros. Toda sentencia de este tipo debe incluir siempre un filtro preciso.

---

## Script 2 — DELETE con NOT IN (Parte A — Andrés)

### Script analizado

```sql
-- Generado para: limpiar las categorías sin productos asociados
DELETE FROM categoria
WHERE id NOT IN (SELECT categoria_id FROM producto);
```

### Qué filas afectaría realmente
En el estado actual de la base: afecta únicamente a las categorías que no tienen ningún producto asociado (por ejemplo, la categoría id=6 "Descontinuados" de `data.sql`). En un escenario real, el resultado depende fuertemente de los valores almacenados en `producto.categoria_id`.

### Por qué es peligroso / no coincide con la seguridad esperada (el problema del NULL con NOT IN)
Aunque en el esquema actual `categoria_id` tiene `NOT NULL`, usar `NOT IN` con una subconsulta es una falla de diseño latente:
1. Si en algún momento esa columna permitiera `NULL` (o se aplicara este patrón sobre otra tabla que sí acepte nulos), basta con que un solo registro de la subconsulta devuelva `NULL` para que toda la expresión `NOT IN` devuelva `UNKNOWN` (equivalente a falso) para cada fila evaluada.
2. Como resultado, el `DELETE` no borraría ninguna fila, fallando silenciosamente sin lanzar ningún error explícito.

### Versión corregida recomendada

```sql
-- Versión corregida utilizando NOT EXISTS
DELETE FROM categoria c
WHERE NOT EXISTS (
    SELECT 1
    FROM producto p
    WHERE p.categoria_id = c.id
);
```

### Por qué es mejor `NOT EXISTS`
- **Seguridad ante NULL:** no se ve afectado si la subconsulta incluye valores nulos; evalúa la existencia lógica fila por fila.
- **Rendimiento:** PostgreSQL suele optimizar mejor las consultas con `NOT EXISTS` usando un anti-join, interrumpiendo la búsqueda apenas encuentra la primera coincidencia en `producto`.
