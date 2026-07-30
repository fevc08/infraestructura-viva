# Estimación de Costos

## Propósito
Desglosar el costo de cada componente de la arquitectura, distinguiendo
qué cubre el Free Tier de AWS (primeros 12 meses) y cuál sería el costo
mensual aproximado una vez que ese periodo termine o el uso lo exceda.

> Precios de referencia: **US East (N. Virginia) — us-east-1, julio 2026.**
> Son estimaciones para dimensionar la solución, no una cotización final;
> los precios de AWS pueden variar.

## Resumen por componente

| Servicio | Recurso | Costo durante Free Tier (12 meses) | Costo estimado post-Free Tier | Notas |
|---|---|---|---|---|
| Cómputo | EC2 `t3.micro` (Auto Scaling Group) | $0 (750 hrs/mes incluidas) | ~$7.50/mes por instancia | Con 2 instancias en el ASG, post-Free Tier serían ~$15/mes |
| Cómputo | AWS Lambda | $0 (1M solicitudes + 400,000 GB-seg gratis **siempre**, no solo 12 meses) | Prácticamente $0 para el volumen de tickets esperado | Este nivel gratuito de Lambda es permanente, no expira |
| Cómputo | ECS/Fargate | No cubierto por Free Tier | ~$0.04048/vCPU-hora + $0.00444/GB-hora | Bajo por ser un microservicio liviano de uso intermitente |
| Base de datos relacional | RDS `db.t3.micro` (Single-AZ) | $0 (750 hrs/mes + 20 GB incluidos) | ~$12-13/mes | Multi-AZ duplicaría este costo (~$25/mes) |
| Base de datos NoSQL | DynamoDB (On-Demand) | $0 hasta 25 GB + 200M solicitudes/mes (nivel gratuito permanente) | ~$1.25 por millón de escrituras, ~$0.25 por millón de lecturas | Volumen esperado de tickets está muy por debajo del umbral |
| Almacenamiento | S3 Standard | $0 (5 GB incluidos) | ~$0.023/GB/mes | Aplica a `acme-frontend-web`, `acme-static-content`, buckets antes de transición |
| Almacenamiento | S3 Glacier / Deep Archive | No cubierto por Free Tier (volumen bajo) | ~$0.004/GB/mes (Glacier), ~$0.00099/GB/mes (Deep Archive) | Costo marginal, es la razón de ser de esta capa |
| Red | NAT Gateway | **No cubierto por Free Tier** | ~$32.85/mes fijo + $0.045/GB procesado | **Principal costo fijo de la arquitectura** — ver nota abajo |
| Red | Application Load Balancer | **No cubierto por Free Tier** | ~$16.20/mes fijo + cargo por LCU-hora según tráfico | Segundo costo fijo más relevante |
| Mensajería | SNS + SQS | $0 (1M solicitudes/mes incluidas, nivel permanente) | Prácticamente $0 para el volumen esperado | |
| Hosting/CDN | CloudFront | $0 (1 TB de transferencia/mes durante 12 meses) | ~$0.085/GB post-Free Tier | |
| Hosting/CDN | ACM (certificado SSL) | $0 | $0 | Certificados públicos de ACM son gratuitos siempre |
| Monitoreo | CloudWatch (métricas, alarmas, logs) | $0 (10 métricas personalizadas + 10 alarmas incluidas) | ~$0.30/métrica adicional, ~$0.10/alarma adicional | El diseño usa 3 métricas y 2 alarmas — dentro del nivel gratuito |
| DNS | Route 53 | No cubierto por Free Tier | ~$0.50/mes por zona hospedada + $0.40 por millón de consultas | Costo bajo y predecible |

## Costo total estimado

| Escenario | Estimado mensual |
|---|---|
| **Durante Free Tier (primeros 12 meses)** | ~$50-55/mes — casi en su totalidad NAT Gateway + ALB, que **no** están cubiertos por ningún nivel gratuito |
| **Post-Free Tier** (año 2 en adelante) | ~$90-110/mes, dependiendo del volumen real de tráfico y almacenamiento |

## Nota importante: el Free Tier no cubre toda la arquitectura
Es un error común asumir que "usar Free Tier" significa costo cero. En este
diseño, **NAT Gateway y Application Load Balancer nunca están cubiertos por
el Free Tier** — cobran desde la hora uno, tengan o no tráfico. Esto explica
por qué, incluso en el primer año, ACME debe presupuestar
**~$50/mes** aproximados, no $0.

Esta es precisamente la justificación de aceptar ese costo fijo a cambio del
beneficio de seguridad que aportan (ver [ADR-0005](../adr/0005-diseno-red-vpc.md))
en vez de exponer las instancias directamente a Internet para evitarlo.

## Validación práctica
Durante el prototipo desplegado en AWS Academy Learner Lab (ver
[`prototipo/README.md`](../prototipo/README.md)), se usó un presupuesto de
**$50 USD** de créditos, suficiente para desplegar y probar VPC, NAT
Gateway, RDS y EC2 durante varias horas sin agotar el saldo — consistente
con la estimación de la tabla de arriba.

## Recomendaciones para reducir costos a futuro
- Sustituir el NAT Gateway por **VPC Gateway Endpoints** (gratuitos) para
  el tráfico hacia S3 y DynamoDB, que hoy pasa innecesariamente por el NAT.
- Evaluar Reserved Instances o Savings Plans para RDS y EC2 una vez que el
  patrón de uso de ACME sea predecible (ahorros de 30-70%).
- Mantener el uso de Lambda y DynamoDB On-Demand mientras el volumen sea
  bajo — son gratuitos o casi gratuitos a esa escala, y solo conviene
  migrar a capacidad reservada si el volumen crece sostenidamente.