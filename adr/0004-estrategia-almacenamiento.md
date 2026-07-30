# ADR-0004: Estrategia de almacenamiento diferenciado (S3 + Glacier)

## Estado
Aceptado

## Fecha
2026-07-26

## Contexto
ACME necesita un almacenamiento confiable que separe datos de uso frecuente
(contenido estático de las apps) de datos de archivado (backups, logs
históricos), sin depender de hardware físico, y minimizando costos mediante
herramientas de free tier.

## Decisión
Se utilizará **Amazon S3** como almacenamiento primario de objetos (clase
Standard) para contenido de acceso frecuente, y **S3 Lifecycle Policies**
para transicionar automáticamente los datos de respaldo y logs hacia
**S3 Glacier / Glacier Deep Archive** según su antigüedad.

## Alternativas consideradas
| Opción | Pros | Contras | ¿Por qué no se eligió? |
|---|---|---|---|
| Solo S3 Standard para todo | Simplicidad, acceso siempre rápido | Costo elevado para datos poco usados | No optimiza costos de archivado |
| EFS para todo | Sistema de archivos compartido, alto rendimiento | Costo por capacidad y operaciones más alto; pensado para acceso concurrente, no archivado | No es la herramienta correcta para backups fríos |
| Storage Gateway on-premise + nube | Permite mantener caché local | Requiere mantener infraestructura híbrida; mayor complejidad operativa | Contradice el objetivo de reducir dependencia on-premise |

## Consecuencias
- **Positivas:** reducción de costos de almacenamiento a largo plazo,
  eliminación de mantenimiento de hardware, alta durabilidad garantizada.
- **Negativas / trade-offs:** recuperar datos desde Glacier toma minutos u
  horas (no apto para datos que requieran acceso inmediato); requiere
  definir bien las reglas de ciclo de vida para no mover datos "vivos" por error.
- **Impacto en costos:** favorable, se paga tarifa premium solo por lo que
  realmente se accede con frecuencia.
- **Impacto en seguridad:** se recomienda cifrado en reposo (SSE-S3) y
  políticas de bucket restringidas por IAM (se detalla en la guía de buenas
  prácticas).