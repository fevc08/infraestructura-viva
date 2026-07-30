# Guía de Buenas Prácticas

## Propósito
Este documento consolida las consideraciones de **seguridad**,
**escalabilidad**, **alta disponibilidad** y **administración de red**
aplicadas transversalmente en las 8 lecciones de la arquitectura
"Infraestructura Viva", en línea con los manuales del módulo.

---

## 1. Seguridad

### 1.1 Firewall (Security Groups y NACLs)
Se aplicó **defensa en profundidad** con dos capas de firewall:

| Capa | Herramienta | Alcance | Detalle |
|---|---|---|---|
| Instancia | Security Groups (con estado) | EC2, RDS, ALB | Cadena `Internet → sg-alb → sg-app → sg-rds`, cada salto solo permite el origen inmediato anterior (ver [ADR-0005](../adr/0005-diseno-red-vpc.md)) |
| Subred | Network ACLs (sin estado) | Subred de datos | Regla explícita: solo tráfico desde la subred de aplicación; todo lo demás se deniega |

**Regla general aplicada:** ningún recurso de datos (RDS, DynamoDB) es
accesible directamente desde Internet. Todo el tráfico entrante pasa por
un único punto controlado: el ALB (app) o CloudFront/API Gateway
(contenido estático y tickets).

### 1.2 Roles de acceso (IAM — principio de menor privilegio)
| Componente | Rol asignado | Permisos mínimos necesarios |
|---|---|---|
| EC2 (app) | `role-ec2-app` | Lectura de secretos de conexión a RDS (Secrets Manager), escritura de logs a CloudWatch |
| Lambda (tickets) | `role-lambda-tickets` | `dynamodb:PutItem` / `GetItem` solo sobre `acme-tickets-soporte`; `sqs:SendMessage` solo sobre la cola de notificaciones |
| ECS/Fargate | `role-ecs-notificaciones` | `sqs:ReceiveMessage` / `DeleteMessage` solo sobre su cola asignada |
| CloudFront (OAC) | Identidad de servicio | Solo `s3:GetObject` sobre `acme-frontend-web`, sin acceso de escritura |

Ningún componente usa credenciales de administrador ni claves de acceso
hardcodeadas — todo el acceso entre servicios se resuelve vía **roles
IAM asumidos**, nunca vía usuarios/contraseñas embebidos en código o
configuración.

### 1.3 Cifrado
- **En tránsito:** HTTPS/TLS en todas las conexiones públicas (ALB,
  CloudFront vía ACM); túnel IPSec para la VPN Site-to-Site on-premise.
- **En reposo:** cifrado nativo habilitado en S3 (SSE-S3), RDS
  (encryption at rest) y DynamoDB (cifrado por defecto).

### 1.4 Exposición mínima
- Buckets S3 de datos y logs: **sin acceso público**, política de bucket
  restringida.
- Bucket de frontend: acceso público **solo a través de CloudFront (OAC)**,
  nunca directo.
- RDS y DynamoDB: sin endpoint público, alcanzables solo desde la app.

---

## 2. Escalabilidad

| Servicio | Mecanismo de escalado | Tipo | Lección |
|---|---|---|---|
| EC2 (app) | Auto Scaling Group, target tracking por CPU | Horizontal | [04](04-computo.md) |
| RDS | Cambio de clase de instancia; Read Replicas a futuro | Vertical / horizontal (lectura) | [02](02-base-datos-relacional.md) |
| DynamoDB | Modo On-Demand (escala automática por solicitud) | Horizontal, automático | [03](03-base-datos-nosql.md) |
| Lambda | Escalado automático por invocación concurrente | Horizontal, automático | [04](04-computo.md) |
| ECS/Fargate | Escalado por profundidad de cola SQS | Horizontal | [04](04-computo.md), [06](06-mensajeria-notificaciones.md) |
| CloudFront | Escalado global nativo (Edge Locations) | Horizontal, gestionado por AWS | [07](07-hosting-web.md) |

