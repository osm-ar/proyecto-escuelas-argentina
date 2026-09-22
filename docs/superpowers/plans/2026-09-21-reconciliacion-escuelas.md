# Pipeline de reconciliación escuelas — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reemplazar el cruce ad-hoc de datos oficiales-vs-OSM por un script único reusable (`reconciliar.py --localidad <x>`), que genere los CSV/GeoJSON de Pergamino y, por primera vez, una corrida real en vivo para Chajarí.

**Architecture:** Módulos Python chicos y puros para cada transformación de datos (fuente oficial, geometría, enriquecimiento DGCyE, verificación Overpass, merge de overrides, generación de salidas), orquestados por un CLI (`reconciliar.py`) que hace todo el I/O (descargas, requests). Las funciones puras se testean con fixtures reales (capturadas de las fuentes en vivo esta sesión); el I/O se valida con `--dry-run` y una corrida de regresión contra los datos actuales de Pergamino.

**Tech Stack:** Python 3, `requests`, `openpyxl` (leer el padrón XLSX), `PyYAML` (leer `locality.yml`), `pytest`. Sin frameworks nuevos.

**Spec:** `docs/superpowers/specs/2026-09-21-reconciliacion-escuelas-design.md`

## Global Constraints

- Disparo manual únicamente (`python3 reconciliar.py --localidad pergamino`), sin cron/CI.
- Nunca abortar todo el run por una falla puntual (WFS por lotes, Overpass con checkpoint+reintento) — ver spec.
- `overrides.csv` es el único archivo que un humano edita a mano; el script nunca escribe en él.
- Nunca inventar un valor (coordenada, `isced:level`, `operator:type`) cuando no hay confianza — se omite o se manda a `verificar_terreno`.
- Representación interna canónica de un registro (usada por TODAS las funciones puras del pipeline), snake_case; los nombres con `:` (`ref:cue`, `ref:cui_dgcye`, `official:legajo`, `operator:type`, `isced:level`) solo aparecen en el paso final de generación de GeoJSON:

```python
# Registro interno canónico
{
    "cue": str,                    # Cueanexo del padrón — clave primaria
    "nombre": str,                 # nombre oficial (columna "Nombre" del padrón)
    "operator_type": str | None,   # "public" | "private" | None
    "ambito": str | None,          # "Urbano" | "Rural" | None
    "domicilio": str | None,
    "lat": float | None,
    "lon": float | None,
    "ref_cui_dgcye": str | None,   # solo si hubo enriquecimiento DGCyE
    "isced_level": str | None,
    "clasificacion": str | None,   # "confirmada" | "a_revisar" | "faltante" | "verificar_terreno"
    "nota": str | None,
    "prioridad_dia_evento": bool,
}
```
- Fuentes verificadas en vivo el 2026-09-21 (ver detalle en cada tarea): Padrón Oficial (`datos.gob.ar`), WFS `mapa.educacion.gob.ar`. El endpoint WFS/GeoServer específico de DGCyE no se pudo ubicar tras varios intentos — el adaptador de Pergamino lee el snapshot ya versionado `pergamino/predios_dgcye.json` en vez de una fuente en vivo (limitación documentada, no bloqueante).

---

### Task 1: Scaffolding del proyecto

**Files:**
- Create: `requirements.txt`
- Create: `locality.py`
- Create: `pergamino/locality.yml`
- Create: `chajari/locality.yml`
- Create: `tests/test_locality.py`
- Create: `tests/__init__.py`

**Interfaces:**
- Produces: `cargar_locality(path: str) -> dict` — lee y valida un `locality.yml`, devuelve el dict parseado con defaults aplicados (`enriquecimiento: None`, `sede_evento: None` si faltan).

- [ ] **Step 1: Crear `requirements.txt`**

```
requests>=2.31
openpyxl>=3.0
PyYAML>=6.0
pytest>=8.0
```

- [ ] **Step 2: Escribir el test de `cargar_locality`**

```python
# tests/test_locality.py
import textwrap
from pathlib import Path

from locality import cargar_locality


def test_carga_locality_completo(tmp_path):
    contenido = textwrap.dedent("""
        nombre: Pergamino
        provincia: "Buenos Aires"
        wfs_localidad_filter: PERGAMINO
        enriquecimiento: dgcye_buenos_aires
        sede_evento:
          lat: -33.900194
          lon: -60.563730
          radio_prioridad_m: 1000
    """)
    p = tmp_path / "locality.yml"
    p.write_text(contenido, encoding="utf-8")

    config = cargar_locality(str(p))

    assert config["nombre"] == "Pergamino"
    assert config["wfs_localidad_filter"] == "PERGAMINO"
    assert config["enriquecimiento"] == "dgcye_buenos_aires"
    assert config["sede_evento"]["radio_prioridad_m"] == 1000


def test_carga_locality_sin_enriquecimiento_ni_sede(tmp_path):
    contenido = textwrap.dedent("""
        nombre: Chajarí
        provincia: "Entre Ríos"
        wfs_localidad_filter: CHAJARI
    """)
    p = tmp_path / "locality.yml"
    p.write_text(contenido, encoding="utf-8")

    config = cargar_locality(str(p))

    assert config["enriquecimiento"] is None
    assert config["sede_evento"] is None
```

- [ ] **Step 3: Correr el test y verificar que falla**

Run: `python3 -m pytest tests/test_locality.py -v`
Expected: FAIL con `ModuleNotFoundError: No module named 'locality'`

- [ ] **Step 4: Implementar `locality.py`**

```python
# locality.py
import yaml


def cargar_locality(path: str) -> dict:
    with open(path, encoding="utf-8") as f:
        data = yaml.safe_load(f)
    data.setdefault("enriquecimiento", None)
    data.setdefault("sede_evento", None)
    return data
```

- [ ] **Step 5: Correr el test y verificar que pasa**

Run: `python3 -m pytest tests/test_locality.py -v`
Expected: PASS (2 tests)

- [ ] **Step 6: Crear `pergamino/locality.yml`**

```yaml
nombre: Pergamino
provincia: "Buenos Aires"
wfs_localidad_filter: PERGAMINO
enriquecimiento: dgcye_buenos_aires
sede_evento:
  lat: -33.900194
  lon: -60.563730
  radio_prioridad_m: 1000
```

- [ ] **Step 7: Crear `chajari/locality.yml`**

```yaml
nombre: Chajarí
provincia: "Entre Ríos"
wfs_localidad_filter: CHAJARI
enriquecimiento: null
sede_evento: null
```

- [ ] **Step 8: Commit**

```bash
touch tests/__init__.py
git add requirements.txt locality.py pergamino/locality.yml chajari/locality.yml tests/
git commit -m "feat: scaffolding y config por localidad (locality.yml)"
```

---

### Task 2: Fuente oficial — Padrón (atributos)

**Files:**
- Create: `fuente_oficial.py`
- Create: `tests/test_fuente_oficial.py`

**Interfaces:**
- Consumes: nada (primera pieza del pipeline)
- Produces:
  - `resolver_url_padron_mas_reciente(resources: list[dict]) -> str`
  - `ubicar_fila_header(filas: list[tuple]) -> int` — índice (0-based) de la fila que contiene "Cueanexo"
  - `parsear_padron(filas: list[tuple], header_idx: int) -> list[dict]` — filas ya leídas de la hoja (tal como las devuelve `openpyxl`), devuelve lista de registros canónicos (sin `lat`/`lon`, sin `clasificacion`)
  - `filtrar_por_localidad(registros: list[dict], termino: str, columna_localidad: str) -> list[dict]` — recibe también las filas crudas para poder filtrar por la columna `Localidad` que no forma parte del registro canónico; ver Step 4 para el detalle de por qué se pasa por separado

