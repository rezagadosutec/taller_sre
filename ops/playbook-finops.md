# Playbook FinOps: pipeline de contactables por país

> No buscamos la cifra exacta: buscamos que sepan de dónde sale el costo,
> quién es el dueño y qué decisión tomarían.
>
## 0. Supuestos de tarifa (iguales para todos)

Precios de lista públicos, revisados en octubre de 2026. Úsenlos tal cual para
que todos los equipos lleguen a números comparables.

| Recurso | Tarifa supuesta | De dónde sale |
| --- | --- | --- |
| GitHub Actions, runner Linux 2-core | USD 0.006 por minuto | Precio de lista de GitHub. **Cada job se redondea al minuto entero.** |
| Databricks SQL Warehouse serverless, 2X-Small | 4 DBU/hora × USD 0.70/DBU = **USD 2.80 por hora** | Tabla de DBU por tamaño de Databricks y tarifa Premium en AWS US East |
| Auto-stop del warehouse | 10 minutos encendido sin uso después de la última query | Valor por defecto; revisen el de su warehouse |
| Hora-persona | USD 25 por hora | Supuesto del taller |

> Ojo: en un repo **público** los minutos de Actions no se cobran, y en
> Databricks Free Edition tampoco pagan el warehouse. Calculamos con precio de
> lista porque en una empresa sí se paga.

Pistas para calcular:

- Minutos de Actions facturados = suma, por job, de su duración redondeada hacia arriba al minuto.
- Minutos de warehouse = desde la primera query hasta la última (`ops_event_log`) **más** el auto-stop.
- Comparen las dos partidas: ¿cuál pesa más y por qué?

### Midan, no supongan: lo que registra Databricks

Las tarifas de arriba son un supuesto. El consumo real lo registra el propio
Databricks, también en Free Edition. Úsenlo para contrastar su estimación.

```sql
-- ¿En qué se fue el consumo? (DBU por producto, últimos 7 días)
SELECT billing_origin_product, sku_name, ROUND(SUM(usage_quantity), 2) AS dbu
FROM system.billing.usage
WHERE usage_date >= current_date() - 7
GROUP BY ALL ORDER BY dbu DESC;

-- ¿Quién lanzó las queries? GitHub Actions aparece como el conector de Python
SELECT client_application, executed_by, COUNT(*) AS sentencias,
       ROUND(SUM(total_duration_ms) / 60000, 1) AS minutos_de_query
FROM system.query.history
WHERE start_time >= current_date() - 7
GROUP BY ALL ORDER BY minutos_de_query DESC;
```

Preguntas:

- Comparen los **minutos de query** con lo que se cobró. ¿Qué parte del costo es trabajo útil y qué parte es el warehouse encendido sin hacer nada?
- ¿Aparece algo que nadie pidió? Por ejemplo `PREDICTIVE_OPTIMIZATION`, el mantenimiento automático de las tablas.
- `executed_by`: ¿con qué usuario corre el pipeline? Si es el de una persona, ¿cómo se separa el costo de CI del trabajo manual?

## 1. ¿Cuánto cuesta una corrida?

El workflow tiene **7 jobs facturables de Actions**: `deploy`, `bronze`, `ref`, `silver`, `validation`, `gold` y `tests`. `bronze` y `ref` corren en paralelo; `gold` y `tests` también. Anoten la duración de cada job en una corrida real de Actions. El log de eventos sirve para ubicar etapas, pero sus marcas son de fin de job, no duración exacta de consultas.

| Stage/job | Minutos de Actions (medir) | Intervalo entre eventos (min, orientativo) | Costo de Actions |
| --- | --- | --- | --- |
| deploy | medir en Actions | medir con query 9 | `ceil(min) × USD 0.006` |
| bronze | medir en Actions | medir con query 9 | `ceil(min) × USD 0.006` |
| ref | medir en Actions | medir con query 9 | `ceil(min) × USD 0.006` |
| silver | medir en Actions | medir con query 9 | `ceil(min) × USD 0.006` |
| validation | medir en Actions | medir con query 9 | `ceil(min) × USD 0.006` |
| gold | medir en Actions | medir con query 9 | `ceil(min) × USD 0.006` |
| tests (no usa Databricks) | medir en Actions | — | `ceil(min) × USD 0.006` |
| **Total por corrida** | sumar los 7 jobs | query 10, más auto-stop | sumar Actions + warehouse |

**Cálculo:** costo de Actions = `USD 0.006 × suma(ceil(duración de cada job en minutos))`. Como piso, si cada job dura menos de un minuto, son 7 minutos facturados = **USD 0.042 por corrida**; la duración real se consulta en Actions. El costo estimado del warehouse es `USD 2.80 × minutos encendido / 60`, incluyendo los 10 minutos de auto-stop. `ops_event_log` es una aproximación: para el consumo cobrado usen `system.billing.usage` y multipliquen los DBU por USD 0.70. No sumen otra vez esos DBU al cálculo estimado.

