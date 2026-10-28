# Runbook: país incorrecto en campañas de contactables

> Un runbook es lo que haría cualquier persona del equipo si esto vuelve a
> pasar a las 3 a. m. Escríbanlo para alguien que no estuvo hoy.

## Cómo me doy cuenta

- La alerta de Marketing reporta contactos de Chile dentro de la campaña de Perú y una campaña de Chile con menos destinatarios.
- Comparar el catálogo y Silver por código; el nombre y la región deben corresponder al código:

	```sql
	SELECT country_code, country_name, region, COUNT(*) AS clientes
	FROM gameday.silver_customers_v2
	GROUP BY country_code, country_name, region
	ORDER BY country_code;

	SELECT * FROM gameday.ref_paises ORDER BY country_code;
	SELECT * FROM gameday.gold_contactables_por_pais ORDER BY pais;
	```

- Revisar `DESCRIBE HISTORY gameday.silver_customers_v2` y `gameday.ops_event_log` para ubicar la primera carga afectada y el commit registrado.

## Contener (que no empeore)

- Pedir a Marketing que pause los envíos basados en Gold.
- No hacer merge a `main` ni lanzar el pipeline manualmente mientras Silver esté mal: cada publicación de Gold podría repetir la segmentación incorrecta.
- Mantener el lote pendiente en el cambio de recuperación; cargarlo junto con el catálogo corregido.

## Recuperar

1. Código: abrir un PR que revierta el commit `3915bc8` (`git revert 3915bc8`). Confirmar que `CL` sea `Chile / Cono Sur` y `PE` sea `Perú / Andina`.
2. Datos: usar `DESCRIBE HISTORY` para identificar la última versión buena de Silver, anterior a la carga con el catálogo incorrecto; restaurarla antes de ejecutar el pipeline:

	```sql
	RESTORE TABLE gameday.silver_customers_v2 TO VERSION AS OF <version_buena>;
	```

	No asumir un número de versión: elegirlo del historial del workspace.
3. Reproceso: después de restaurar Silver y corregir el catálogo, ejecutar una vez el pipeline desde el commit corregido. Bronze vuelve a leer los CSV; el `MERGE` de Silver usa `customer_id` y Gold se reemplaza con el resultado recalculado.
4. Pendientes: incluir `customers_lunes.csv` en `data/source/` para que ese mismo reproceso incorpore los seis clientes del lunes.

## Cómo confirmo que volvimos a un estado bueno

- Ejecutar **Game day – Verificar** y exigir resultado verde: 26 clientes únicos en Silver, catálogo correcto, Gold igual a los conteos recalculados desde Silver y cero duplicados.
- Mantener Marketing en pausa hasta tener ese resultado; luego comunicar la recuperación y autorizar los envíos.

## A quién aviso

| Momento | A quién | Qué le digo |
| --- | --- | --- |
| Al detectar | Marketing | Pausar campañas por país mientras confirmamos la segmentación. |
| Al recuperar | Marketing y equipo de datos | Catálogo, Silver y Gold verificados; se puede reanudar el envío. |
