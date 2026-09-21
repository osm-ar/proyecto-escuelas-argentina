# Changelog — Pergamino

## 2026-09-20
- Reconciliación afinada vía análisis de consistencia entre fuentes (polígonos), recupera 3 escuelas adicionales.
- Pool final del challenge: 138 tareas (`escuelas_todas.geojson`), de las cuales 40 quedan marcadas con `_prioridad_dia_evento = true` para el mapatón presencial (`escuelas_dia_evento.geojson`).
- Redactada la plantilla de instrucciones para MapRoulette (`escuelas_instrucciones_challenge.md`), a nivel challenge (no por tarea), con lecciones aprendidas del challenge de Luján 2025.
- Verificado el link de verificación automática en Overpass Turbo (carga la consulta, centra el mapa y corre solo).

## 2026-09-19
- Descubierta la API WFS de DGCyE para predios educativos de Pergamino (128 escuelas).
- Cruce inicial contra OSM: 35 confirmadas, 16 a revisar, 135 faltantes.
- Fuente comparada contra el padrón municipal de educación (187 registros vs 32 en OSM en el primer paso).
