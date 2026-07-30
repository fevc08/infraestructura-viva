# Plan de Monitoreo y Notificaciones

## Objetivo
Detectar de forma proactiva condiciones anómalas en los componentes
críticos de la arquitectura, notificar al equipo correspondiente en menos
de 1 minuto desde que se cruza un umbral, y ejecutar remediación
automática cuando sea seguro hacerlo sin intervención humana.

## Alcance
Este plan cubre los 3 componentes con mayor impacto ante una falla:
cómputo (EC2), base de datos relacional (RDS) y procesamiento serverless
de tickets (Lambda). Ver [`docs/08-monitoreo-incidentes.md`](../docs/08-monitoreo-incidentes.md)
para el diseño conceptual completo.

---

## 1. Métricas monitorizadas

| # | Métrica (CloudWatch) | Namespace | Estadística | Período | Componente |
|---|---|---|---|---|---|
| 1 | `CPUUtilization` | `AWS/EC2` | Promedio | 5 min | Instancias del Auto Scaling Group |
| 2 | `CPUUtilization` / `DatabaseConnections` | `AWS/RDS` | Promedio | 5 min | Instancia RDS primaria |
| 3 | `Errors` | `AWS/Lambda` | Suma | 5 min | Función `tickets-processor` |

### Métricas complementarias (monitoreadas, sin alarma dedicada)
| Métrica | Namespace | Propósito |
|---|---|---|
| `Duration` | `AWS/Lambda` | Detectar degradación de performance antes de que cause errores |
| `NetworkIn` / `NetworkOut` | `AWS/EC2` | Contexto adicional ante picos de CPU |
| `FreeStorageSpace` | `AWS/RDS` | Prevenir quedarse sin espacio en disco |
| `ApproximateNumberOfMessagesVisible` | `AWS/SQS` | Detectar acumulación en la cola antes de que escale a incidente |

---

## 2. Alarmas configuradas

### Alarma 1 — CPU alta en EC2
| Parámetro | Valor |
|---|---|
| Nombre | `alarma-cpu-alta-ec2` |
| Métrica | `AWS/EC2 → CPUUtilization` |
| Estadística | Promedio |
| Umbral | `> 80%` |
| Período de evaluación | 5 minutos, 2 datapoints consecutivos (10 min sostenido) |
| Operador de comparación | `GreaterThanThreshold` |
| Acción en estado ALARM | 1) Publicar en SNS `acme-alertas-incidentes` · 2) Ejecutar política de Auto Scaling (target tracking, +1 instancia) |
| Acción en estado OK | Publicar en SNS (recuperación) — informativo |
| Severidad | Media (alta si se correlaciona con otra alarma, ver sección 4) |

### Alarma 2 — Errores en Lambda de tickets
| Parámetro | Valor |
|---|---|
| Nombre | `alarma-errores-lambda` |
| Métrica | `AWS/Lambda → Errors` |
| Estadística | Suma |
| Umbral | `> 5` errores |
| Período de evaluación | 5 minutos, 1 datapoint |
| Operador de comparación | `GreaterThanThreshold` |
| Acción en estado ALARM | Publicar en SNS `acme-alertas-incidentes` |
| Severidad | Alta (afecta directamente la creación de tickets de soporte) |

---

## 3. Canales de notificación

| Canal | Suscriptor | Cuándo se usa |
|---|---|---|
| Email | Equipo técnico / soporte | Toda alarma, siempre |
| SMS | Equipo on-call | Solo alarmas de severidad alta o incidentes correlacionados |
| SQS (`acme-cola-incidentes`) | Procesamiento automático | Todas las alarmas — permite creación automática de tickets |

---

## 4. Correlación de eventos y escalamiento de severidad

Regla en **EventBridge**: si `alarma-cpu-alta-ec2` y `alarma-errores-lambda`
pasan a estado `ALARM` dentro de una ventana de **15 minutos**, se
interpreta como **incidente de severidad alta** (posible causa común, ej.
pico de tráfico general), y se ejecuta:

1. Publicación de un mensaje de severidad "alta" en SNS (distinto del
   mensaje individual de cada alarma).
2. Invocación de una función Lambda que crea automáticamente un ticket en
   `acme-tickets-soporte` (DynamoDB) con `prioridad = alta`.
3. Notificación por **SMS** al equipo on-call (no solo email).

| Escenario | Severidad | Notificación | Acción automática |
|---|---|---|---|
| Solo `alarma-cpu-alta-ec2` | Media | Email | Auto Scaling (+1 instancia) |
| Solo `alarma-errores-lambda` | Alta | Email | Ninguna (requiere revisión humana) |
| Ambas alarmas correlacionadas (≤15 min) | **Alta / Crítica** | Email + SMS | Auto Scaling + ticket automático en DynamoDB |

---

## 5. Protocolo de actuación (runbook)

1. **Detección:** CloudWatch identifica la métrica fuera de umbral y
   transiciona la alarma a estado `ALARM`.
2. **Notificación inmediata:** SNS distribuye el aviso a los canales
   correspondientes según la tabla de la sección 4.
3. **Remediación automática (si aplica):** Auto Scaling agrega capacidad
   sin intervención humana (solo para la alarma de CPU).
4. **Triage humano:** el equipo revisa `CloudWatch Logs` de la instancia
   o función afectada para identificar causa raíz.
5. **Registro:** si hubo correlación de eventos, ya existe un ticket
   automático en DynamoDB — el equipo lo actualiza con su diagnóstico y
   resolución.
6. **Cierre y post-mortem:** una vez resuelto, se documenta la causa raíz
   y, si corresponde, se ajustan los umbrales de este plan (ver sección 6).

---

## 6. Revisión y mejora continua
- Los umbrales de este plan se revisan **trimestralmente** o inmediatamente
  después de un incidente que revele que un umbral era demasiado sensible
  (falsos positivos) o insuficiente (detección tardía).
- Toda alarma que dispare 3+ veces por falsos positivos en un mes debe
  ser recalibrada o documentarse por qué se mantiene así.

---

## 7. Evidencia
Las capturas de pantalla de las alarmas en estado `ALARM`/`OK`, el
historial de eventos de Auto Scaling, y los tickets generados
automáticamente se documentan en [`monitoreo/evidencias/`](evidencias/).

## Métricas cubiertas (según pauta de evaluación)
- ✅ 3 métricas clave monitorizadas
- ✅ 2 alarmas con notificación SNS
- ✅ 1 acción automática documentada (Auto Scaling ante CPU alta)
- ✅ Protocolo de actuación y correlación de eventos documentado