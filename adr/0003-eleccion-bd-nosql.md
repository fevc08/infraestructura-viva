# ADR-0003: Elección de base de datos NoSQL para datos semiestructurados

## Estado
Aceptado

## Fecha
2026-07-26

## Contexto
El área de soporte genera datos con estructura variable (tickets por
distintos canales, con atributos que difieren entre sí). Se requiere un
almacenamiento flexible, de baja latencia y escalado automático, sin
depender de hardware físico.

## Decisión
Se utilizará **Amazon DynamoDB** con modo de capacidad **On-Demand**, tabla
`acme-tickets-soporte`, con 3 Índices Secundarios Globales (GSI) para cubrir
los patrones de consulta más frecuentes del equipo de soporte.

## Alternativas consideradas
| Opción | Pros | Contras | ¿Por qué no se eligió? |
|---|---|---|---|
| Amazon DocumentDB | Compatible con API MongoDB, útil si hubiera migración desde Mongo | Requiere gestionar un clúster (más cercano a instancia gestionada que serverless) | No hay migración previa desde MongoDB; DynamoDB es más simple y económico para este caso |
| Modelar todo en RDS con tablas flexibles (JSON columns) | Un solo motor de base de datos para todo | Pierde las ventajas de escalado horizontal automático de un NoSQL nativo; consultas sobre JSON menos eficientes | No resuelve el requerimiento explícito de integrar una solución NoSQL |
| ElastiCache (Redis) | Latencia ultrabaja | Es una caché en memoria, no un almacenamiento persistente principal | No es un reemplazo de base de datos persistente, sino un complemento |

## Consecuencias
- **Positivas:** escalado automático sin intervención, integración nativa
  con Lambda (arquitectura serverless), modelo de costos por solicitud
  favorable en etapas iniciales.
- **Negativas / trade-offs:** el modelado de datos NoSQL requiere pensar de
  antemano los patrones de acceso (los GSI no se pueden modificar
  libremente después sin costo); consistencia eventual en lecturas desde
  GSI (a diferencia de la lectura fuertemente consistente en la tabla base).
- **Impacto en costos:** favorable — On-Demand evita sobreaprovisionar
  capacidad; ideal para el volumen inicial de tickets de ACME.
- **Impacto en seguridad:** acceso controlado mediante roles IAM asignados
  a la función Lambda (principio de menor privilegio — se detalla en la
  guía de buenas prácticas).