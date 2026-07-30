# Red en la Nube (Amazon VPC)

## Objetivo
Diseñar una red virtual privada que aísle los servicios internos (app,
base de datos) de los servicios expuestos públicamente, con reglas de
seguridad explícitas y balanceo de carga para alta disponibilidad.

## Componentes utilizados
- **Amazon VPC** (10.0.0.0/16)
- **Subredes públicas y privadas** distribuidas en 2 zonas de disponibilidad
- **Internet Gateway** (acceso público) y **NAT Gateway** (salida controlada
  desde subredes privadas)
- **Security Groups** (firewall a nivel de instancia) y **NACLs** (firewall a
  nivel de subred)
- **Application Load Balancer (ALB)**
- **AWS Site-to-Site VPN** (opcional, fase de transición on-premise → nube)

## Diseño de la VPC

| Subred | CIDR | Zona de disponibilidad | Tipo | Contenido |
|---|---|---|---|---|
| `subnet-publica-a` | 10.0.1.0/24 | AZ-A | Pública | Application Load Balancer, NAT Gateway |
| `subnet-privada-app-a` | 10.0.2.0/24 | AZ-A | Privada | Instancias EC2 (app ventas/finanzas), Auto Scaling Group |
| `subnet-privada-datos-b` | 10.0.3.0/24 | AZ-B | Privada | Instancia RDS (Multi-AZ) |

> Se usan 2 zonas de disponibilidad para preparar la arquitectura de cara a
> alta disponibilidad real (aunque la métrica mínima solicitada es 2 subredes
> en 1 VPC, se documentan 3 para reflejar la segmentación por capas:
> pública / aplicación / datos).

## Reglas de seguridad (hasta 4, según pauta)

| # | Regla | Tipo | Origen → Destino | Puerto | Propósito |
|---|---|---|---|---|---|
| 1 | `sg-alb` | Security Group | Internet (0.0.0.0/0) → ALB | 443 (HTTPS), 80 (HTTP→redirect) | Único punto de entrada público |
| 2 | `acme-sg-app` | Security Group | `sg-alb` → EC2 (app) | 8080 | Solo el ALB puede hablar con la app, nunca Internet directo |
| 3 | `acme-sg-rds` | Security Group | `acme-sg-app` → RDS | 3306 (MySQL) | Solo la app puede consultar la base de datos |
| 4 | `nacl-datos` | Network ACL | Subred de datos | Deny all excepto subred de app | Segunda capa de defensa a nivel de subred (stateless) |

> **Nota de implementación:** los nombres `sg-app` y `sg-rds` no son válidos en
> AWS (el prefijo `sg-` está reservado para IDs autogenerados). En el
> despliegue real del prototipo se usaron `acme-sg-app` y `acme-sg-rds`.

## Balanceador de carga
- **Application Load Balancer (ALB)** desplegado en la subred pública.
- Distribuye tráfico HTTP/HTTPS hacia las instancias EC2 del Auto Scaling
  Group (subred privada de app).
- Health checks sobre `/health` cada 30 segundos; instancias no saludables
  se retiran automáticamente del target group.
- Termina el certificado SSL (ver Lección 7 para el dominio/certificado).

## Aislamiento de servicios internos vs. exposición pública
- Solo la subred pública tiene ruta hacia el **Internet Gateway**.
- Las subredes privadas usan el **NAT Gateway** únicamente para tráfico
  saliente (actualizaciones de paquetes, llamadas a APIs externas),
  nunca para tráfico entrante.
- La subred de datos no tiene ruta de salida a Internet en absoluto —
  solo es alcanzable desde la subred de aplicación.

## VPN Site-to-Site (opcional, fase de transición)
Mientras dure la migración, se puede establecer un túnel **IPSec Site-to-Site
VPN** entre la red on-premise de ACME y la VPC, permitiendo que sistemas
legados aún no migrados se comuniquen con los nuevos servicios en la nube
de forma cifrada, sin exponerlos a Internet pública.

## Justificación
- **Segmentación por capas:** limita el "blast radius" ante un compromiso de
  seguridad — si la app se ve comprometida, el atacante no llega
  directamente a la base de datos sin pasar por el `sg-rds`.
- **Defensa en profundidad:** Security Groups (con estado, a nivel de
  instancia) + NACLs (sin estado, a nivel de subred) juntos, no uno solo.
- **Costo:** NAT Gateway y ALB tienen costo por hora; se justifica frente al
  riesgo de exponer instancias directamente a Internet.

## Métricas cubiertas (según pauta de evaluación)
- ✅ 1 VPC con 2 subredes (mínimo) → se documentan 3 (pública, app, datos)
- ✅ Hasta 4 reglas de seguridad configuradas (tabla arriba)
- ✅ 1 balanceador de carga activo (ALB)

## Validación práctica (prototipo)
Este diseño de red fue desplegado y validado realmente en AWS Academy
Learner Lab como parte del prototipo del proyecto (ver
[`prototipo/README.md`](../prototipo/README.md)):
- VPC, 3 subredes, Internet Gateway, NAT Gateway y route tables — creados y
  funcionales.
- Conectividad EC2 (subred privada) → RDS (subred privada de datos) vía
  `acme-sg-rds` — verificada con una conexión MySQL exitosa.
- Salida a Internet desde subred privada vía NAT Gateway — verificada con
  `curl https://checkip.amazonaws.com` desde la instancia.

## Referencia
Basado en Manual #5: Servicios de red en la nube (VPC, subredes, NACL,
Security Groups, VPN Site-to-Site, Direct Connect, Route 53).