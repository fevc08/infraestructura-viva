# ADR-0008: Estrategia de monitoreo con CloudWatch y correlación vía EventBridge

## Estado
Aceptado

## Fecha
2026-07-26

## Contexto
Se requiere detectar incidentes de forma proactiva en los componentes de
cómputo, base de datos y aplicaciones serverless, notificar automáticamente
a los equipos correspondientes, y ejecutar remediaciones automáticas cuando
sea posible, evitando además "alarm fatigue" ante eventos aislados no
críticos.

## Decisión
Se utilizará **Amazon CloudWatch** para métricas, logs y alarmas, integrado
con **SNS** para notificaciones, con **Auto Scaling** como acción de
remediación automática ante CPU alta, y **EventBridge** para correlacionar
múltiples alarmas y elevar la severidad de un incidente cuando corresponda.

## Alternativas consideradas
| Opción | Pros | Contras | ¿Por qué no se eligió? |
|---|---|---|---|
| Datadog / New Relic | Dashboards más ricos, mejor trazabilidad de aplicaciones | Costos de licenciamiento, curva de aprendizaje adicional | Fuera del alcance de un proyecto en Free Tier/AWS Academy |
| Alarmas aisladas sin correlación (solo CloudWatch Alarms) | Configuración más simple | No distingue un incidente puntual de uno sistémico; puede generar ruido de notificaciones | No cumple con el requerimiento explícito de "correlacionar eventos" |
| Remediación 100% manual (solo notificar, sin Auto Scaling) | Menor riesgo de acciones automáticas no deseadas | Tiempo de respuesta más lento ante picos de carga | Contradice el objetivo de "detectar y corregir incidentes de forma proactiva" |

## Consecuencias
- **Positivas:** menor tiempo medio de detección y remediación (MTTD/MTTR);
  visibilidad centralizada sin costo de licenciamiento adicional; reducción
  de ruido de alarmas gracias a la correlación de eventos.
- **Negativas / trade-offs:** la lógica de correlación en EventBridge añade
  una capa adicional de configuración que debe mantenerse actualizada a
  medida que se agreguen nuevos componentes a la arquitectura.
- **Impacto en costos:** bajo, CloudWatch cobra por métrica personalizada,
  alarma y solicitud de logs; volumen moderado para el alcance del proyecto.
- **Impacto operativo:** alto valor, cierra el ciclo completo entre
  monitoreo (Lección 8), mensajería (Lección 6) y gestión de incidentes
  (tickets en DynamoDB, Lección 3).