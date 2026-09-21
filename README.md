# Proyecto Escuelas OSM Argentina

Metodología y datos para mapear establecimientos educativos en OpenStreetMap,
cruzando fuentes oficiales (Ministerio de Educación / DGCyE / padrones
provinciales) con el estado actual de OSM, para armar challenges de
MapRoulette en encuentros y mapatones de la comunidad OSM Argentina.

Usado por primera vez en el Encuentro OSM Argentina 2025 (Luján) y adaptado
para Pergamino 2026. Pensado como modelo replicable para otras localidades.

## Estructura

Un subdirectorio por localidad/evento trabajado (ej. `pergamino/`), cada uno con:

- `escuelas_todas.geojson` — pool completo de tareas para el challenge de MapRoulette
- `escuelas_dia_evento.geojson` — subconjunto de prioridad alta (mapatón presencial)
- `escuelas_confirmadas.csv` / `escuelas_faltantes.csv` / `escuelas_a_revisar.csv` / `escuelas_verificar_terreno.csv` — resultado de la conciliación de datos
- `escuelas_instrucciones_challenge.md` — plantilla de instrucciones para pegar en MapRoulette
- fuentes crudas usadas en el cruce (`padron_educacion.xlsx`, `predios_dgcye.json`, `osm_sin_nombre.csv`, `escuelas_osm_snapshot.json`)

Ver el `CHANGELOG.md` de cada subdirectorio para la cronología de cada cruce
de datos (evitamos poner fechas en los nombres de archivo; el historial vive
en git y en el changelog).

## Metodología (resumen)

1. Obtener el padrón oficial de escuelas de la localidad (Ministerio/Provincia/Municipio).
2. Cruzar contra OSM vía Overpass — por CUE si está disponible, si no por nombre + distancia.
3. Clasificar cada escuela: confirmada / a revisar / faltante / a verificar en terreno.
4. Armar el GeoJSON de tareas para MapRoulette, con un link de verificación Overpass embebido por tarea (radio ~400m, corre automáticamente).
5. Priorizar un subconjunto (zona cercana a la sede) para el día del evento, dejando el resto en prioridad media/baja para que se complete después.
6. Usar un hashtag de changeset por evento (ej. `#PergaminoOSM2026`) en vez de un tag en el elemento, para no dejar en el mapa datos que solo tienen sentido para un evento puntual.

## Estado

| Localidad | Estado | Tareas pool | Prioridad día evento |
|---|---|---|---|
| Pergamino | Datos listos, challenge de MapRoulette **pendiente de crear** | 138 | 40 |
| Luján | Challenge completado (encuentro 2025) | 248 (218 + 30 Chajarí) | — |