**Nota de diseño:** la columna `Localidad` del padrón no es parte del registro canónico (no se usa después de filtrar), así que `parsear_padron` devuelve, junto con la lista de registros, la lista paralela de localidades crudas para poder filtrar. Para mantenerlo simple, `parsear_padron` devuelve tuplas `(registro, localidad_cruda)`.

- [ ] **Step 1: Escribir los tests con datos reales capturados**

Estas filas son una copia literal de las que devuelve hoy el padrón oficial para Pergamino (verificado el 2026-09-21):

```python
# tests/test_fuente_oficial.py
from fuente_oficial import (
    resolver_url_padron_mas_reciente,
    ubicar_fila_header,
    parsear_padron,
    filtrar_por_localidad,
)

FILAS_CRUDAS = [
    ("Listado de establecimientos educativos y sus ofertas activas.",),
    ("Fuente: Padrón Oficial de Establecimientos Educativos",),
    (),
    ("Establecimiento - Localización",),
    (
        "Jurisdicción", "Sector", "Ámbito", "Departamento",
        "Código de departamento", "Localidad", "Código de localidad",
        "Cueanexo", "Nombre", "Domicilio", "C. P.", "Teléfono", "Mail",
        "Común",
    ),
    (
        "Buenos Aires", "Estatal", "Rural", "PERGAMINO", "06623",
        "PERGAMINO", "06623031", "060081900",
        "ESCUELA DE EDUCACIÓN PRIMARIA Nº57 FERMIN ORTIZ BASUALDO",
        "RUTA 188 Y CRUCE O.BASUALDO   ORTIZ BASUALDO", "2703",
        "02477 50-6565", "graballhorst13@abc.gob.ar", 1,
    ),
    (
        "Buenos Aires", "Privado", "Urbano", "PERGAMINO", "06623",
        "PERGAMINO", "06623031", "060147300",
        "INSTITUTO DE CAPACITACION Y DESARROLLO",
        "ITALIA 151  ", "2700", "02477 42-4455", "info@icade.edu.ar", 1,
    ),
    (
        "Entre Ríos", "Estatal", "Urbano", "FEDERACION", "06623",
        "CHAJARI", "06623031", "300014400",
        "ESCUELA PABLO PIZZURNO Nº9",
        "PARAJE LOS 14", "3228", "403140", None, 1,
    ),
]


def test_resolver_url_padron_toma_el_recurso_mas_reciente():
    resources = [
        {"url": "https://.../2023_padron.xlsx", "last_modified": "2024-08-01T00:00:00"},
        {"url": "https://.../2026.06.16_padron.xlsx", "last_modified": "2026-07-24T00:00:00"},
        {"url": "https://.../2025.09.24_padron.xlsx", "last_modified": "2026-05-21T00:00:00"},
    ]
    assert resolver_url_padron_mas_reciente(resources) == "https://.../2026.06.16_padron.xlsx"


def test_ubicar_fila_header():
    assert ubicar_fila_header(FILAS_CRUDAS) == 4


def test_parsear_padron_devuelve_registros_canonicos():
    resultado = parsear_padron(FILAS_CRUDAS, header_idx=4)

    assert len(resultado) == 3
    reg, localidad = resultado[0]
    assert reg["cue"] == "060081900"
    assert reg["nombre"] == "ESCUELA DE EDUCACIÓN PRIMARIA Nº57 FERMIN ORTIZ BASUALDO"
    assert reg["operator_type"] == "public"
    assert reg["ambito"] == "Rural"
    assert localidad == "PERGAMINO"

    reg_privado, _ = resultado[1]
    assert reg_privado["operator_type"] == "private"


def test_filtrar_por_localidad_es_insensible_a_mayusculas():
    registros = parsear_padron(FILAS_CRUDAS, header_idx=4)
    filtrados = filtrar_por_localidad(registros, "pergamino")

    assert len(filtrados) == 2
    assert all(r["cue"].startswith("06") for r in filtrados)


def test_filtrar_por_localidad_chajari():
    registros = parsear_padron(FILAS_CRUDAS, header_idx=4)
    filtrados = filtrar_por_localidad(registros, "chajari")

    assert len(filtrados) == 1
    assert filtrados[0]["cue"] == "300014400"
```

- [ ] **Step 2: Correr los tests y verificar que fallan**

Run: `python3 -m pytest tests/test_fuente_oficial.py -v`
Expected: FAIL con `ModuleNotFoundError: No module named 'fuente_oficial'`

- [ ] **Step 3: Implementar `fuente_oficial.py` (parte padrón)**

```python
# fuente_oficial.py
import time

import openpyxl
import requests

SECTOR_A_OPERATOR_TYPE = {"Estatal": "public", "Privado": "private"}

CKAN_PACKAGE_SHOW = (
    "https://www.datos.gob.ar/api/3/action/package_show"
    "?id=padron-oficial-de-establecimientos-educativos"
)


def resolver_url_padron_mas_reciente(resources: list[dict]) -> str:
    mas_reciente = max(resources, key=lambda r: r["last_modified"])
    return mas_reciente["url"]


def ubicar_fila_header(filas: list[tuple]) -> int:
    for i, fila in enumerate(filas):
        if fila and "Cueanexo" in fila:
            return i
    raise ValueError("No se encontró la fila de encabezado (columna 'Cueanexo')")


def parsear_padron(filas: list[tuple], header_idx: int) -> list[tuple[dict, str]]:
    header = filas[header_idx]
    idx = {nombre: i for i, nombre in enumerate(header)}

    resultado = []
    for fila in filas[header_idx + 1:]:
        if not fila or not fila[idx["Cueanexo"]]:
            continue
        registro = {
            "cue": str(fila[idx["Cueanexo"]]),
            "nombre": fila[idx["Nombre"]],
            "operator_type": SECTOR_A_OPERATOR_TYPE.get(fila[idx["Sector"]]),
            "ambito": fila[idx["Ámbito"]],
            "domicilio": fila[idx["Domicilio"]],
            "lat": None,
            "lon": None,
            "ref_cui_dgcye": None,
            "isced_level": None,
            "clasificacion": None,
            "nota": None,
            "prioridad_dia_evento": False,
        }
        localidad = fila[idx["Localidad"]]
        resultado.append((registro, localidad))
    return resultado


def filtrar_por_localidad(
    registros: list[tuple[dict, str]], termino: str
) -> list[dict]:
    termino = termino.upper()
    return [reg for reg, localidad in registros if termino in (localidad or "").upper()]


def descargar_padron_bytes(url: str, intentos: int = 3) -> bytes:
    for intento in range(intentos):
        try:
            r = requests.get(url, timeout=60)
            r.raise_for_status()
            return r.content
        except requests.RequestException:
            if intento == intentos - 1:
                raise
            time.sleep(2 * (intento + 1))


def fetch_padron_localidad(termino_localidad: str) -> list[dict]:
    """Descarga el padrón más reciente y devuelve los registros de una localidad."""
    meta = requests.get(CKAN_PACKAGE_SHOW, timeout=30).json()
    url = resolver_url_padron_mas_reciente(meta["result"]["resources"])
    contenido = descargar_padron_bytes(url)

    import io

    wb = openpyxl.load_workbook(io.BytesIO(contenido), read_only=True)
    ws = wb[wb.sheetnames[0]]
    filas = list(ws.iter_rows(values_only=True))

    header_idx = ubicar_fila_header(filas)
    registros = parsear_padron(filas, header_idx)
    return filtrar_por_localidad(registros, termino_localidad)
```

