# Postmortem: <nombre del incidente>

> Sin culpables: buscamos qué falló en el proceso, no quién se equivocó.

**Equipo y roles:** Comandante: [ Cesar Hijar ] · Operador: [ Efren Carrillo ] · Comunicador: [ Diego Flores ] · Comunicador: [ Celso Ibarcerna ] · Especialista: [ Alex Galindo ] 

## Resumen

[ En 2 o 3 frases: qué pasó, a quién afectó y cómo se resolvió. ]

## Impacto

- ¿Qué clientes, tablas o procesos se vieron afectados? ¿Cuántos?
- [ ... ]

## Línea de tiempo

| Hora | Qué pasó |
| --- | --- |
| [ viernes ... ] | [ entra el cambio ... ] |
| [ ... ] | [ inicio del incidente (preparar-produccion) ] |
| [ ... ] | [ detectamos la causa ] |
| [ ... ] | [ contenemos ] |
| [ ... ] | [ primer verificar en verde ] |

**MTTR:** [ minutos desde el inicio del incidente hasta el primer verificar en verde ]

## Causa raíz

- ¿Qué cambio lo causó? ¿Por qué los checks no lo frenaron?
- [ ... ]

## Qué funcionó y qué no

- [ ... ]

## Acción preventiva

- Check nuevo en `framework/validation/validate_silver.py`: [ ¿qué valida? ]
- ¿Lo probaron? ¿Falla si vuelven a aplicar el cambio del viernes? [ sí / no ]
- Otras acciones (con dueño y fecha): [ ... ]
