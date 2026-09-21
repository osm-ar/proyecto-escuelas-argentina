# Plantilla de instrucciones — Challenge MapRoulette "Escuelas Pergamino 2026"

Pegar esto tal cual en el campo **Instruction** del challenge (nivel challenge, no por tarea —
así todas las tareas quedan con el mismo formato, a diferencia de Luján donde solo 2 de 221
tareas usaron la versión "paso a paso" y el resto quedó con una plantilla más pobre).

MapRoulette reemplaza `{{propiedad}}` por el valor de esa propiedad en el GeoJSON de cada
tarea automáticamente — no hay que tocar nada tarea por tarea.

---

```md
## 🏫 {{name}}

### 🔍 Paso 1 — Verificar que no exista ya en OSM (OBLIGATORIO)

👉 [Abrir verificación automática en Overpass Turbo]({{_overpass_verificacion}})
(busca escuelas/jardines/institutos en un radio de 400m de este punto)

- Si aparece una escuela que es claramente esta misma (nombre igual o muy similar):
  **no la agregues**. Marcá la tarea como "Ya existe" (`Already Fixed` / already_fixed) y,
  si podés, agregale el tag `ref:cue={{ref:cue}}` al elemento que ya está en OSM.
- Si no aparece nada, o lo que aparece es claramente otra institución: continuá al paso 2.

⚠️ **Este predio puede compartir edificio con otra escuela.** Fijate si el nombre de abajo
coincide con algo que ya viste en el paso anterior antes de asumir que falta.

### 📍 Paso 2 — Posicionar el punto

La coordenada de esta tarea viene del predio oficial (DGCyE), puede no ser exactamente la
puerta de entrada. Ajustá la posición del punto sobre el edificio real usando la imagen
satelital antes de guardar.

### 🏷️ Paso 3 — Tags para copiar en JOSM / iD

```
amenity={{amenity}}
name={{name}}
ref:cue={{ref:cue}}
ref:cui_dgcye={{ref:cui_dgcye}}
operator:type={{operator:type}}
isced:level={{isced:level}}
source={{source}}
```

📎 Fuente oficial del dato (Ministerio de Educación): {{official:legajo}}

*(Si `isced:level` no aparece en la lista de arriba es porque no lo pudimos inferir con
confianza del nombre oficial — dejalo sin ese tag, no inventes un valor.)*

### ✅ Paso 4 — Completar

- Guardá el cambio en OpenStreetMap con un changeset comment que incluya
  `#PergaminoOSM2026` y `#ProyectoEscuelasOSM`.
- Marcá la tarea como completada en MapRoulette.

---
🎪 Encuentro OSM Argentina 2026 · Pergamino · Proyecto Escuelas OSM
```

---

## Notas para quien arme el challenge (no van en el texto de arriba)

- **Archivo a cargar**: `escuelas_todas.geojson` (138 tareas, en este mismo directorio) — se decidió cargar
  todo el pool de una vez, no solo las 40 del día, porque se completa en los días siguientes
  al encuentro, no solo durante el mapatón presencial.
- **Priority Rules de MapRoulette**: configurar una regla de prioridad **Alta** usando la
  propiedad `_prioridad_dia_evento = true` (40 tareas, urbanas y cercanas a la sede — para
  que aparezcan primero durante el mapatón presencial) y dejar el resto en prioridad
  **Media/Baja** por defecto, para que se sigan completando después sin necesidad de un
  segundo challenge separado.
- **Verificación del link de Overpass**: ✅ probado y funcionando (2026-09-20) — carga la
  consulta, centra el mapa en el punto correcto y corre automáticamente. El servidor público
  de Overpass a veces da timeout por sobrecarga (pasó en el primer intento de prueba); si a
  algún mapeador le pasa el día del evento, que reintente en unos segundos antes de asumir
  que la tarea está mal armada.
- **Changeset comment / tag de seguimiento**: se propone `#PergaminoOSM2026` en el
  changeset en vez de un tag propio en el elemento (como era `sotm_lujan2025=yes`).
  Un hashtag de changeset se puede buscar después en el historial sin ensuciar el dato del
  mapa con un tag que solo tiene sentido para un evento puntual y un año. Si de todas formas
  se quiere un tag de seguimiento en el elemento, usar algo reutilizable entre años, ej.
  `survey:project=escuelas_pergamino` en vez de `sotm_pergamino2026=yes`.
- **QA antes de publicar** (ver hallazgos del challenge de Luján):
  - [ ] Cantidad de tareas cargadas == cantidad de features del GeoJSON (40).
  - [ ] Ningún nombre de tarea contiene "test", "prueba" o similar.
  - [ ] Sin coordenadas exactamente duplicadas entre tareas.
  - [ ] El ID de cada tarea usa el CUE (`ref:cue`), no un contador manual tipo "Pergamino 001"
    (eso fue lo que causó las tareas duplicadas de Luján, por un typo de espacio en el prefijo).
- **cooperativeWork**: evaluar armar el challenge como *Cooperative Challenge* en vez de
  "copiar tags a mano", para que el mapeador aplique el cambio sugerido directo desde
  MapRoulette (reduce el error de tipeo que dejó `isced:level=No especificado` pegado en
  Luján). Si no se llega a configurar a tiempo para el evento, el formato de arriba
  (copiar/pegar) funciona igual como respaldo.
- **Revisión**: activar el review de MapRoulette al menos para una muestra de las tareas
  completadas el día del evento (en Luján no se revisó ninguna).