- [ ] **Step 4: Correr los tests y verificar que pasan**

Run: `python3 -m pytest tests/test_fuente_oficial.py -v`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git add fuente_oficial.py tests/test_fuente_oficial.py
git commit -m "feat: fetch y parseo del Padrón Oficial de Establecimientos Educativos"
```

---

### Task 3: Fuente oficial — Geometría (WFS)

**Files:**
- Modify: `fuente_oficial.py`
- Modify: `tests/test_fuente_oficial.py`

**Interfaces:**
- Consumes: registros canónicos de Task 2 (con `lat`/`lon` en `None`)
- Produces:
  - `parsear_features_wfs(geojson: dict) -> dict[str, tuple[float, float]]` — mapa `cue -> (lat, lon)`
  - `cruzar_geometria(registros: list[dict], geometrias: dict) -> tuple[list[dict], list[dict]]` — `(con_geometria, sin_geometria)`; los `sin_geometria` quedan con `clasificacion="verificar_terreno"` y `nota` explicando por qué
  - `fetch_geometria_localidad(termino_localidad: str, lote: int = 200) -> dict[str, tuple[float, float]]` — pagina el WFS con `count`/`startIndex`, reintenta por lote

Fixture real capturada el 2026-09-21 contra `mapa.educacion.gob.ar` para Chajarí (usada en los tests):

```python
WFS_RESPUESTA_CHAJARI = {
    "totalFeatures": 2,
    "features": [
        {
            "geometry": {"type": "MultiPoint", "coordinates": [[-58.023706909063, -30.736083483106]]},
            "properties": {"cueanexo": "300014400", "localidad": "CHAJARI"},
        },
        {
            "geometry": {"type": "MultiPoint", "coordinates": [[-58.01, -30.74]]},
            "properties": {"cueanexo": "300028600", "localidad": "CHAJARI"},
        },
    ],
}
```

- [ ] **Step 1: Agregar los tests de geometría**

```python
# agregar a tests/test_fuente_oficial.py
from fuente_oficial import parsear_features_wfs, cruzar_geometria

WFS_RESPUESTA_CHAJARI = {
    "totalFeatures": 2,
    "features": [
        {
            "geometry": {"type": "MultiPoint", "coordinates": [[-58.023706909063, -30.736083483106]]},
            "properties": {"cueanexo": "300014400", "localidad": "CHAJARI"},
        },
        {
            "geometry": {"type": "MultiPoint", "coordinates": [[-58.01, -30.74]]},
            "properties": {"cueanexo": "300028600", "localidad": "CHAJARI"},
        },
    ],
}


def test_parsear_features_wfs():
    mapa = parsear_features_wfs(WFS_RESPUESTA_CHAJARI)

    assert mapa["300014400"] == (-30.736083483106, -58.023706909063)
    assert mapa["300028600"] == (-30.74, -58.01)


def test_cruzar_geometria_separa_los_que_no_matchean():
    registros = [
        {"cue": "300014400", "nombre": "A", "lat": None, "lon": None, "clasificacion": None, "nota": None},
        {"cue": "999999999", "nombre": "B", "lat": None, "lon": None, "clasificacion": None, "nota": None},
    ]
    geometrias = parsear_features_wfs(WFS_RESPUESTA_CHAJARI)

    con_geo, sin_geo = cruzar_geometria(registros, geometrias)

    assert len(con_geo) == 1
    assert con_geo[0]["lat"] == -30.736083483106
    assert len(sin_geo) == 1
    assert sin_geo[0]["clasificacion"] == "verificar_terreno"
    assert "sin coordenadas" in sin_geo[0]["nota"]
```

- [ ] **Step 2: Correr y verificar que fallan**

Run: `python3 -m pytest tests/test_fuente_oficial.py -v`
Expected: FAIL con `ImportError: cannot import name 'parsear_features_wfs'`

- [ ] **Step 3: Agregar a `fuente_oficial.py`**

```python
# agregar a fuente_oficial.py
WFS_BASE_URL = "https://mapa.educacion.gob.ar/geoserver/mapa_interactivo/ows"


def parsear_features_wfs(geojson: dict) -> dict[str, tuple[float, float]]:
    mapa = {}
    for f in geojson.get("features", []):
        cue = f["properties"].get("cueanexo")
        coords = f.get("geometry", {}).get("coordinates")
        if not cue or not coords:
            continue
        lon, lat = coords[0] if isinstance(coords[0], list) else coords
        mapa[str(cue)] = (lat, lon)
    return mapa


def cruzar_geometria(
    registros: list[dict], geometrias: dict[str, tuple[float, float]]
) -> tuple[list[dict], list[dict]]:
    con_geometria, sin_geometria = [], []
    for reg in registros:
        coords = geometrias.get(reg["cue"])
        if coords:
            reg["lat"], reg["lon"] = coords
            con_geometria.append(reg)
        else:
            reg["clasificacion"] = "verificar_terreno"
            reg["nota"] = "sin coordenadas en WFS, ubicar manualmente"
            sin_geometria.append(reg)
    return con_geometria, sin_geometria


def fetch_geometria_localidad(termino_localidad: str, lote: int = 200) -> dict:
    geometrias = {}
    offset = 0
    while True:
        params = {
            "service": "WFS",
            "version": "2.0.0",
            "request": "GetFeature",
            "typeName": "mapa_interactivo:establecimientos",
            "outputFormat": "application/json",
            "cql_filter": f"localidad ILIKE '%{termino_localidad}%'",
            "count": lote,
            "startIndex": offset,
        }
        data = _get_con_reintento(WFS_BASE_URL, params)
        if data is None:
            print(f"⚠️  No se pudo traer el lote WFS offset={offset}, se sigue con lo que hay")
            break
        pagina = parsear_features_wfs(data)
        geometrias.update(pagina)
        if len(data.get("features", [])) < lote:
            break
        offset += lote
    return geometrias


def _get_con_reintento(url: str, params: dict, intentos: int = 3) -> dict | None:
    for intento in range(intentos):
        try:
            r = requests.get(url, params=params, timeout=30)
            r.raise_for_status()
            return r.json()
        except requests.RequestException:
            if intento == intentos - 1:
                return None
            time.sleep(2 * (intento + 1))
