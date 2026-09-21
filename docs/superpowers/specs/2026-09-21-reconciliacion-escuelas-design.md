# Pipeline de reconciliación escuelas oficial ↔ OSM

## Contexto y objetivo

El cruce de datos oficiales (Ministerio de Educación / DGCyE) contra OSM que
dio origen a `pergamino/` se hizo esta sesión con herramientas ad-hoc, sin un
script reusable. Antes del Encuentro OSM Argentina 2026 puede hacer falta
re-correrlo (por ejemplo, si algún mapper ya cargó escuelas en el ínterin), y
además se quiere generalizar el mismo mecanismo para procesar Chajarí en vivo
(hoy solo existe un reprocesamiento del dataset viejo de 2025, usado como
challenge de control QA en MapRoulette).

Este documento diseña **solo la parte A** (pipeline de reconciliación). La
parte B (página/dashboard de comparación de resultados) es un proyecto
separado, a diseñar después, una vez que el formato de salida de A esté
estable.

**Disparo:** manual, a mano, antes del evento (`python3 reconciliar.py
--localidad pergamino`). No hay cron ni GitHub Actions — fuera de alcance por
ahora.

**Localidades cubiertas por este diseño:** Pergamino (con enriquecimiento
DGCyE) y Chajarí (sin enriquecimiento provincial, solo fuente nacional) — son
los dos casos reales que validan la generalización. No se diseña para
provincias hipotéticas sin caso de uso concreto.

## Layout de archivos

```
proyecto-escuelas-argentina/
  reconciliar.py                  # script único, parametrizado por --localidad
  adapters/
    __init__.py
    dgcye_buenos_aires.py         # enriquecimiento opcional, solo Buenos Aires
  pergamino/
    locality.yml
    overrides.csv
    escuelas_confirmadas.csv
    escuelas_a_revisar.csv
    escuelas_faltantes.csv
    escuelas_verificar_terreno.csv
    escuelas_todas.geojson
    escuelas_dia_evento.geojson
    CHANGELOG.md
  chajari/                        # NUEVO — corrida en vivo
    locality.yml
    overrides.csv
    (mismos archivos de salida que pergamino/)
    CHANGELOG.md
  chajari-control/                # SIN CAMBIOS — fixture de QA ya usado en el
                                   # challenge de MapRoulette de control, no lo
                                   # genera ni lo toca este pipeline
```

## `locality.yml`

```yaml
nombre: Pergamino
provincia: "Buenos Aires"
wfs_localidad_filter: "localidad ILIKE '%pergamino%'"
enriquecimiento: dgcye_buenos_aires   # nombre de módulo en adapters/, o null
sede_evento:                          # null si no hay evento en esa localidad
  lat: -33.900194
  lon: -60.563730
  radio_prioridad_m: 1000
```

Chajarí usa `enriquecimiento: null` y `sede_evento: null` (no hay evento ahí
actualmente; el propósito de esta corrida es validar el pipeline con datos
reales, no priorizar tareas para un mapatón).

## Stages de `reconciliar.py`

1. **`fetch_oficial_nacional(config)`** — WFS del Ministerio de Educación
   (`mapa.educacion.gob.ar/geoserver/mapa_interactivo/ows`, capa
   `mapa_interactivo:establecimientos`), filtrado con `cql_filter` según
   `wfs_localidad_filter`. Portado de
   `sandbox_claude/osm/core/argentina_schools_mapper_v2.py` (método
   `get_wfs_data`/`get_chajari_schools`, ya probado contra Chajarí en 2025).
   Normaliza cada feature a un registro común: `cue, nombre, sector, lat,
   lon`. `sector` (público/privado del padrón) se mapea directo a
   `operator:type` (`public`/`private`) en la salida.
   - **Paginado por lotes**: usa `count`/`startIndex` del WFS (como ya hacía
     el script viejo) en vez de un solo request — por ejemplo 200 features
     por lote. Si un lote falla, reintenta con backoff corto; si sigue
     fallando después de los reintentos, se sigue con los lotes restantes y
     el resumen final deja explícito qué rango no se pudo traer (nunca
     aborta todo el run por un lote puntual). Para las localidades de este
     diseño (Pergamino ~130, Chajarí ~30) probablemente entra en un solo
     lote, pero el mecanismo queda armado igual — es barato y evita el
     supuesto de "siempre entra en un request".

2. **`fetch_enriquecimiento(config, registros)`** — si `locality.yml` define
   un `enriquecimiento`, importa ese adaptador de `adapters/` y lo aplica
   (agrega/mejora campos como `ref:cui_dgcye`, `_domicilio_padron`,
   `_ambito_padron`, mejor precisión de coordenadas, y `isced:level` mapeado
   desde `nivel_oferta` del padrón). Si es `null` (caso Chajarí), no hay
   `ref:cui_dgcye` ni domicilio/ámbito, e `isced:level` se infiere por
   heurística de nombre (portada de `infer_education_level` del script
   viejo) — si no hay confianza, se omite el campo, nunca se inventa un
   valor (mismo criterio que ya documenta la plantilla de instrucciones).