**Principio aplicado:** priorizar escalado horizontal y automático sobre
el vertical manual, salvo en RDS donde el escalado vertical es la primera
palanca disponible antes de introducir la complejidad de réplicas.

---

## 3. Alta disponibilidad y estrategias de replicación

| Componente | Estrategia de replicación / HA | Detalle |
|---|---|---|
| RDS | **Multi-AZ** | Réplica sincrónica en zona de disponibilidad distinta; failover automático ante falla (ver [ADR-0002](../adr/0002-eleccion-bd-relacional.md)) |
| EC2 (app) | Auto Scaling Group **multi-AZ** | Instancias distribuidas en al menos 2 AZ, el ALB redirige tráfico solo a instancias saludables |
| S3 / DynamoDB | Replicación nativa multi-AZ (gestionada por AWS) | No requiere configuración adicional — parte del SLA del servicio |
| SNS / SQS | Alta disponibilidad nativa multi-AZ | Mensajes replicados automáticamente por el servicio gestionado |
| Datos críticos (RDS) | Backups automáticos + snapshots diarios | Retención 7 días, restauración point-in-time |
| Datos NoSQL | Point-in-Time Recovery (PITR) | Restauración a cualquier segundo dentro de 35 días |

**Principio aplicado:** ningún componente con estado (RDS, DynamoDB, S3)
depende de una única zona de disponibilidad. El cómputo sin estado (EC2,
Lambda) se reemplaza automáticamente ante falla; el cómputo con estado
persistente se replica de forma síncrona (RDS) o nativa (DynamoDB/S3).

---

## 4. Administración de red

- **Segmentación por capa:** subred pública (entrada), subred privada de
  aplicación, subred privada de datos — cada una con su propósito único
  (ver [ADR-0005](../adr/0005-diseno-red-vpc.md)).
- **Salida controlada:** las subredes privadas usan **NAT Gateway** para
  tráfico saliente (parches, APIs externas), sin exponer IP pública a las
  instancias internas.
- **Punto único de entrada:** todo el tráfico web público converge en el
  **ALB**, facilitando auditoría y aplicación centralizada de reglas.
- **Conectividad híbrida:** VPN Site-to-Site (IPSec) para sistemas
  on-premise aún no migrados, evitando exponerlos directamente a Internet
  durante la transición.
- **DNS gestionado:** Route 53 para resolución de nombres y, a futuro,
  health checks con failover automático entre endpoints.

---

## 5. Costos y viabilidad (Free Tier / AWS Academy)
- Clases de instancia `t3.micro` / `db.t3.micro` en EC2 y RDS.
- DynamoDB y Lambda en modalidad pago-por-uso, sin capacidad reservada.
- Transición automática a Glacier para minimizar costo de datos fríos.
- Evaluación explícita de trade-off costo/beneficio en cada ADR antes de
  elegir el servicio (ej. Aurora vs. RDS estándar, Fargate vs. EC2 para
  el microservicio de notificaciones).

---

## 6. Observabilidad como práctica transversal
Cada componente reporta métricas a CloudWatch desde su diseño inicial
(no como capa agregada después), permitiendo detectar degradación antes
de que se convierta en incidente (ver [Lección 8](08-monitoreo-incidentes.md)).

---

## Checklist resumen

- [x] Firewall en 2 capas (Security Groups + NACL)
- [x] Roles IAM de menor privilegio por componente (sin credenciales embebidas)
- [x] Cifrado en tránsito y en reposo en todos los servicios de datos
- [x] Sin exposición pública directa de bases de datos ni buckets de datos
- [x] Escalado horizontal automático donde es posible
- [x] Alta disponibilidad multi-AZ en todos los componentes con estado
- [x] Backups y estrategias de replicación documentadas por componente
- [x] Red segmentada por capas con salida controlada
- [x] Diseño ajustado a Free Tier / AWS Academy