```

- [ ] **Step 4: Correr y verificar que pasan**

Run: `python3 -m pytest tests/test_fuente_oficial.py -v`
Expected: PASS (7 tests)

- [ ] **Step 5: Verificación manual contra el servicio real**

Run:
```bash
python3 -c "
from fuente_oficial import fetch_geometria_localidad
g = fetch_geometria_localidad('CHAJARI')
print(len(g), 'escuelas con geometria')
print(list(g.items())[:3])
"
```
Expected: imprime más de 30 resultados (ya se verificó el 2026-09-21 que da 44 `totalFeatures` para Chajarí).

- [ ] **Step 6: Commit**

```bash
git add fuente_oficial.py tests/test_fuente_oficial.py
git commit -m "feat: cruce de geometria via WFS del Ministerio, paginado y con reintento"
```

---

### Task 4: Adaptador DGCyE (enriquecimiento Pergamino)

**Files:**
- Create: `adapters/__init__.py`
- Create: `adapters/dgcye_buenos_aires.py`
- Create: `tests/test_dgcye_adapter.py`

**Interfaces:**
- Consumes: registros canónicos de Task 2+3 (con `lat`/`lon` ya asignados)
- Produces: `enriquecer(registros: list[dict], snapshot_path: str) -> list[dict]`

**Nota de diseño (limitación documentada):** no se encontró un endpoint WFS/GeoServer público del DGCyE tras varios intentos (ver spec). Este adaptador lee el snapshot ya versionado `pergamino/predios_dgcye.json` (128 features, `cue_clave` con formato `"<CUE>-<sufijo>"`) en vez de una fuente en vivo. Queda documentado como limitación conocida, no bloquea el resto del pipeline.

- [ ] **Step 1: Escribir el test con una muestra real del snapshot**

```python
# tests/test_dgcye_adapter.py
import json

from adapters.dgcye_buenos_aires import enriquecer

SNAPSHOT_MUESTRA = {
    "type": "FeatureCollection",
    "features": [
        {
            "properties": {
                "cui": 608100020,
                "distrito": "Pergamino",
                "region_educativa": "Región 13",
                "cue_clave": "061514000-0081AE0001",
                "latitud": -33.89916498281925,
                "longitud": -60.57261631923284,
            }
        }
    ],
}


def test_enriquecer_agrega_cui_y_mejora_coordenadas(tmp_path):
    snapshot_path = tmp_path / "predios_dgcye.json"
    snapshot_path.write_text(json.dumps(SNAPSHOT_MUESTRA), encoding="utf-8")

    registros = [
        {"cue": "061514000", "lat": -33.9, "lon": -60.57, "ref_cui_dgcye": None},
        {"cue": "999999999", "lat": -30.0, "lon": -58.0, "ref_cui_dgcye": None},
    ]

    resultado = enriquecer(registros, str(snapshot_path))

    enriquecido = next(r for r in resultado if r["cue"] == "061514000")
    assert enriquecido["ref_cui_dgcye"] == "608100020"
    assert enriquecido["lat"] == -33.89916498281925
    assert enriquecido["lon"] == -60.57261631923284

    no_enriquecido = next(r for r in resultado if r["cue"] == "999999999")
    assert no_enriquecido["ref_cui_dgcye"] is None
```

- [ ] **Step 2: Correr y verificar que falla**

Run: `python3 -m pytest tests/test_dgcye_adapter.py -v`
Expected: FAIL con `ModuleNotFoundError: No module named 'adapters'`

- [ ] **Step 3: Implementar el adaptador**

```python
# adapters/__init__.py
```

```python
# adapters/dgcye_buenos_aires.py
"""Enriquecimiento con datos de predios educativos de la DGCyE (Buenos Aires).

LIMITACION CONOCIDA: no se encontro un endpoint WFS/GeoServer publico del
DGCyE (ver docs/superpowers/specs/2026-09-21-reconciliacion-escuelas-design.md).
Este adaptador lee un snapshot ya versionado en vez de consultar en vivo.
"""
import json


def enriquecer(registros: list[dict], snapshot_path: str) -> list[dict]:
    with open(snapshot_path, encoding="utf-8") as f:
        snapshot = json.load(f)

    por_cue = {}
    for feature in snapshot["features"]:
        props = feature["properties"]
        cue = str(props["cue_clave"]).split("-")[0]
        por_cue[cue] = props

    for reg in registros:
        props = por_cue.get(reg["cue"])
        if not props:
            continue
        reg["ref_cui_dgcye"] = str(props["cui"])
        reg["lat"] = props["latitud"]
        reg["lon"] = props["longitud"]

    return registros
```

- [ ] **Step 4: Correr y verificar que pasa**

Run: `python3 -m pytest tests/test_dgcye_adapter.py -v`
Expected: PASS (1 test)

- [ ] **Step 5: Verificación manual contra el snapshot real**

Run:
```bash
python3 -c "
from adapters.dgcye_buenos_aires import enriquecer
registros = [{'cue': '060081900', 'lat': 0, 'lon': 0, 'ref_cui_dgcye': None}]
r = enriquecer(registros, 'pergamino/predios_dgcye.json')
print(r)
"
```
Expected: si `060081900` está en el snapshot, se ve `ref_cui_dgcye` seteado y lat/lon reales de Pergamino.

- [ ] **Step 6: Commit**

```bash
git add adapters/ tests/test_dgcye_adapter.py
git commit -m "feat: adaptador de enriquecimiento DGCyE (lee snapshot versionado)"
```

---

### Task 5: Verificación Overpass y clasificación

**Files:**
- Create: `overpass_check.py`
- Create: `tests/test_overpass_check.py`

**Interfaces:**
- Consumes: registros con `lat`/`lon` ya asignados (de Tasks 2-4)
- Produces:
  - `construir_query_overpass(lat: float, lon: float, radio_m: int = 400) -> str`
  - `construir_link_overpass_turbo(query: str, lat: float, lon: float) -> str`
  - `clasificar_registro(cue: str, nombre: str, elementos_cercanos: list[dict]) -> tuple[str, str | None]` — devuelve `(clasificacion, nota)`
  - `fetch_overpass_con_reintento(lat: float, lon: float, consultar_fn=None) -> list[dict] | None` — `consultar_fn` inyectable para tests; `None` en el resultado significa que agotó los reintentos (inconcluso)
  - `verificar_localidad(registros: list[dict], checkpoint_path: str) -> list[dict]` — recorre todos, pausa entre consultas, guarda checkpoint cada 10, retoma si ya existe

- [ ] **Step 1: Escribir los tests**

```python
# tests/test_overpass_check.py
import json

from overpass_check import (
    construir_query_overpass,
    construir_link_overpass_turbo,
    clasificar_registro,
    fetch_overpass_con_reintento,
)


def test_construir_query_overpass_formato_exacto():
    query = construir_query_overpass(-33.897554465256, -60.56641799211592)
    assert query == (
        "[out:json][timeout:25];\n(\n  "
        'nwr["amenity"~"school|kindergarten|college|university"]'
        "(around:400,-33.897554465256,-60.56641799211592);\n);\nout center tags;"
    )


def test_construir_link_overpass_turbo_coincide_con_el_ya_validado():
    query = construir_query_overpass(-33.897554465256, -60.56641799211592)
    link = construir_link_overpass_turbo(query, -33.897554465256, -60.56641799211592)

    # Este es el link real que se probó y funcionó en MapRoulette (2026-09-20).
    esperado = (
        "https://overpass-turbo.eu/?Q=%5Bout%3Ajson%5D%5Btimeout%3A25%5D%3B%0A%28%0A"
        "%20%20nwr%5B%22amenity%22~%22school%7Ckindergarten%7Ccollege%7Cuniversity%22%5D"
        "%28around%3A400%2C-33.897554465256%2C-60.56641799211592%29%3B%0A%29%3B%0A"
        "out+center+tags%3B&C=-33.897554465256%3B-60.56641799211592%3B17&R="
    )
    assert link == esperado


def test_clasificar_confirmada_por_cue_exacto():
    elementos = [{"tags": {"ref:cue": "060081900", "amenity": "school"}}]
    clasificacion, nota = clasificar_registro("060081900", "ESCUELA N 57", elementos)
    assert clasificacion == "confirmada"


