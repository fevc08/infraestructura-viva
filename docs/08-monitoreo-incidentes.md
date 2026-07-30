# Monitoreo y Correlación de Incidentes (Amazon CloudWatch)

## Objetivo
Monitorizar los componentes clave de la arquitectura (cómputo, base de
datos, red) y configurar alarmas con notificación automática y acciones de
remediación, correlacionando eventos para responder proactivamente ante
incidentes.

## Componentes utilizados
- **Amazon CloudWatch Metrics** (métricas de cómputo, BD y red)
- **Amazon CloudWatch Alarms** (umbrales + acciones)
- **Amazon CloudWatch Logs** (centralización de logs de aplicación)
- **Amazon EventBridge** (correlación de eventos y disparo de acciones)
- **Integración con SNS** (tópico `acme-alertas-incidentes`, ver Lección 6)

## Métricas clave monitorizadas (3, según pauta)

| # | Métrica | Origen | Qué detecta |
|---|---|---|---|
| 1 | `CPUUtilization` | EC2 (Auto Scaling Group, app ventas/finanzas) | Saturación de cómputo que puede degradar la app |
| 2 | `DatabaseConnections` / `CPUUtilization` (RDS) | Amazon RDS | Presión sobre la base de datos transaccional |
| 3 | `Errors` (invocaciones fallidas) | AWS Lambda (función de tickets) | Fallos en el procesamiento de tickets de soporte |

## Alarmas configuradas (2, con notificación SNS)

| # | Alarma | Condición | Notificación |
|---|---|---|---|
| 1 | `alarma-cpu-alta-ec2` | `CPUUtilization` > 80% durante 10 min | Publica en `acme-alertas-incidentes` (SNS) → email + SMS al equipo de soporte |
| 2 | `alarma-errores-lambda` | `Errors` > 5 en ventana de 5 min | Publica en `acme-alertas-incidentes` (SNS) → email al equipo técnico |

## Acción automática documentada (1)
Cuando se dispara `alarma-cpu-alta-ec2`, además de la notificación por SNS,
se ejecuta una **política de Auto Scaling (Target Tracking)** que agrega
automáticamente una instancia EC2 al grupo (ver Lección 4 – prueba de
escalabilidad), sin intervención humana. Esto reduce el tiempo de respuesta
ante picos de carga de minutos (intervención manual) a segundos
(remediación automática).

## Correlación de eventos
Se usa **Amazon EventBridge** como bus de eventos para correlacionar
señales de distintos servicios:
- Si `alarma-cpu-alta-ec2` y `alarma-errores-lambda` se disparan dentro de
  la misma ventana de 15 minutos, EventBridge lo interpreta como un
  **incidente de severidad alta** (posible causa común, ej. un pico de
  tráfico general) y dispara una regla que:
  1. Publica un mensaje de severidad "alta" al tópico SNS (distinto del
     mensaje individual de cada alarma).
  2. Invoca una función Lambda que crea automáticamente un ticket en
     `acme-tickets-soporte` (DynamoDB, ver Lección 3) con prioridad "alta",
     cerrando el ciclo entre monitoreo y gestión de incidentes.

## Protocolo de actuación (resumen)
1. **Detección:** CloudWatch identifica la métrica fuera de umbral.
2. **Notificación:** SNS avisa al equipo correspondiente (email/SMS).
3. **Remediación automática (si aplica):** Auto Scaling agrega capacidad.
4. **Registro:** si hay correlación de múltiples alarmas, se crea un ticket
   automático de alta prioridad para seguimiento humano.
5. **Post-mortem:** el equipo revisa CloudWatch Logs para identificar causa
   raíz y documentar el incidente.

## Justificación
- **Por qué CloudWatch (y no una herramienta de terceros como Datadog):**
  integración nativa y sin costo adicional de licenciamiento con todos los
  servicios ya usados (EC2, RDS, Lambda, SNS), ideal para una primera
  migración con foco en minimizar costos (Free Tier/AWS Academy).
- **Por qué correlacionar eventos (y no solo alarmas aisladas):** permite
  distinguir un incidente puntual (una sola alarma) de un incidente
  sistémico (varias alarmas simultáneas), priorizando la respuesta del
  equipo humano donde realmente se necesita.

## Métricas cubiertas (según pauta de evaluación)
- ✅ 3 métricas clave monitorizadas (CPU EC2, RDS, Errores Lambda)
- ✅ 2 alarmas con notificación SNS
- ✅ 1 acción automática documentada (Auto Scaling ante CPU alta)

## Referencia
Basado en Manual #8: Servicios de monitoreo y correlación de incidentes
(CloudWatch: métricas, logs, eventos/EventBridge y alarmas).