El workflow corre por `push` a `main` si cambia `data/**`, `framework/**` o el propio workflow, y también por ejecución manual. No tiene una frecuencia mensual fija: cuenten las corridas completadas de los últimos 30 días en Actions y multipliquen por el costo medio por corrida. **Costo mensual estimado = corridas del mes × costo medio por corrida.** El repo no contiene el historial de ejecuciones ni las marcas de Databricks; por eso no sería fiable inventar un total mensual.

## 2. ¿Cuánto costó el incidente?

Contabilicen las ejecuciones extraordinarias desde que empezó el incidente hasta el primer `Game day – Verificar` verde. Incluyan preparación, intentos fallidos, reprocesos y verificaciones manuales que hayan usado el warehouse; no cuenten la corrida normal como costo incremental.

- Corridas extra: contar en Actions. **Costo de cómputo incremental = corridas extra × costo por corrida medido en la sección 1.**
- Trabajo humano: sumar las horas efectivas de cada participante. **Costo de horas-persona = horas totales × USD 25.** Si todos trabajaron el mismo tiempo, es `personas × horas por persona × USD 25`.
- **Costo total del incidente = cómputo incremental + horas-persona.** Anoten por separado el MTTR transcurrido y las horas-persona: no son lo mismo.
- Comparación de recuperación: una recarga completa vuelve a leer todos los CSV de `data/source/`, recrea el catálogo y recalcula Gold; Silver usa `MERGE` por `customer_id`. Comparen el costo de esa corrida completa con el reproceso selectivo que realmente hicieron usando las ejecuciones y DBU observados. Sin esos registros no se puede afirmar un ahorro numérico.

## 3. ¿Quién es el dueño?

El dueño no se escribe en un Excel: se deja **en el recurso**, con tags, igual
que en Azure o AWS. Así un reporte de costos lo puede leer solo.

- `gold_contactables_por_pais` lo consume Marketing. El dueño propuesto del costo y del pipeline es el **equipo de datos (`data-platform`)**; Marketing es el consumidor. Confirmen el equipo responsable en el PR antes de aplicar los tags.
- Tagueen la tabla desde el código (por ejemplo en `deploy_silver_v2.sql`, para que se aplique en cada deploy):

  ```sql
  ALTER TABLE gameday.gold_contactables_por_pais
    SET TAGS ('owner' = 'data-platform', 'cost_center' = 'taller-sre', 'consumer' = 'marketing');
  ```

- Compruébenlo con una query, no a ojo:

  ```sql
  SELECT tag_name, tag_value
  FROM information_schema.table_tags
  WHERE schema_name = 'gameday' AND table_name = 'gold_contactables_por_pais';
  ```

- Política como código: el workflow **FinOps – Política de tags** corre después de
  cada pipeline y **falla si una tabla Gold no tiene tag `owner`**. Si lo corren hoy, sale en rojo.
  Añadan el tag, compruébenlo con la query y ejecuten **FinOps – Política de tags** desde Actions; el resultado esperado es verde.

- Los tags de una tabla sirven para gobierno, pero **no** aparecen en la factura.
  Para atribuir costo, el tag va en lo que consume: el SQL Warehouse (sección
  *Tags* al editarlo): `owner=data-platform`, `cost_center=taller-sre` y
  `environment=workshop`. Ese tag llega a `system.billing.usage.custom_tags`.
  Si dos equipos comparten warehouse, el consumo se atribuye al recurso compartido,
  no a cada equipo; separen warehouses o usen un mecanismo de imputación por equipo.

## 4. ¿Qué deciden?

> Hallazgo: `data/**` dispara el pipeline, pero Bronze solo lee `data/source/`.
> Por tanto, añadir o editar un archivo en `data/incoming/` dispara una corrida
> que no ingiere ese lote. Además, `framework/**` incluye documentación y tests.

| Costo oculto que encontramos | Decisión (apagar / ajustar / priorizar) | Dueño | Fecha |
| --- | --- | --- | --- |
| Push a `data/incoming/` (y cambios no ejecutables en `framework/`) inicia jobs aunque no cambie lo que procesa el pipeline. | Ajustar los filtros `paths` para incluir `data/source/**`, `data/ref/**`, `framework/{deployment,ingestion,processing,validation,monitoring,tests}/**` y el workflow; mantener `workflow_dispatch` para reprocesos intencionales. Revisar la frecuencia de corridas después del cambio. | Equipo de datos (`data-platform`) | 2026-10-28 |

Bonus (opcional): implementen la decisión en el workflow y comparen el número de corridas y el costo mensual estimado antes y después.