3. **`fetch_osm_existente(registros)`** — por cada escuela, un query Overpass
   individual de radio 400m (el mismo formato ya validado y que el mapeador
   ve en la tarea — `nwr["amenity"~"school|kindergarten|college|university"]
   (around:400,lat,lon)`), no una descarga de todo el país.
   - **Lotes + checkpoint**: procesa en grupos chicos (ej. 10 escuelas) con
     una pausa corta entre lotes, para no saturar el servidor público de
     Overpass (ya sabemos que a veces da timeout por sobrecarga). Después de
     cada lote guarda un checkpoint (`.reconciliar_estado_<localidad>.json`,
     no versionado) con lo resuelto hasta ahí. Si el run se corta o un lote
     falla después de reintentar, la próxima corrida retoma desde el
     checkpoint en vez de volver a consultar escuelas ya resueltas.
   - Reintenta una vez con backoff corto si una consulta puntual da timeout;
     si sigue fallando, no asume nada — esa escuela sigue a clasificar con
     un flag `overpass_inconcluso=true`, pero el run **continúa** con el
     resto (nunca se aborta todo por una consulta puntual).

4. **`clasificar(registros)`** — por registro:
   - CUE exacto encontrado en el resultado de Overpass → `confirmada`
   - Coincidencia de nombre/distancia pero no exacta → `a_revisar`
   - `overpass_inconcluso=true` → `a_revisar` con nota "timeout, verificar
     manual"
   - Sin ningún resultado cercano → `faltante`

5. **Merge con `overrides.csv`** (ver sección siguiente) — se aplica después
   de clasificar, antes de generar salidas.

6. **`generar_salidas(registros, config)`** — reescribe los CSV por balde +
   `escuelas_todas.geojson` (todas las `faltante`, con el schema de
   propiedades ya usado en Pergamino: `id, amenity, name, ref:cue,
   ref:cui_dgcye?, source, official:legajo, operator:type?, isced:level?,
   _overpass_verificacion, _prioridad_dia_evento`) + `escuelas_dia_evento.geojson`
   (subconjunto dentro de `radio_prioridad_m` de `sede_evento`, si está
   configurada) + agrega una entrada fechada a `CHANGELOG.md` con el resumen
   de la corrida (cuántas en cada balde, qué cambió respecto a la corrida
   anterior).

## Mecanismo de merge (`overrides.csv`)

Columnas: `ref:cue,decision,nota`. `decision` ∈ `confirmada / faltante /
a_revisar / verificar_terreno / excluir`.

- El script **solo lee** `overrides.csv`, nunca escribe en él — es 100%
  input humano, versionado en git.
- Después de clasificar automáticamente, si una CUE aparece en
  `overrides.csv`, esa decisión gana siempre sobre la automática.
- `excluir` saca el registro de todas las salidas (para casos como
  duplicados reales entre padrones, detectados a mano).
- Regenerar todo (pasos 1-6) es siempre seguro de re-correr: el único
  archivo que un humano toca directamente es `overrides.csv`; todo lo demás
  se reescribe desde cero en cada corrida.

**Migración inicial:** las decisiones manuales que ya se tomaron esta sesión
para Pergamino (las 7 de `escuelas_a_revisar.csv` y las 10 de
`escuelas_verificar_terreno.csv`) se migran a `pergamino/overrides.csv` en la
primera implementación, para no perderlas cuando el script corra por primera
vez.

## Testing / validación

- Flag `--dry-run`: corre las etapas 1-5 pero en vez de escribir archivos
  imprime un resumen (cuántos registros cambiaron de balde respecto a la
  última corrida, cuántos son nuevos). Permite correrlo antes del evento sin
  comprometerse a los cambios.
- **Validación de regresión (Pergamino):** primera corrida real se compara
  contra la clasificación actual (hecha a mano esta sesión) — si coincide en
  su mayoría, valida la lógica antes de confiar en ella para la corrida en
  vivo de Chajarí.
- No hay suite de tests automatizada (no se justifica para un script de
  corrida manual, puntual, antes de un evento) — la validación es el
  dry-run + la comparación de regresión.

## Fuera de alcance (explícito)

- Parte B (dashboard/página de comparación de resultados) — proyecto
  separado, después de este.
- Automatización/cron — disparo manual únicamente, por ahora.
- Otras provincias/localidades sin caso de uso concreto — el diseño
  generaliza sobre Pergamino y Chajarí, no sobre hipotéticas futuras.
- Reprocesar `chajari-control/` — ese fixture ya cumplió su propósito (QA
  del challenge de MapRoulette) y no lo toca este pipeline.