def test_clasificar_a_revisar_por_nombre_similar():
    elementos = [{"tags": {"name": "Escuela N57 Fermin Ortiz", "amenity": "school"}}]
    clasificacion, nota = clasificar_registro(
        "060081900", "ESCUELA DE EDUCACIÓN PRIMARIA Nº57 FERMIN ORTIZ BASUALDO", elementos
    )
    assert clasificacion == "a_revisar"


def test_clasificar_faltante_sin_elementos_cercanos():
    clasificacion, nota = clasificar_registro("060081900", "ESCUELA N 57", [])
    assert clasificacion == "faltante"


def test_fetch_overpass_con_reintento_reintenta_y_despues_funciona():
    llamadas = {"n": 0}

    def consultar_fn(lat, lon):
        llamadas["n"] += 1
        if llamadas["n"] < 2:
            raise TimeoutError("simulated timeout")
        return [{"tags": {"amenity": "school"}}]

    resultado = fetch_overpass_con_reintento(-33.9, -60.5, consultar_fn=consultar_fn)

    assert llamadas["n"] == 2
    assert resultado == [{"tags": {"amenity": "school"}}]


def test_fetch_overpass_con_reintento_agota_reintentos():
    def consultar_fn(lat, lon):
        raise TimeoutError("simulated timeout")

    resultado = fetch_overpass_con_reintento(-33.9, -60.5, consultar_fn=consultar_fn)

    assert resultado is None
```

- [ ] **Step 2: Correr y verificar que fallan**

Run: `python3 -m pytest tests/test_overpass_check.py -v`
Expected: FAIL con `ModuleNotFoundError: No module named 'overpass_check'`

- [ ] **Step 3: Implementar `overpass_check.py`**

```python
# overpass_check.py
import json
import os
import time
import urllib.parse
from difflib import SequenceMatcher

import requests

OVERPASS_API_URL = "https://overpass-api.de/api/interpreter"


def construir_query_overpass(lat: float, lon: float, radio_m: int = 400) -> str:
    return (
        "[out:json][timeout:25];\n(\n  "
        f'nwr["amenity"~"school|kindergarten|college|university"]'
        f"(around:{radio_m},{lat},{lon});\n);\nout center tags;"
    )


def construir_link_overpass_turbo(query: str, lat: float, lon: float) -> str:
    params = {"Q": query, "C": f"{lat};{lon};17", "R": ""}
    return "https://overpass-turbo.eu/?" + urllib.parse.urlencode(params)


def clasificar_registro(
    cue: str, nombre: str, elementos_cercanos: list[dict], umbral_nombre: float = 0.6
) -> tuple[str, str | None]:
    if not elementos_cercanos:
        return "faltante", None

    for el in elementos_cercanos:
        if el.get("tags", {}).get("ref:cue") == cue:
            return "confirmada", None

    nombre_norm = nombre.lower()
    for el in elementos_cercanos:
        nombre_osm = el.get("tags", {}).get("name", "")
        score = SequenceMatcher(None, nombre_norm, nombre_osm.lower()).ratio()
        if score >= umbral_nombre:
            return "a_revisar", f"nombre similar en OSM: '{nombre_osm}' (score={score:.2f})"

    return "a_revisar", "hay elementos educativos cerca pero ningún nombre similar"


def _consultar_overpass_real(lat: float, lon: float) -> list[dict]:
    query = construir_query_overpass(lat, lon)
    r = requests.post(OVERPASS_API_URL, data={"data": query}, timeout=30)
    r.raise_for_status()
    return r.json().get("elements", [])


def fetch_overpass_con_reintento(
    lat: float, lon: float, consultar_fn=None, intentos: int = 2
) -> list[dict] | None:
    consultar_fn = consultar_fn or _consultar_overpass_real
    for intento in range(intentos):
        try:
            return consultar_fn(lat, lon)
        except Exception:
            if intento == intentos - 1:
                return None
            time.sleep(3)
    return None


def verificar_localidad(
    registros: list[dict], checkpoint_path: str, pausa_s: float = 1.5, guardar_cada: int = 10
) -> list[dict]:
    resueltos = {}
    if os.path.exists(checkpoint_path):
        with open(checkpoint_path, encoding="utf-8") as f:
            resueltos = json.load(f)

    for i, reg in enumerate(registros):
        if reg["cue"] in resueltos:
            reg["clasificacion"], reg["nota"] = resueltos[reg["cue"]]
            continue

        elementos = fetch_overpass_con_reintento(reg["lat"], reg["lon"])
        if elementos is None:
            reg["clasificacion"] = "a_revisar"
            reg["nota"] = "timeout de Overpass, verificar manual"
        else:
            reg["clasificacion"], reg["nota"] = clasificar_registro(
                reg["cue"], reg["nombre"], elementos
            )

        resueltos[reg["cue"]] = (reg["clasificacion"], reg["nota"])
        time.sleep(pausa_s)

        if (i + 1) % guardar_cada == 0:
            with open(checkpoint_path, "w", encoding="utf-8") as f:
                json.dump(resueltos, f)

    with open(checkpoint_path, "w", encoding="utf-8") as f:
        json.dump(resueltos, f)

    return registros
```

- [ ] **Step 4: Correr y verificar que pasan**

Run: `python3 -m pytest tests/test_overpass_check.py -v`
Expected: PASS (7 tests)

- [ ] **Step 5: Commit**

```bash
git add overpass_check.py tests/test_overpass_check.py
git commit -m "feat: verificacion Overpass (query, clasificacion, checkpoint reanudable)"
```

---

### Task 6: Overrides (merge de decisiones manuales)

**Files:**
- Create: `overrides.py`
- Create: `tests/test_overrides.py`

**Interfaces:**
- Consumes: registros ya clasificados (Task 5)
- Produces:
  - `cargar_overrides(path: str) -> dict[str, dict]` — `cue -> {"decision": str, "nota": str}`
  - `aplicar_overrides(registros: list[dict], overrides: dict) -> list[dict]` — los `excluir` se sacan de la lista

- [ ] **Step 1: Escribir los tests**

```python
# tests/test_overrides.py
import csv

from overrides import cargar_overrides, aplicar_overrides


