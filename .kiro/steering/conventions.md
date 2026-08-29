# Food Store — Convenciones de Base de Datos

## Nombres de tablas

Los nombres de tabla son **en singular y en español**:

```
categoria, producto, usuario, pedido, detalle_pedido
```

No usar plural (`productos`), no usar inglés (`product`), no usar PascalCase.

## Columna `eliminado` — Borrado lógico

**Nunca se ejecuta `DELETE`** sobre ninguna tabla del proyecto.

Todas las tablas tienen `eliminado BOOLEAN NOT NULL DEFAULT FALSE`.
La baja de un registro consiste siempre en:

```sql
UPDATE <tabla> SET eliminado = TRUE WHERE id = :id AND eliminado = FALSE;
```

Toda consulta de datos vigentes debe filtrar `WHERE eliminado = FALSE`.
Las vistas `v_categorias_vigentes`, `v_productos_vigentes`, `v_pedidos_resumen` y `v_pedido_detalle` ya aplican ese filtro; úsalas en lugar de escribir el filtro a mano cuando sea posible.

### Baja de un pedido completo

La baja de un pedido requiere una transacción explícita que marque primero los detalles y luego el pedido:

```sql
BEGIN;
    UPDATE detalle_pedido SET eliminado = TRUE WHERE pedido_id = :id;
    UPDATE pedido SET eliminado = TRUE WHERE id = :id;
COMMIT;
```

## Columna `precio_unitario` en `detalle_pedido`

Este campo **congela el precio** del producto en el momento de la venta.
No se actualiza si el precio del producto cambia después.
El trigger `trg_subtotal` lo copia de `producto.precio` si llega `NULL`.

## Columna `disponible` en `producto`

Controla si el producto puede ser pedido. El procedimiento `sp_crear_pedido` rechaza productos con `disponible = FALSE` aunque tengan stock.

## Restricciones de integridad destacadas

- `categoria.nombre` → `UNIQUE`
- `usuario.mail` → `UNIQUE`
- `detalle_pedido(pedido_id, producto_id)` → `UNIQUE` (un producto no puede repetirse en el mismo pedido)
- `producto.precio >= 0`, `producto.stock >= 0`, `detalle_pedido.cantidad > 0`
- `detalle_pedido.pedido_id` tiene `ON DELETE RESTRICT` (no se puede borrar un pedido con detalles via DELETE — lo cual es consistente con la política de borrado lógico)

## Tipos de datos a respetar

| Campo | Tipo |
|---|---|
| IDs | `BIGINT GENERATED ALWAYS AS IDENTITY` |
| Precios y totales | `NUMERIC(10,2)` / `NUMERIC(12,2)` |
| Fechas con zona | `TIMESTAMPTZ` (campo `created_at`) |
| Fecha del pedido | `DATE` (campo `fecha`) |
| Contraseña | `VARCHAR(255)` — almacenar solo hash, nunca texto plano |

## Índices existentes

```sql
idx_producto_categoria_id   -- producto.categoria_id
idx_pedido_usuario_id       -- pedido.usuario_id
idx_producto_nombre_vigente -- producto(nombre) WHERE eliminado = FALSE  (índice parcial)
```

No duplicar estos índices al agregar nuevas consultas.
