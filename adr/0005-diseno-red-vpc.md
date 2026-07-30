# ADR-0005: Diseño de red segmentada (VPC con subredes públicas/privadas)

## Estado
Aceptado

## Fecha
2026-07-26

## Contexto
ACME necesita aislar sus servicios internos (aplicación y base de datos) de
Internet, exponiendo únicamente lo estrictamente necesario, y distribuir el
tráfico web de forma balanceada y tolerante a fallos.

## Decisión
Se implementará una **VPC (10.0.0.0/16)** con segmentación en 3 subredes
(pública, privada de aplicación, privada de datos), Security Groups en
cadena (Internet → ALB → App → RDS) y un **Application Load Balancer**
como único punto de entrada público.

## Alternativas consideradas
| Opción | Pros | Contras | ¿Por qué no se eligió? |
|---|---|---|---|
| Todo en subred pública (sin segmentación) | Configuración más simple | RDS y EC2 quedarían expuestos directamente a Internet | Viola el requerimiento explícito de aislar servicios internos |
| Solo Security Groups, sin NACLs | Menos configuración | Una sola capa de defensa; un error en el SG expone todo | Se prefiere defensa en profundidad (2 capas) |
| AWS Direct Connect en vez de VPN | Rendimiento estable, sin pasar por Internet pública | Costo elevado, requiere enlace físico dedicado; no aplica a un proyecto en Free Tier/Academy | Sobredimensionado para el alcance y presupuesto de este proyecto |

## Consecuencias
- **Positivas:** superficie de ataque mínima (solo el ALB es alcanzable
  desde Internet); tolerancia a fallos gracias a distribución en 2 AZ;
  cumple el requerimiento de aislar servicios internos vs. exponer
  aplicaciones críticas de forma segura.
- **Negativas / trade-offs:** mayor complejidad de configuración y más
  componentes que mantener (NAT Gateway, múltiples Security Groups, NACLs).
- **Impacto en costos:** el NAT Gateway y el ALB generan costo por hora +
  por GB procesado; se considera necesario y razonable dado el beneficio
  de seguridad.
- **Impacto en escalabilidad:** el diseño multi-AZ permite que el Auto
  Scaling Group (ADR-0001) y el Multi-AZ de RDS (ADR-0002) operen sin
  cambios adicionales de red.

## Notas de implementación real
Durante el despliegue del prototipo en AWS Academy Learner Lab, los nombres
de los Security Groups se ajustaron de `sg-app`/`sg-rds` a
**`acme-sg-app`/`acme-sg-rds`**, ya que AWS reserva el prefijo `sg-` para
identificadores autogenerados. La decisión y el diseño de red documentados
en este ADR no cambian — solo la convención de nombres.