def test_cargar_overrides(tmp_path):
    p = tmp_path / "overrides.csv"
    with open(p, "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["ref:cue", "decision", "nota"])
        w.writerow(["060081900", "confirmada", "verificado en terreno el 20/09"])
        w.writerow(["060999999", "excluir", "duplicado con otro CUE"])

    overrides = cargar_overrides(str(p))

    assert overrides["060081900"]["decision"] == "confirmada"
    assert overrides["060999999"]["decision"] == "excluir"


def test_aplicar_overrides_pisa_la_clasificacion_automatica():
    registros = [
        {"cue": "060081900", "clasificacion": "faltante", "nota": None},
        {"cue": "111", "clasificacion": "faltante", "nota": None},
    ]
    overrides = {"060081900": {"decision": "confirmada", "nota": "verificado en terreno"}}

    resultado = aplicar_overrides(registros, overrides)

    reg = next(r for r in resultado if r["cue"] == "060081900")
    assert reg["clasificacion"] == "confirmada"
    assert reg["nota"] == "verificado en terreno"


def test_aplicar_overrides_excluir_saca_el_registro():
    registros = [
        {"cue": "060081900", "clasificacion": "faltante", "nota": None},
        {"cue": "999", "clasificacion": "faltante", "nota": None},
    ]
    overrides = {"999": {"decision": "excluir", "nota": "duplicado"}}

    resultado = aplicar_overrides(registros, overrides)

    assert len(resultado) == 1
    assert resultado[0]["cue"] == "060081900"
```

- [ ] **Step 2: Correr y verificar que fallan**

Run: `python3 -m pytest tests/test_overrides.py -v`
Expected: FAIL con `ModuleNotFoundError: No module named 'overrides'`

- [ ] **Step 3: Implementar `overrides.py`**

```python
# overrides.py
import csv
import os


def cargar_overrides(path: str) -> dict[str, dict]:
    if not os.path.exists(path):
        return {}
    overrides = {}
    with open(path, encoding="utf-8") as f:
        for fila in csv.DictReader(f):
            overrides[fila["ref:cue"]] = {"decision": fila["decision"], "nota": fila["nota"]}
    return overrides


def aplicar_overrides(registros: list[dict], overrides: dict) -> list[dict]:
    resultado = []
    for reg in registros:
        override = overrides.get(reg["cue"])
        if override:
            if override["decision"] == "excluir":
                continue
            reg["clasificacion"] = override["decision"]
            reg["nota"] = override["nota"]
        resultado.append(reg)
    return resultado
```

- [ ] **Step 4: Correr y verificar que pasan**

Run: `python3 -m pytest tests/test_overrides.py -v`
Expected: PASS (3 tests)

- [ ] **Step 5: Migrar las decisiones manuales ya tomadas para Pergamino**

Crear `pergamino/overrides.csv` migrando `pergamino/escuelas_a_revisar.csv` (7 filas, columna `cue`) y `pergamino/escuelas_verificar_terreno.csv` (10 filas, columnas `cue_a`/`cue_b`, cada par entra dos veces, una por cada CUE):

```bash
python3 -c "
import csv

filas_salida = [('ref:cue', 'decision', 'nota')]

with open('pergamino/escuelas_a_revisar.csv', encoding='utf-8') as f:
    for r in csv.DictReader(f):
        nota = f\"posible match OSM: {r['osm_name']} (dist={r['dist_m']}m, score={r['name_score']})\"
        filas_salida.append((r['cue'], 'a_revisar', nota))

with open('pergamino/escuelas_verificar_terreno.csv', encoding='utf-8') as f:
    for r in csv.DictReader(f):
        nota = r['nota']
        filas_salida.append((r['cue_a'], 'verificar_terreno', nota))
        filas_salida.append((r['cue_b'], 'verificar_terreno', nota))

with open('pergamino/overrides.csv', 'w', newline='', encoding='utf-8') as f:
    csv.writer(f).writerows(filas_salida)

print(f'{len(filas_salida) - 1} overrides migrados')
"
```

Expected: imprime `24 overrides migrados` (7 + 2×10, menos algún duplicado de CUE si comparte fila).

- [ ] **Step 6: Crear `chajari/overrides.csv` vacío (solo header)**

```bash
mkdir -p chajari
echo "ref:cue,decision,nota" > chajari/overrides.csv
```

- [ ] **Step 7: Commit**

```bash
git add overrides.py tests/test_overrides.py pergamino/overrides.csv chajari/overrides.csv
git commit -m "feat: mecanismo de overrides + migrar decisiones manuales de Pergamino"
```

---

### Task 7: Generación de salidas (CSV, GeoJSON, CHANGELOG)

**Files:**
- Create: `salidas.py`
- Create: `tests/test_salidas.py`

**Interfaces:**
- Consumes: registros finales (post-overrides) de Task 6
- Produces:
  - `construir_feature_geojson(registro: dict) -> dict`
  - `filtrar_dia_evento(registros: list[dict], sede: dict) -> list[dict]` — distancia haversine
  - `escribir_salidas(registros: list[dict], carpeta: str, sede_evento: dict | None) -> dict` — escribe todos los CSV + los dos GeoJSON, devuelve un resumen (conteos por balde) para el CHANGELOG y el modo `--dry-run`
  - `agregar_entrada_changelog(carpeta: str, resumen: dict) -> None`

- [ ] **Step 1: Escribir los tests**

```python
# tests/test_salidas.py
import json

from salidas import construir_feature_geojson, filtrar_dia_evento, escribir_salidas


def _registro(**overrides):
    base = {
        "cue": "060081900",
        "nombre": "ESCUELA N 57",
        "operator_type": "public",
        "ambito": "Rural",
        "domicilio": "RUTA 188",
        "lat": -33.9,
        "lon": -60.57,
        "ref_cui_dgcye": "608100020",
        "isced_level": "1",
        "clasificacion": "faltante",
        "nota": None,
        "prioridad_dia_evento": False,
    }
    base.update(overrides)
    return base


def test_construir_feature_geojson_usa_los_nombres_de_tag_con_dos_puntos():
    feature = construir_feature_geojson(_registro())

    props = feature["properties"]
    assert props["amenity"] == "school"
    assert props["name"] == "ESCUELA N 57"
    assert props["ref:cue"] == "060081900"
    assert props["ref:cui_dgcye"] == "608100020"
    assert props["operator:type"] == "public"
    assert props["isced:level"] == "1"
    assert "_overpass_verificacion" in props
    assert feature["geometry"]["coordinates"] == [-60.57, -33.9]


def test_construir_feature_geojson_omite_isced_level_si_no_hay_confianza():
    feature = construir_feature_geojson(_registro(isced_level=None))
    assert "isced:level" not in feature["properties"]


def test_filtrar_dia_evento_usa_radio():
    sede = {"lat": -33.900194, "lon": -60.563730, "radio_prioridad_m": 1000}
    cerca = _registro(cue="1", lat=-33.900194, lon=-60.563730)
    lejos = _registro(cue="2", lat=-34.5, lon=-61.0)

    resultado = filtrar_dia_evento([cerca, lejos], sede)

    assert len(resultado) == 1
    assert resultado[0]["cue"] == "1"


def test_escribir_salidas_genera_todos_los_archivos(tmp_path):
    registros = [
        _registro(cue="1", clasificacion="confirmada"),
        _registro(cue="2", clasificacion="faltante"),
        _registro(cue="3", clasificacion="a_revisar"),
        _registro(cue="4", clasificacion="verificar_terreno", lat=None, lon=None),
    ]

    resumen = escribir_salidas(registros, str(tmp_path), sede_evento=None)

    assert (tmp_path / "escuelas_confirmadas.csv").exists()
    assert (tmp_path / "escuelas_faltantes.csv").exists()
    assert (tmp_path / "escuelas_a_revisar.csv").exists()
    assert (tmp_path / "escuelas_verificar_terreno.csv").exists()
    assert (tmp_path / "escuelas_todas.geojson").exists()

    todas = json.loads((tmp_path / "escuelas_todas.geojson").read_text())
    assert len(todas["features"]) == 1  # solo la "faltante"

    assert resumen == {"confirmada": 1, "faltante": 1, "a_revisar": 1, "verificar_terreno": 1}
```

- [ ] **Step 2: Correr y verificar que fallan**

Run: `python3 -m pytest tests/test_salidas.py -v`
Expected: FAIL con `ModuleNotFoundError: No module named 'salidas'`

- [ ] **Step 3: Implementar `salidas.py`**

```python
# salidas.py
import csv
import datetime
import json
import math
import os

from overpass_check import construir_link_overpass_turbo, construir_query_overpass

CAMPOS_CSV = [
    "cue", "nombre", "operator_type", "ambito", "domicilio",
    "lat", "lon", "ref_cui_dgcye", "isced_level", "clasificacion", "nota",
]


def construir_feature_geojson(registro: dict) -> dict:
    query = construir_query_overpass(registro["lat"], registro["lon"])
    props = {
        "id": f"escuela_{registro['cue']}",
        "amenity": "school",
        "name": registro["nombre"],
        "ref:cue": registro["cue"],
        "source": "Padrón Oficial de Establecimientos Educativos",
        "_overpass_verificacion": construir_link_overpass_turbo(
            query, registro["lat"], registro["lon"]
        ),
        "_prioridad_dia_evento": registro["prioridad_dia_evento"],
    }
    if registro.get("ref_cui_dgcye"):
        props["ref:cui_dgcye"] = registro["ref_cui_dgcye"]
    if registro.get("operator_type"):
        props["operator:type"] = registro["operator_type"]
    if registro.get("isced_level"):
        props["isced:level"] = registro["isced_level"]

    return {
        "type": "Feature",
        "geometry": {"type": "Point", "coordinates": [registro["lon"], registro["lat"]]},
        "properties": props,
    }


def _distancia_m(lat1, lon1, lat2, lon2) -> float:
    R = 6371000
    p1, p2 = math.radians(lat1), math.radians(lat2)
    dp = math.radians(lat2 - lat1)
    dl = math.radians(lon2 - lon1)
    a = math.sin(dp / 2) ** 2 + math.cos(p1) * math.cos(p2) * math.sin(dl / 2) ** 2
    return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))


