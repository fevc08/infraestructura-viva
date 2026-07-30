# Arquitectura General: Infraestructura Viva (ACME)

## Resumen ejecutivo

Este documento presenta la propuesta de arquitectura cloud para
**Soluciones Digitales ACME**, diseñada para migrar sus sistemas de
ventas, finanzas y soporte desde infraestructura on-premise hacia AWS.
La arquitectura resuelve seis problemáticas centrales (cómputo escalable,
bases de datos relacionales y NoSQL, almacenamiento diferenciado, red
segmentada, mensajería confiable y monitoreo proactivo) combinando
servicios gestionados que reducen la carga operativa del equipo de
Innovación Tecnológica, dentro de los márgenes de costo de Free Tier /
AWS Academy.

## Contexto y problemática

ACME opera actualmente con hardware físico propio, lo que implica costos
de mantenimiento elevados, escalabilidad limitada ante picos de demanda,
y riesgo operativo concentrado en un único punto de falla. El objetivo de
este proyecto es diseñar (no necesariamente desplegar en producción) una
arquitectura que resuelva:

| # | Problemática de ACME | Cómo se resuelve | Lección relacionada |
|---|---|---|---|
| 1 | Cómputo que no escala ante demanda variable | Auto Scaling Group (EC2) + Lambda + ECS/Fargate | [Lección 4](04-computo.md) |
| 2 | Datos transaccionales y semiestructurados mezclados sin optimización | RDS (MySQL) para ventas/finanzas + DynamoDB para tickets de soporte | [Lección 2](02-base-datos-relacional.md), [Lección 3](03-base-datos-nosql.md) |
| 3 | Almacenamiento sin diferenciación de costo por uso | S3 Standard (frecuente) + Glacier (archivado) con lifecycle policies | [Lección 1](01-almacenamiento.md) |
| 4 | Red plana, sin aislamiento entre servicios críticos | VPC segmentada (pública/privada), Security Groups, ALB, VPN opcional | [Lección 5](05-red.md) |
| 5 | Sin visibilidad ni respuesta automática ante incidentes | CloudWatch + EventBridge + Auto Scaling como remediación | [Lección 8](08-monitoreo-incidentes.md) |
| 6 | Comunicación entre sistemas sin garantía de entrega | SNS + SQS + Dead Letter Queue | [Lección 6](06-mensajeria-notificaciones.md) |

Adicionalmente, se resuelve la exposición pública del sitio corporativo de
forma performante mediante S3 + CloudFront ([Lección 7](07-hosting-web.md)).

## Principios de diseño transversales

Estos principios se aplicaron de forma consistente en las 8 lecciones y
quedan detallados en profundidad en [`buenas-practicas.md`](buenas-practicas.md):

- **Seguridad por capas:** ningún servicio de datos (RDS, DynamoDB) es
  alcanzable directamente desde Internet; todo el tráfico pasa por un
  punto de entrada controlado (ALB, CloudFront, API Gateway).
- **Alta disponibilidad:** distribución en múltiples zonas de
  disponibilidad (Multi-AZ en RDS, Auto Scaling Group multi-AZ).
- **Costo optimizado:** uso de Free Tier, capacidad on-demand
  (DynamoDB, Lambda), y transición automática a almacenamiento frío
  (Glacier) para datos poco accedidos.
- **Observabilidad desde el diseño:** cada componente reporta métricas a
  CloudWatch desde el día uno, no como una capa agregada después.
- **Desacoplamiento:** los servicios se comunican mediante colas/tópicos
  (SNS/SQS) en vez de llamadas síncronas directas donde es posible,
  reduciendo el impacto de fallas en cascada.

## Vista general de la arquitectura

![Diagrama de arquitectura general](../diagrams/arquitectura-general.png)

> Diagrama editable disponible en [`diagrams/arquitectura-general.drawio`](../diagrams/arquitectura-general.drawio)

### Recorrido del diagrama (de afuera hacia adentro)

