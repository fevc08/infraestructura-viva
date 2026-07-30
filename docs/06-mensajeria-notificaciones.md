# Notificación y Mensajería (SNS + SQS)

## Objetivo
Establecer un canal de mensajería que desacople los microservicios entre sí
y envíe alertas automáticas a los equipos de soporte/gerencia ante
incidentes, con alta disponibilidad y política de reintentos.

## Componentes utilizados
- **Amazon SNS** (patrón publicador/suscriptor — push)
- **Amazon SQS** (cola estándar — pull, para desacoplar microservicios)
- **Dead Letter Queue (DLQ)** (política de reintentos)

## Diseño propuesto

### Tópico SNS: `acme-alertas-incidentes`
Publicadores: alarmas de CloudWatch (ver Lección 8), función Lambda de
tickets críticos (ver Lección 3).

| # | Suscriptor | Protocolo | Propósito |
|---|---|---|---|
| 1 | Email equipo de soporte | Email | Recibir detalle del incidente para atención inmediata |
| 2 | SMS equipo on-call | SMS | Alerta urgente fuera de horario laboral |
| 3 | `acme-cola-incidentes` (SQS) | SQS | Encolar el incidente para procesamiento automático (ej. por el microservicio ECS de notificaciones, Lección 4) |

### Cola SQS: `acme-cola-tickets-notificaciones`
- Tipo: **Estándar** (no requiere orden estricto, prioriza throughput).
- Productor: función Lambda de creación de tickets (Lección 3).
- Consumidor: microservicio en ECS/Fargate (Lección 4), que procesa la
  notificación y actualiza el estado del ticket.
- **Message Retention Period:** 4 días.
- **Visibility Timeout:** 30 segundos (tiempo que un mensaje queda oculto a
  otros consumidores mientras se procesa).

## Patrón de reintentos y alta disponibilidad
- **Reintentos SQS:** si un consumidor no confirma (ack) el procesamiento de
  un mensaje dentro del Visibility Timeout, el mensaje vuelve a estar
  disponible automáticamente para reintento.
- **maxReceiveCount = 3:** tras 3 intentos fallidos, el mensaje se mueve
  automáticamente a una **Dead Letter Queue (`acme-cola-dlq`)** para
  revisión manual, evitando que un mensaje "envenenado" bloquee la cola
  indefinidamente.
- **Alta disponibilidad:** tanto SNS como SQS replican los mensajes en
  múltiples zonas de disponibilidad de forma nativa (gestionado por AWS,
  sin configuración adicional).
- **Reintentos SNS:** AWS reintenta automáticamente la entrega a
  suscriptores HTTP/S con backoff exponencial ante fallos temporales del
  endpoint.

## Integración con correo/mensaje
La suscripción por Email al tópico `acme-alertas-incidentes` se confirma una
única vez (link de confirmación enviado por SNS), y a partir de ahí el equipo
de soporte recibe automáticamente cada alerta generada por el sistema de
monitoreo.

## Justificación
- **Por qué SNS + SQS combinados (no solo uno):** SNS resuelve "un evento,
  múltiples canales" (email + SMS + cola); SQS resuelve "desacoplar
  productor y consumidor" para que el microservicio de notificaciones
  procese a su propio ritmo sin perder mensajes ante picos de carga.
- **Cola estándar vs. FIFO:** se eligió estándar porque no es crítico el
  orden exacto de los tickets/alertas, y se prioriza mayor throughput.
- **DLQ como red de seguridad:** evita pérdida silenciosa de notificaciones
  críticas ante errores repetidos del consumidor.

## Métricas cubiertas (según pauta de evaluación)
- ✅ 1 SNS + 1 SQS funcionales (mínimo)
- ✅ Hasta 3 suscriptores o integraciones externas (email, SMS, SQS)
- ✅ 1 política de reintento aplicada (maxReceiveCount + DLQ)

## Referencia
Basado en Manual #6: Servicios de notificación y mensajería (comparativa
SNS vs. SQS, colas estándar vs. FIFO, y consideraciones de seguridad/costos).