def filtrar_dia_evento(registros: list[dict], sede: dict) -> list[dict]:
    return [
        r
        for r in registros
        if r["lat"] is not None
        and _distancia_m(r["lat"], r["lon"], sede["lat"], sede["lon"])
        <= sede["radio_prioridad_m"]
    ]


def escribir_salidas(registros: list[dict], carpeta: str, sede_evento: dict | None) -> dict:
    if sede_evento:
        for r in filtrar_dia_evento([r for r in registros if r["lat"]], sede_evento):
            r["prioridad_dia_evento"] = True

    baldes = {"confirmada": [], "faltante": [], "a_revisar": [], "verificar_terreno": []}
    for r in registros:
        baldes.setdefault(r["clasificacion"], []).append(r)

    nombre_archivo = {
        "confirmada": "escuelas_confirmadas.csv",
        "faltante": "escuelas_faltantes.csv",
        "a_revisar": "escuelas_a_revisar.csv",
        "verificar_terreno": "escuelas_verificar_terreno.csv",
    }
    for clave, archivo in nombre_archivo.items():
        with open(os.path.join(carpeta, archivo), "w", newline="", encoding="utf-8") as f:
            w = csv.DictWriter(f, fieldnames=CAMPOS_CSV)
            w.writeheader()
            for r in baldes[clave]:
                w.writerow({k: r.get(k) for k in CAMPOS_CSV})

    faltantes_geojson = {
        "type": "FeatureCollection",
        "features": [construir_feature_geojson(r) for r in baldes["faltante"]],
    }
    with open(os.path.join(carpeta, "escuelas_todas.geojson"), "w", encoding="utf-8") as f:
        json.dump(faltantes_geojson, f, ensure_ascii=False, indent=2)

    dia_evento = [r for r in baldes["faltante"] if r.get("prioridad_dia_evento")]
    dia_evento_geojson = {
        "type": "FeatureCollection",
        "features": [construir_feature_geojson(r) for r in dia_evento],
    }
    with open(os.path.join(carpeta, "escuelas_dia_evento.geojson"), "w", encoding="utf-8") as f:
        json.dump(dia_evento_geojson, f, ensure_ascii=False, indent=2)

    return {clave: len(lista) for clave, lista in baldes.items()}


def agregar_entrada_changelog(carpeta: str, resumen: dict) -> None:
    fecha = datetime.date.today().isoformat()
    linea = (
        f"\n## {fecha}\n"
        f"- Confirmadas: {resumen['confirmada']}, Faltantes: {resumen['faltante']}, "
        f"A revisar: {resumen['a_revisar']}, Verificar en terreno: {resumen['verificar_terreno']}\n"
    )
    path = os.path.join(carpeta, "CHANGELOG.md")
    modo = "a" if os.path.exists(path) else "w"
    with open(path, modo, encoding="utf-8") as f:
        if modo == "w":
            f.write(f"# Changelog\n{linea}")
        else:
            f.write(linea)
```

- [ ] **Step 4: Correr y verificar que pasan**

Run: `python3 -m pytest tests/test_salidas.py -v`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add salidas.py tests/test_salidas.py
git commit -m "feat: generacion de CSV/GeoJSON de salida + entrada de CHANGELOG"
```

---

### Task 8: CLI `reconciliar.py` (orquestación + `--dry-run`)

**Files:**
- Create: `reconciliar.py`
- Create: `tests/test_reconciliar.py`

**Interfaces:**
- Consumes: todo lo de Tasks 1-7
- Produces: `ejecutar(localidad_dir: str, dry_run: bool = False) -> dict` — función principal, separada de `main()` para poder testearla sin CLI real

- [ ] **Step 1: Escribir un test de integración con las piezas ya testeadas, pero con las funciones de I/O inyectadas**

```python
# tests/test_reconciliar.py
from reconciliar import ejecutar


def test_ejecutar_dry_run_no_escribe_archivos(tmp_path, monkeypatch):
    import shutil

    localidad_dir = tmp_path / "pergamino"
    localidad_dir.mkdir()
    (localidad_dir / "locality.yml").write_text(
        "nombre: Pergamino\nprovincia: Buenos Aires\nwfs_localidad_filter: PERGAMINO\n",
        encoding="utf-8",
    )
    (localidad_dir / "overrides.csv").write_text("ref:cue,decision,nota\n", encoding="utf-8")

    def fake_fetch_padron_localidad(termino):
        return [
            {
                "cue": "1", "nombre": "ESCUELA X", "operator_type": "public",
                "ambito": "Urbano", "domicilio": "X", "lat": None, "lon": None,
                "ref_cui_dgcye": None, "isced_level": None, "clasificacion": None,
                "nota": None, "prioridad_dia_evento": False,
            }
        ]

    def fake_fetch_geometria_localidad(termino):
        return {"1": (-33.9, -60.5)}

    def fake_verificar_localidad(registros, checkpoint_path, **kwargs):
        for r in registros:
            r["clasificacion"] = "faltante"
        return registros

    monkeypatch.setattr("reconciliar.fetch_padron_localidad", fake_fetch_padron_localidad)
    monkeypatch.setattr("reconciliar.fetch_geometria_localidad", fake_fetch_geometria_localidad)
    monkeypatch.setattr("reconciliar.verificar_localidad", fake_verificar_localidad)

    resumen = ejecutar(str(localidad_dir), dry_run=True)

    assert resumen["faltante"] == 1
    assert not (localidad_dir / "escuelas_faltantes.csv").exists()


def test_ejecutar_sin_dry_run_escribe_archivos(tmp_path, monkeypatch):
    localidad_dir = tmp_path / "pergamino"
    localidad_dir.mkdir()
    (localidad_dir / "locality.yml").write_text(
        "nombre: Pergamino\nprovincia: Buenos Aires\nwfs_localidad_filter: PERGAMINO\n",
        encoding="utf-8",
    )
    (localidad_dir / "overrides.csv").write_text("ref:cue,decision,nota\n", encoding="utf-8")

    def fake_fetch_padron_localidad(termino):
        return [
            {
                "cue": "1", "nombre": "ESCUELA X", "operator_type": "public",
                "ambito": "Urbano", "domicilio": "X", "lat": None, "lon": None,
                "ref_cui_dgcye": None, "isced_level": None, "clasificacion": None,
                "nota": None, "prioridad_dia_evento": False,
            }
        ]

    monkeypatch.setattr("reconciliar.fetch_padron_localidad", fake_fetch_padron_localidad)
    monkeypatch.setattr("reconciliar.fetch_geometria_localidad", lambda t: {"1": (-33.9, -60.5)})
    monkeypatch.setattr(
        "reconciliar.verificar_localidad",
        lambda registros, checkpoint_path, **kw: [dict(r, clasificacion="faltante") for r in registros],
    )

    ejecutar(str(localidad_dir), dry_run=False)

    assert (localidad_dir / "escuelas_faltantes.csv").exists()
    assert (localidad_dir / "CHANGELOG.md").exists()
```

