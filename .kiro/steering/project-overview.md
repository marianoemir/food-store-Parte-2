# Food Store — Visión General del Proyecto

## Descripción

Sistema de venta de comida implementado íntegramente en **PostgreSQL**.
Cubre el ciclo completo: categorías → productos → usuarios → pedidos con sus detalles.

## Archivos del proyecto

| Archivo | Contenido |
|---|---|
| `schema.sql` | Tipos ENUM, tablas, constraints e índices |
| `objects.sql` | Vistas, función de cálculo, triggers y procedimiento `sp_crear_pedido` |
| `data.sql` | Datos de prueba (categorías, productos, usuarios, pedidos de ejemplo) |
| `queries.sql` | Historias de usuario resueltas + consultas analíticas |
| `transacciones.sql` | Escenarios de atomicidad, aislamiento y bloqueo concurrente |

## Tablas del modelo

```
categoria
producto       → FK categoria_id → categoria
usuario
pedido         → FK usuario_id   → usuario
detalle_pedido → FK pedido_id    → pedido
               → FK producto_id  → producto
```

## Tipos ENUM definidos

```sql
CREATE TYPE rol          AS ENUM ('ADMIN','USUARIO');
CREATE TYPE estado_pedido AS ENUM ('PENDIENTE','CONFIRMADO','TERMINADO','CANCELADO');
CREATE TYPE forma_pago   AS ENUM ('TARJETA','TRANSFERENCIA','EFECTIVO');
```

## Orden de ejecución

```
schema.sql → objects.sql → data.sql → queries.sql / transacciones.sql
```

Cada archivo depende del anterior; ejecutarlos fuera de orden producirá errores de referencia.

## Vistas disponibles

| Vista | Descripción |
|---|---|
| `v_categorias_vigentes` | Categorías con `eliminado = FALSE` |
| `v_productos_vigentes` | Productos y categorías activos, con JOIN entre ambas tablas |
| `v_pedidos_resumen` | Pedidos vigentes con nombre completo del usuario |
| `v_pedido_detalle` | Líneas de detalle vigentes con nombre del producto |
