# ADR-0002: Elección de motor y configuración de base de datos relacional

## Estado
Aceptado

## Fecha
2026-07-26

## Contexto
Los departamentos de ventas y finanzas requieren una base de datos
transaccional confiable, sin depender de mantenimiento de hardware físico,
con alta disponibilidad y backups automáticos, evitando sobrecostos y
minimizando el trabajo operativo del equipo de Innovación Tecnológica.

## Decisión
Se utilizará **Amazon RDS con motor MySQL**, en clase de instancia
`db.t3.micro` (Free Tier), con **Multi-AZ habilitado** y **backups
automáticos** con retención de 7 días.

## Alternativas consideradas
| Opción | Pros | Contras | ¿Por qué no se eligió? |
|---|---|---|---|
| RDS PostgreSQL | Muy buena documentación, robusto para apps educativas | Equipo tiene más experiencia previa con MySQL | Sin ventaja clara para este caso de uso |
| Amazon Aurora (MySQL compatible) | Hasta 5x más rendimiento, 6 copias en 3 AZ, hasta 15 read replicas | Mayor costo, pensado para cargas de misión crítica de gran escala | Sobredimensionado para el volumen inicial de ACME; no es Free Tier |
| VM propia (EC2) con MySQL instalado manualmente | Control total de configuración | Requiere gestionar parches, backups y HA manualmente | Contradice el objetivo de reducir carga operativa y dependencia de hardware |

## Consecuencias
- **Positivas:** administración simplificada (AWS gestiona backups, parches,
  HA), failover automático ante fallas de zona de disponibilidad, modelo de
  costos predecible dentro de Free Tier.
- **Negativas / trade-offs:** menor control sobre configuración fina del
  motor comparado con una instalación propia; Multi-AZ implica un costo
  adicional (mitigado en esta etapa por Free Tier/AWS Academy).
- **Impacto en costos:** favorable en el corto plazo (Free Tier); se deberá
  reevaluar al escalar (posible migración a Aurora si el tráfico crece
  significativamente — ver sección "Escalabilidad futura").
- **Impacto en seguridad:** la instancia se despliega en subred privada,
  sin acceso público directo, solo alcanzable desde la capa de aplicación
  (detalle en ADR-0005, diseño de red).