- [ ] **Step 2: Correr y verificar que fallan**

Run: `python3 -m pytest tests/test_reconciliar.py -v`
Expected: FAIL con `ModuleNotFoundError: No module named 'reconciliar'`

- [ ] **Step 3: Implementar `reconciliar.py`**

```python
# reconciliar.py
import argparse
import os

from adapters.dgcye_buenos_aires import enriquecer as enriquecer_dgcye
from fuente_oficial import fetch_geometria_localidad, fetch_padron_localidad
from locality import cargar_locality
from overpass_check import verificar_localidad
from overrides import aplicar_overrides, cargar_overrides
from salidas import agregar_entrada_changelog, escribir_salidas

ADAPTADORES = {"dgcye_buenos_aires": enriquecer_dgcye}


def ejecutar(localidad_dir: str, dry_run: bool = False) -> dict:
    config = cargar_locality(os.path.join(localidad_dir, "locality.yml"))

    registros = fetch_padron_localidad(config["wfs_localidad_filter"])
    geometrias = fetch_geometria_localidad(config["wfs_localidad_filter"])

    con_geo = []
    sin_geo = []
    for reg in registros:
        coords = geometrias.get(reg["cue"])
        if coords:
            reg["lat"], reg["lon"] = coords
            con_geo.append(reg)
        else:
            reg["clasificacion"] = "verificar_terreno"
            reg["nota"] = "sin coordenadas en WFS, ubicar manualmente"
            sin_geo.append(reg)

    if config["enriquecimiento"]:
        adaptador = ADAPTADORES[config["enriquecimiento"]]
        snapshot_path = os.path.join(localidad_dir, "predios_dgcye.json")
        con_geo = adaptador(con_geo, snapshot_path)

    checkpoint_path = os.path.join(localidad_dir, f".reconciliar_estado_{config['nombre']}.json")
    con_geo = verificar_localidad(con_geo, checkpoint_path)

    todos = con_geo + sin_geo
    overrides = cargar_overrides(os.path.join(localidad_dir, "overrides.csv"))
    todos = aplicar_overrides(todos, overrides)

    if dry_run:
        resumen = {}
        for r in todos:
            resumen[r["clasificacion"]] = resumen.get(r["clasificacion"], 0) + 1
        return resumen

    resumen = escribir_salidas(todos, localidad_dir, config["sede_evento"])
    agregar_entrada_changelog(localidad_dir, resumen)
    return resumen


def main():
    parser = argparse.ArgumentParser(description="Reconciliar escuelas oficiales vs OSM")
    parser.add_argument("--localidad", required=True, help="carpeta de la localidad, ej. pergamino")
    parser.add_argument("--dry-run", action="store_true")
    args = parser.parse_args()

    resumen = ejecutar(args.localidad, dry_run=args.dry_run)
    print("Resumen:", resumen)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Correr y verificar que pasan**

Run: `python3 -m pytest tests/test_reconciliar.py -v`
Expected: PASS (2 tests)

- [ ] **Step 5: Correr toda la suite junta**

Run: `python3 -m pytest -v`
Expected: PASS (todos los tests de Tasks 1-8)

- [ ] **Step 6: Commit**

```bash
git add reconciliar.py tests/test_reconciliar.py
git commit -m "feat: CLI reconciliar.py que orquesta todo el pipeline, con --dry-run"
```

---

### Task 9: Validación de regresión (Pergamino) y primera corrida en vivo (Chajarí)

**Files:**
- Modify: `pergamino/CHANGELOG.md` (vía el script)
- Modify: `chajari/CHANGELOG.md` (nuevo, vía el script)

**Interfaces:**
- Consumes: todo el pipeline completo (Tasks 1-8)
- Produces: nada nuevo — esta tarea es de validación, no de código

- [ ] **Step 1: Correr dry-run para Pergamino y comparar contra los datos actuales**

```bash
python3 reconciliar.py --localidad pergamino --dry-run
```

Expected: un resumen con números en el mismo orden de magnitud que los actuales (29 confirmadas, ~150 faltantes antes de overrides, 7 a_revisar, 10 verificar_terreno — puede variar un poco porque Overpass es data viva). Si hay una diferencia grande e inesperada (ej. 0 confirmadas), **no seguir** — investigar antes de correr en real.

- [ ] **Step 2: Correr Pergamino en real y revisar el diff con git**

```bash
python3 reconciliar.py --localidad pergamino
git diff --stat pergamino/
```

Expected: cambios acotados en los CSV/GeoJSON (no un archivo vacío ni una caída masiva de escuelas). Revisar a mano `git diff pergamino/escuelas_confirmadas.csv` antes de commitear.

- [ ] **Step 3: Commit de la corrida de Pergamino**

```bash
git add pergamino/
git commit -m "chore: recorrer reconciliacion de Pergamino con el pipeline nuevo"
```

- [ ] **Step 4: Primera corrida en vivo para Chajarí**

```bash
python3 reconciliar.py --localidad chajari
```

Expected: genera `chajari/escuelas_confirmadas.csv`, `_faltantes.csv`, `_a_revisar.csv`, `_verificar_terreno.csv`, `escuelas_todas.geojson`, `CHANGELOG.md`. Dado que Chajarí ya se mapeó bastante en el Encuentro 2025, se espera una proporción alta de `confirmada` (a diferencia de Pergamino) — si sale mayormente `faltante`, revisar antes de confiar en el resultado.

- [ ] **Step 5: Commit de la primera corrida de Chajarí**

```bash
git add chajari/
git commit -m "feat: primera corrida en vivo del pipeline para Chajari"
git push origin master
```

---

## Self-Review (completado al escribir este plan)

**Cobertura del spec:**
- Stage 1a/1b (padrón + WFS) → Tasks 2, 3
- Adaptador DGCyE → Task 4
- Overpass + clasificación → Task 5
- Merge con overrides + migración inicial → Task 6
- Generación de salidas + CHANGELOG → Task 7
- CLI + `--dry-run` → Task 8
- Validación de regresión (Pergamino) + corrida en vivo (Chajarí) → Task 9
- `chajari-control/` fuera de alcance → no se toca en ningún task ✓

**Consistencia de tipos:** el diccionario "registro canónico" (Global Constraints) se usa igual en Tasks 2-8; verificado que las claves (`cue`, `nombre`, `operator_type`, `lat`, `lon`, `ref_cui_dgcye`, `isced_level`, `clasificacion`, `nota`, `prioridad_dia_evento`) son las mismas en todos los tests y en las implementaciones.
