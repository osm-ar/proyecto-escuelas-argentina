# Chajarí — Control QA

No es un dataset de mapeo nuevo. Reempaqueta las 30 escuelas de Chajarí
(Entre Ríos) ya trabajadas en el Encuentro OSM Argentina 2025, con el mismo
schema/pipeline usado para Pergamino 2026, para validar antes del evento que:

- el GeoJSON carga con la cantidad correcta de tareas en MapRoulette
- la plantilla de instrucciones (`{{propiedad}}`) renderiza bien
- el link de verificación automática en Overpass funciona
- el sistema de duplicados detecta correctamente escuelas que ya existen

Generado desde `escuelas_chajari_sotm2025_20250903_1536.geojson` (workspace
2025). Nota: 9 de las 30 escuelas originales no tenían CUE asignado y 1 CUE
estaba duplicado — el id de tarea usa un fallback por índice para esos casos
(ver `build_chajari_control.py` en el historial de este commit).
