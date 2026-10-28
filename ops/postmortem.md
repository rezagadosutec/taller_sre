# Postmortem: clientes de Chile segmentados como Perú

> Sin culpables: buscamos qué falló en el proceso, no quién se equivocó.

**Equipo y roles:** Comandante: [Cesar Chijar ] · Operador: [ Efren Carrillo ] · Comunicador: [ Diego Flores ]

## Resumen

El commit `3915bc8` asignó `CL` a Perú/Andina. Silver conservó ese cruce y Gold agrupó a esos clientes junto con Perú, mientras el total global de contactables permaneció igual. El catálogo y el lote del lunes ya están preparados localmente; la restauración de Silver y el primer `Game day – Verificar` en verde siguen pendientes en Databricks.

## Impacto

- ¿Qué clientes, tablas o procesos se vieron afectados? ¿Cuántos?
- Se afectaron `gameday.ref_paises`, `gameday.silver_customers_v2` y `gameday.gold_contactables_por_pais`. Según los CSV del snapshot del viernes hay 20 IDs únicos: 8 clientes con código `CL`, de los cuales 6 son contactables; Perú tiene 5 contactables. Con el catálogo malo, Gold habría mostrado esos 11 contactables bajo Perú y ninguno bajo Chile. Confirmar las filas publicadas y destinatarios reales con las tablas/eventos antes del cierre.
- Después de cargar `customers_lunes.csv`, el verificador espera 26 IDs únicos en Silver.

## Línea de tiempo

| Hora | Qué pasó |
| --- | --- |
| 2026-10-23 17:48 -0500 | Se crea `3915bc8`, que cambia el mapeo de `CL` a Perú/Andina. |
| lunes 9:05 | Marketing reporta la segmentación incorrecta; consultar `ops_event_log` para la hora exacta de inicio. |
| [ hora de `ops_event_log` ] | Se confirma el código `CL` mal asociado en catálogo/Silver. |
| [ hora ] | Marketing pausa las campañas; se retiene la publicación hasta restaurar Silver. |
| [ hora del primer verificar verde ] | Pendiente: registrar el resultado y calcular el MTTR desde `incidente_inicio`. |

**MTTR:** Pendiente del primer `Game day – Verificar` en verde.

## Causa raíz

- ¿Qué cambio lo causó? ¿Por qué los checks no lo frenaron?
- Causa raíz: el commit `3915bc8` modificó el catálogo y dejó `CL,CHL,Perú,Andina`. `load_ref.py` reemplaza la tabla de referencia en cada corrida; `load_silver.sql` copia `country_name` y `region` desde ese catálogo, y Gold agrupa por ambos campos. Los checks existentes contaban errores de nombre y edad, pero no contrastaban el país con su código ISO; el total global no detectó el cambio de segmento.

## Qué funcionó y qué no

- [ ... ]
- La métrica global se mantuvo estable y ocultó el cruce entre países. El historial de Git y el historial Delta permiten separar el arreglo del código de la restauración de datos.

## Acción preventiva

- Check nuevo en `framework/validation/validate_silver.py`: compara los mapeos esperados con el catálogo y con cada fila Silver; registra las discrepancias y falla antes de Gold.
- Validación local: sintaxis Pylance y prueba local de dataset correctas. Falta ejecutar el pipeline con el catálogo malo para confirmar que el job falla en `validation`, y luego verificar el reproceso real.
- Otras acciones: restaurar Silver a la versión Delta buena identificada en `DESCRIBE HISTORY`, reprocesar los 26 IDs y registrar aquí el resultado de `Game day – Verificar`.
