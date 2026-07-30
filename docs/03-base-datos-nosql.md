# Base de Datos NoSQL (Amazon DynamoDB)

## Objetivo
Desplegar una base de datos NoSQL para manejar datos semiestructurados del
área de soporte (tickets), que varían en atributos según el canal y tipo de
consulta, con baja latencia y escalado automático.

## Por qué NoSQL (y no solo RDS)
Los datos de soporte (tickets, interacciones con clientes) no siempre tienen
el mismo esquema: un ticket por chat puede tener campos distintos a uno por
teléfono. Forzarlos a un esquema relacional rígido añadiría complejidad
innecesaria. DynamoDB permite flexibilidad de atributos y escalado horizontal
automático sin gestión de servidores.

## Componentes utilizados
- **Amazon DynamoDB** (modelo clave-valor/documento)
- **Modo de capacidad On-Demand** (pago por solicitud, ideal para free tier
  y cargas impredecibles)
- **Global Secondary Indexes (GSI)** para consultas rápidas por distintos
  atributos
- **Point-in-Time Recovery (PITR)** como estrategia de respaldo

## Diseño de la tabla: `acme-tickets-soporte`

| Atributo | Tipo | Rol |
|---|---|---|
| `ticket_id` | String (UUID) | Partition Key (clave primaria) |
| `cliente_id` | String | Atributo / clave de índice |
| `canal` | String (email, chat, telefono) | Atributo |
| `prioridad` | String (alta, media, baja) | Atributo |
| `estado` | String (abierto, en_progreso, cerrado) | Atributo / clave de índice |
| `agente_asignado` | String | Atributo / clave de índice |
| `asunto` | String | Atributo |
| `descripcion` | String | Atributo |
| `fecha_creacion` | String (ISO 8601) | Atributo / clave de ordenamiento en índices |
| `fecha_actualizacion` | String (ISO 8601) | Atributo |

## Índices secundarios (GSI)

| Índice | Partition Key | Sort Key | Consulta que resuelve |
|---|---|---|---|
| `estado-fecha-index` | `estado` | `fecha_creacion` | "Todos los tickets abiertos, ordenados por fecha" (dashboard de soporte) |
| `cliente-index` | `cliente_id` | `fecha_creacion` | "Historial de tickets de un cliente específico" |
| `agente-estado-index` | `agente_asignado` | `estado` | "Tickets asignados a un agente, filtrados por estado" |

## Integración con una aplicación de ejemplo
Flujo propuesto (serverless, se detalla en Lección 4 – Cómputo):
1. Un formulario web (o app de soporte) envía un `POST /tickets` a
   **API Gateway**.
2. API Gateway invoca una función **AWS Lambda**.
3. La función Lambda valida los datos y ejecuta `put_item` sobre
   `acme-tickets-soporte` mediante el SDK de AWS (boto3).
4. Se retorna el `ticket_id` generado como confirmación.

Esta integración demuestra el uso de DynamoDB como backend de una app real,
sin necesidad de servidores dedicados.

## Estrategia de respaldo
- **Point-in-Time Recovery (PITR)** habilitado: permite restaurar la tabla a
  cualquier segundo dentro de los últimos 35 días.
- **On-Demand Backup** manual antes de cambios estructurales importantes
  (ej. antes de agregar un nuevo GSI en producción).

## Justificación
- **Escalado automático:** DynamoDB ajusta capacidad sin intervención
  manual, ideal ante picos de tickets (ej. incidentes masivos).
- **Baja latencia:** consultas de milisegundos, adecuado para un dashboard
  de soporte en tiempo real.
- **Costo:** modo On-Demand evita pagar por capacidad no utilizada,
  alineado con el uso de Free Tier/AWS Academy.
- **Complementa a RDS:** mientras RDS maneja datos transaccionales
  estructurados (ventas/finanzas), DynamoDB maneja datos semiestructurados
  y de alta variabilidad (soporte).

## Métricas cubiertas (según pauta de evaluación)
- ✅ 1 tabla NoSQL creada (`acme-tickets-soporte`)
- ✅ Hasta 3 índices configurados (3 GSI detallados arriba)
- ✅ 1 integración funcional con una aplicación (flujo API Gateway → Lambda → DynamoDB)

## Referencia
Basado en Manual #3: Servicios de bases de datos NoSQL (comparativa
DynamoDB, DocumentDB, Redshift, Neptune, ElastiCache y casos de uso).