1. **Usuarios finales** acceden al sitio corporativo estático vía
   **Route 53 → CloudFront → S3** (`acme-frontend-web`), y a la aplicación
   transaccional vía **Internet Gateway → ALB → Auto Scaling Group (EC2)**,
   dentro de la VPC.
2. La app en EC2 (subred privada) se conecta a **RDS Multi-AZ** (subred de
   datos, otra zona de disponibilidad) para las operaciones de
   ventas/finanzas.
3. Los tickets de soporte llegan vía **API Gateway → Lambda**, que escribe
   en **DynamoDB** y encola una notificación en **SQS**, consumida por un
   microservicio en **ECS/Fargate**.
4. Las alertas de incidentes se centralizan en **SNS**, que distribuye a
   email, SMS y a la cola de procesamiento automático.
5. **CloudWatch** recolecta métricas de EC2, RDS y Lambda; sus alarmas
   disparan tanto notificaciones (SNS) como acciones automáticas
   (Auto Scaling); **EventBridge** correlaciona múltiples alarmas para
   escalar la severidad de un incidente.
6. Los buckets de archivado (`acme-backups-datos`, `acme-logs-monitoreo`)
   reciben datos desde RDS y CloudWatch Logs respectivamente, y transicionan
   a **Glacier** según su política de ciclo de vida.
7. Opcionalmente, sistemas legados on-premise se conectan a la VPC vía
   **VPN Site-to-Site**, mientras dura la migración completa.

## Resumen de componentes por dominio

| Dominio | Servicio principal | Documento |
|---|---|---|
| Almacenamiento | S3 + Glacier | [01-almacenamiento.md](01-almacenamiento.md) |
| Base de datos relacional | RDS (MySQL) | [02-base-datos-relacional.md](02-base-datos-relacional.md) |
| Base de datos NoSQL | DynamoDB | [03-base-datos-nosql.md](03-base-datos-nosql.md) |
| Cómputo | EC2, Lambda, ECS/Fargate | [04-computo.md](04-computo.md) |
| Red | VPC, ALB, Security Groups | [05-red.md](05-red.md) |
| Mensajería | SNS, SQS | [06-mensajeria-notificaciones.md](06-mensajeria-notificaciones.md) |
| Hosting web | S3 + CloudFront | [07-hosting-web.md](07-hosting-web.md) |
| Monitoreo | CloudWatch, EventBridge | [08-monitoreo-incidentes.md](08-monitoreo-incidentes.md) |

## Decisiones de arquitectura (ADR)

Cada decisión relevante quedó registrada como ADR, incluyendo alternativas
consideradas y consecuencias. Ver índice completo en el
[README principal](../README.md#-índice-de-lecciones-documento--decisión-de-arquitectura).

## Consideraciones de costos

La arquitectura fue diseñada para operar dentro de los límites de **Free
Tier de AWS / AWS Academy Learner Lab**:
- Instancias EC2 y RDS en clases `t3.micro` / `db.t3.micro`.
- DynamoDB y Lambda en modalidad de pago por uso (sin capacidad reservada).
- Uso de Glacier para minimizar el costo de datos de baja frecuencia de
  acceso.
- NAT Gateway y ALB son los principales costos fijos por hora — se
  justifican por el beneficio de seguridad que aportan (ver
  [ADR-0005](../adr/0005-diseno-red-vpc.md)).

Un desglose línea por línea se documenta en
[`costos-estimados.md`](costos-estimados.md).

## Escalabilidad y evolución futura

- **Corto plazo:** agregar Read Replicas a RDS si crece la carga de
  reportería de finanzas.
- **Mediano plazo:** evaluar migración de RDS a Aurora si el volumen
  transaccional supera lo que justifica el diferencial de costo
  (ver alternativas en [ADR-0002](../adr/0002-eleccion-bd-relacional.md)).
- **Largo plazo:** una vez completada la migración, evaluar el retiro de
  la VPN Site-to-Site y desconexión total del entorno on-premise.

## Referencia
Este documento consolida las 8 lecciones del Manual de Arquitectura Cloud
(SOFOFA) según lo solicitado en el Proyecto ABP "Infraestructura Viva".