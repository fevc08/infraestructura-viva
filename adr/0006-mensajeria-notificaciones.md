# ADR-0006: Arquitectura de mensajería con SNS + SQS y política de reintentos

## Estado
Aceptado

## Fecha
2026-07-26

## Contexto
Se requiere comunicar eventos e incidentes a múltiples equipos y canales
(soporte, gerencia, sistemas automatizados), además de desacoplar la
creación de tickets del procesamiento de sus notificaciones, garantizando
que ningún mensaje se pierda ante fallos temporales.

## Decisión
Se combinará **Amazon SNS** (tópico `acme-alertas-incidentes`, patrón
pub/sub hacia email, SMS y una cola SQS) con **Amazon SQS** (cola estándar
`acme-cola-tickets-notificaciones` + Dead Letter Queue `acme-cola-dlq`) para
desacoplar productores y consumidores, con `maxReceiveCount = 3` como
política de reintentos.

## Alternativas consideradas
| Opción | Pros | Contras | ¿Por qué no se eligió? |
|---|---|---|---|
| Solo SNS (sin SQS) | Más simple, un solo servicio | Si un suscriptor tipo Lambda/ECS falla, no hay cola de espera persistente para reintentar de forma controlada por el consumidor | No cubre el caso de desacoplar productor/consumidor con control de flujo |
| Solo SQS (sin SNS) | Simplicidad para un solo consumidor | No permite enviar el mismo evento a múltiples canales (email + SMS + cola) sin lógica adicional en la app | No resuelve el requerimiento de notificar a distintos equipos simultáneamente |
| Cola FIFO en vez de Estándar | Garantiza orden estricto | Menor throughput; el caso de uso (notificaciones/tickets) no requiere orden estricto | Costo de rendimiento sin beneficio real para este caso |

## Consecuencias
- **Positivas:** un solo evento puede notificar a múltiples canales sin
  acoplar el productor a la lógica de cada receptor; los mensajes no se
  pierden ante fallos temporales del consumidor gracias a la DLQ.
- **Negativas / trade-offs:** mayor cantidad de componentes a monitorear
  (tópico, cola principal, DLQ); requiere definir bien el
  `Visibility Timeout` para evitar procesamiento duplicado.
- **Impacto en costos:** bajo — se cobra por solicitud/mensaje; el volumen
  esperado de incidentes y tickets es moderado.
- **Impacto en confiabilidad:** alto — la combinación SNS+SQS+DLQ es un
  patrón estándar de la industria para mensajería resiliente.