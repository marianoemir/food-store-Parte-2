# Food Store — Instrucciones para Agentes

## Stack Tecnológico

| Componente | Tecnología / Versión | Propósito |
|---|---|---|
| Motor BD | PostgreSQL | Base de datos relacional |
| Lenguaje | SQL / PL/pgSQL | Definición de esquema, vistas, funciones y procedimientos |

## Orden de ejecución de los .sql

`schema.sql → objects.sql → data.sql → queries.sql / transacciones.sql`

Cada archivo depende del anterior; correrlos fuera de orden produce errores de referencia. `data.sql` y `queries.sql` ya llaman a `sp_crear_pedido`.

## Base de Conocimiento y Steering

- `schema.sql`: Tipos ENUM, tablas, constraints e índices.
- `objects.sql`: Vistas, función de cálculo, triggers y procedimiento `sp_crear_pedido`.
- `data.sql`: Datos de prueba (categorías, productos, usuarios, pedidos).
- `queries.sql`: Historias de usuario resueltas y consultas analíticas.
- `transacciones.sql`: Escenarios de atomicidad, aislamiento y concurrencia.
- `protocolo_seguridad.md`: **Leer primero**: flujo obligatorio antes de tocar la BD (ver abajo).
- `.kiro/steering/project-overview.md`: Visión general y orden de ejecución.
- `.kiro/steering/conventions.md`: Convenciones (nombres en singular, borrado lógico, tipos).
- `.kiro/steering/objects-and-patterns.md`: Triggers automáticos y procedimiento `sp_crear_pedido`.

## Protocolo de seguridad ante la BD (obligatorio)

Ningún script (propio o generado por IA) se ejecuta directo sobre la base con datos. Flujo siempre:

1. Trabajar sobre una copia descartable: `createdb -T plantilla_food_store copia_trabajo`.
2. Todo script de escritura corre primero dentro de `BEGIN...ROLLBACK` y se inspecciona antes de aceptarlo.
3. Cambios estructurales (ALTER/DROP/CREATE TRIGGER/CREATE FUNCTION/migración) requieren `pg_dump` de respaldo previo a `respaldos/`.
4. Recién al final: `COMMIT` y luego commit en Git.

Detalles y comandos exactos en `protocolo_seguridad.md`.

## Reglas Duras del Proyecto

1. **Nombres de tablas:** En singular y español (`categoria`, `producto`, `usuario`, `pedido`, `detalle_pedido`).
2. **Borrado lógico:** Nunca ejecutar `DELETE`. Usar `UPDATE <tabla> SET eliminado = TRUE WHERE id = :id AND eliminado = FALSE`. La baja de un pedido completo requiere transacción: primero `detalle_pedido`, luego `pedido`.
3. **Altas de pedidos:** Usar siempre `CALL sp_crear_pedido(...)`. Nunca hacer `INSERT INTO pedido` + `INSERT INTO detalle_pedido` manualmente.
4. **Triggers automáticos:** No modificar los triggers de subtotal y totales (`trg_subtotal`, `trg_total_ins`, `trg_total_upd`). Al insertar en `detalle_pedido` solo se proveen `pedido_id`, `producto_id` y `cantidad`; `precio_unitario`, `subtotal` y `pedido.total` se completan solos.
5. **Vistas vigentes:** Utilizar vistas vigentes (`v_categorias_vigentes`, `v_productos_vigentes`, `v_pedidos_resumen`, `v_pedido_detalle`) para filtrar registros activos (`eliminado = FALSE`) en vez de escribir el filtro a mano.
