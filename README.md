# Food Store — TP de Concurrencia e IA (Base de Datos II)

Proyecto integrador de un sistema de venta de comida, implementado en PostgreSQL. Este repositorio contiene el Trabajo Práctico de Laboratorio de la Semana 2 (Unidad 1: Integridad, Transacciones y Concurrencia), resuelto en grupo de 3 integrantes con OpenCode y Kiro como herramientas de IA.

## Grupo

Grupo 10

**Integrantes:**
- Mariano Chriino
- Andres Fabre
- Facundo Quiroga

## Repositorio

**Link:**  https://github.com/marianoemir/food-store-Parte-2

## Cómo está organizado este repositorio

### Archivos de la base de datos
Están dentro de la carpeta **`Archivos necesarios para la BD/`**:

| Archivo | Contenido |
|---|---|
| `schema.sql` | Tipos ENUM, tablas, constraints e índices |
| `objects.sql` | Vistas, función de cálculo, triggers y procedimiento `sp_crear_pedido` |
| `data.sql` | Datos de prueba (categorías, productos, usuarios, pedidos) |
| `queries.sql` | Historias de usuario resueltas y consultas analíticas |
| `transacciones.sql` | Escenarios de atomicidad, aislamiento y concurrencia (previos a este TP) |

**Orden de ejecución:** `schema.sql → objects.sql → data.sql → queries.sql`

### Entregables del TP de Concurrencia

| Archivo | Qué contiene |
|---|---|
| `protocolo_seguridad.md` | Los 3 pasos de seguridad (copia, transacción, respaldo) aplicados en cada operación de IA sobre la base |
| `duia.md` | Declaración de Uso de IA de las 3 restricciones de integridad generadas con OpenCode (una por integrante) |
| `informe_concurrencia.md` | Los 4 escenarios de concurrencia reproducidos con dos sesiones, con la explicación de la IA verificada contra el motor real |
| `ejercicio_lectura_critica.md` | Análisis y corrección de los 2 scripts SQL deliberadamente peligrosos del TP |

### Configuración de IA
- `AGENTS.md`: instrucciones para OpenCode sobre la estructura y reglas del proyecto.
- `.kiro/steering/`: convenciones del esquema para Kiro (nombres de tablas, borrado lógico, tipos ENUM, triggers existentes).

## Quién hizo qué

El TP se dividió en 3 partes, una por integrante del grupo, cada una con: una restricción de integridad nueva (generada con OpenCode y verificada en el motor), un escenario de concurrencia reproducido con dos sesiones, y el análisis de uno de los dos scripts de lectura crítica.

| Parte | Restricción de integridad | Escenario de concurrencia | Script analizado |
|---|---|---|---|
| A | Trigger: pedido no puede volver de CONFIRMADO a PENDIENTE | Interbloqueo real (deadlock, error 40P01) | Script 2 (DELETE con NOT IN) |
| B | CHECK: fecha de pedido no puede ser futura | Lectura fantasma | Script 1 (UPDATE sin WHERE) |
| C | CHECK: formato de email válido en usuario | Bloqueo de filas / lectura no repetible | — |

El detalle completo de cada parte (prompt usado, qué generó la IA, qué se aceptó o corrigió, y la verificación con el motor real) está en `duia.md`, `informe_concurrencia.md` y `ejercicio_lectura_critica.md`.

## Cómo levantar el proyecto localmente

```bash
createdb plantilla_food_store
psql -d plantilla_food_store -f "Archivos necesarios para la BD/schema.sql"
psql -d plantilla_food_store -f "Archivos necesarios para la BD/objects.sql"
psql -d plantilla_food_store -f "Archivos necesarios para la BD/data.sql"
```

Antes de aplicar cualquier cambio sobre la base, se sigue el flujo de `protocolo_seguridad.md`: nunca se trabaja sobre `plantilla_food_store` directamente, siempre sobre una copia descartable.
