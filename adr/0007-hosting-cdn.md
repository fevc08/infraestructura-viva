# ADR-0007: Alojamiento de contenido estático vía S3 + CloudFront

## Estado
Aceptado

## Fecha
2026-07-26

## Contexto
ACME necesita publicar contenido estático (portal informativo, recursos de
marketing) con buena performance para usuarios en distintas ubicaciones
geográficas, sin incurrir en el costo de mantener un servidor web dedicado
solo para esto.

## Decisión
Se alojará el contenido estático en un bucket **Amazon S3** con hosting
estático habilitado, distribuido globalmente mediante **Amazon CloudFront**,
con acceso al bucket restringido únicamente a través de CloudFront (OAC) y
certificado SSL gestionado por **ACM**.

## Alternativas consideradas
| Opción | Pros | Contras | ¿Por qué no se eligió? |
|---|---|---|---|
| Amazon Lightsail | Hosting "todo en uno" simple, costo fijo mensual | Requiere administrar una instancia (parches, disponibilidad); pensado para sitios con lógica de servidor | El contenido es puramente estático; no se necesita un servidor completo |
| Servir el contenido estático desde la misma EC2 de la app | Un solo servidor que administrar | Mezcla responsabilidades (app dinámica + estático); no aprovecha caché global; consume recursos de cómputo innecesariamente | Ineficiente en costo y rendimiento comparado con S3+CloudFront |
| Bucket S3 público (sin CloudFront) | Configuración más simple | Sin caché global (mayor latencia para usuarios lejanos), sin mitigación DDoS integrada, bucket expuesto directamente a Internet | No cumple con el objetivo de "buena performance global" ni con buenas prácticas de seguridad |

## Consecuencias
- **Positivas:** menor costo operativo, mejor tiempo de respuesta global,
  superficie de exposición del bucket reducida al mínimo.
- **Negativas / trade-offs:** la invalidación de caché de CloudFront ante
  actualizaciones de contenido requiere un paso adicional (crear una
  invalidation) para que los usuarios vean cambios de inmediato.
- **Impacto en costos:** favorable, se paga por GB transferido y
  solicitudes, generalmente menor que mantener cómputo dedicado.
- **Impacto en seguridad:** alto, el uso de OAC impide el acceso directo
  al bucket, y ACM provee HTTPS sin costo adicional.