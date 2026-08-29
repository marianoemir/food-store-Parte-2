# Protocolo de Seguridad — Food Store

Este documento define los tres pasos obligatorios que se aplican **siempre**, sin excepción, 
antes de que cualquier script (propio o generado por IA) toque la base de datos del proyecto.

## Motor y entorno

- Motor: PostgreSQL
- Base de trabajo local: `food_store_dev`
- Base "plantilla" con el esquema ya aplicado: `plantilla_food_store`

## Paso 1 — Copia

Nunca se trabaja directamente sobre la base que contiene datos que importan (ni siquiera 
datos de prueba que ya llevan tiempo cargados). Antes de cualquier cambio se crea una copia 
descartable:

```bash
createdb -T plantilla_food_store copia_trabajo
```

Si la plantilla `plantilla_food_store` todavía no existe, se crea una vez a partir del esquema:

```bash
createdb plantilla_food_store
psql -d plantilla_food_store -f schema.sql
psql -d plantilla_food_store -f objects.sql
psql -d plantilla_food_store -f data.sql
```

Todo el trabajo de este TP (triggers, pruebas de concurrencia, scripts generados por IA) se 
ejecuta sobre `copia_trabajo`, nunca sobre `plantilla_food_store` ni sobre ninguna base que 
tenga datos reales de producción.

## Paso 2 — Transacción

Todo script que escriba datos (INSERT, UPDATE, DELETE, o que agregue restricciones) se ejecuta 
primero dentro de una transacción abierta, para poder inspeccionar el efecto antes de confirmar 
nada:

```sql
BEGIN;

-- acá va el script generado por la IA o el cambio propio

-- se revisa: cuántas filas afectó, qué mensajes tiró, si el resultado
-- es el esperado

ROLLBACK; -- primero SIEMPRE se revierte para confirmar que se entendió el efecto
```

Recién cuando el efecto fue inspeccionado y es el esperado, se repite la operación terminando 
en `COMMIT` en lugar de `ROLLBACK`.

## Paso 3 — Respaldo

Antes de cualquier cambio estructural (ALTER, DROP, CREATE TRIGGER, CREATE FUNCTION, o cualquier 
migración), se saca un respaldo de la copia de trabajo, independiente del `ROLLBACK`:

```bash
pg_dump copia_trabajo > respaldos/copia_trabajo_YYYYMMDD_HHMM.sql
```

Los respaldos se guardan en la carpeta `respaldos/` del repo (o fuera del repo si el archivo 
es pesado), con fecha y hora en el nombre para poder identificar el punto exacto al que 
volver si algo sale mal.

## Regla de fondo

Ningún script generado por OpenCode o Kiro se ejecuta directamente sobre la base. El flujo 
siempre es:

1. Se pide el cambio a la IA en modo Plan (sin tocar archivos).
2. Se revisa el plan propuesto.
3. Se aplica y se lee el `git diff` completo, línea por línea.
4. Se prueba el efecto dentro de `BEGIN...ROLLBACK` sobre `copia_trabajo`.
5. Si el cambio es estructural, se saca respaldo antes del `COMMIT` final.
6. Recién ahí se hace `COMMIT` y se commitea el cambio en Git.

Ningún paso de esta lista se salta, incluso cuando el cambio parece trivial.