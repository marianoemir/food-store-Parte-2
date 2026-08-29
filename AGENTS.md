# Food Store — Instrucciones para Agentes

## Stack Tecnológico

| Componente | Tecnología / Versión | Propósito |
|---|---|---|
| Motor BD | PostgreSQL | Base de datos relacional |
| Lenguaje | SQL / PL/pgSQL | Definición de esquema, vistas, funciones y procedimientos |

## Base de Conocimiento y Steering

- `schema.sql`: Tipos ENUM, tablas, constraints e índices.
- `objects.sql`: Vistas, funciones de cálculo, triggers y procedimiento `sp_crear_pedido`.
- `data.sql`: Datos de prueba (categorías, productos, usuarios, pedidos).
- `queries.sql`: Historias de usuario resueltas y consultas analíticas.
- `transacciones.sql`: Escenarios de atomicidad, aislamiento y concurrencia.
- `.kiro/steering/project-overview.md`: Visión general y orden de ejecución.
- `.kiro/steering/conventions.md`: Convenciones (nombres en singular, borrado lógico, tipos).
- `.kiro/steering/objects-and-patterns.md`: Triggers automáticos y procedimiento `sp_crear_pedido`.

## Reglas Duras del Proyecto

1. **Nombres de tablas:** En singular y español (`categoria`, `producto`, `usuario`, `pedido`, `detalle_pedido`).
2. **Borrado lógico:** Nunca ejecutar `DELETE`. Usar `UPDATE <tabla> SET eliminado = TRUE WHERE id = :id`.
3. **Altas de pedidos:** Usar siempre `CALL sp_crear_pedido(...)`. Nunca hacer `INSERT INTO pedido` + `INSERT INTO detalle_pedido` manualmente.
4. **Triggers automáticos:** No modificar los triggers de subtotal y totales (`trg_subtotal`, `trg_total_ins`, `trg_total_upd`). El total del pedido se calcula automáticamente.
5. **Vistas vigentes:** Utilizar vistas vigentes (`v_categorias_vigentes`, `v_productos_vigentes`, `v_pedidos_resumen`, `v_pedido_detalle`) para filtrar registros activos (`eliminado = FALSE`).
