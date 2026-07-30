# Infraestructura Viva: Propuesta de Arquitectura Cloud para ACME

> Proyecto ABP - Módulo 4: Arquitectura Cloud - SOFOFA

## 📋 Descripción del proyecto

**Soluciones Digitales ACME** es una empresa que actualmente opera con
infraestructura on-premise y necesita migrar sus sistemas críticos
(ventas, finanzas y soporte) hacia la nube de AWS. Este repositorio
documenta la **propuesta de arquitectura cloud** diseñada para resolver
seis problemáticas centrales de la organización: cómputo escalable,
integración de bases de datos relacionales y NoSQL, almacenamiento
diferenciado, red segmentada y segura, mensajería/notificaciones
confiables, y monitoreo con respuesta automática ante incidentes.

Este es un proyecto **de diseño y documentación de arquitectura**, no
contiene código de aplicación ni despliegue real. Cada carpeta refleja la
estructura que tendría el proyecto si se implementara, y cada decisión
técnica relevante está justificada mediante un **ADR (Architecture
Decision Record)**.

## 🎯 Objetivo general

Diseñar una arquitectura en AWS que permita a ACME operar de forma
escalable, segura y económicamente eficiente (aprovechando Free Tier /
AWS Academy), reemplazando su infraestructura física por servicios
gestionados en la nube.

## 🗂️ Estructura del repositorio

```
infraestructura-viva/
├── README.md                    ← estás aquí
├── docs/                        ← documentación técnica por componente
├── adr/                         ← decisiones de arquitectura (ADR)
├── diagrams/                    ← diagrama de arquitectura (draw.io + imagen)
├── prototipo/                   ← instrucciones de despliegue / demo
├── monitoreo/                   ← plan de monitoreo y evidencias
├── portafolio/                  ← reflexión personal del proyecto
└── presentacion/                ← resumen ejecutivo / presentación final
```

## 📚 Índice de lecciones (documento + decisión de arquitectura)

| # | Componente | Documento | ADR asociado |
|---|---|---|---|
| 1 | Almacenamiento en cloud | [`docs/01-almacenamiento.md`](docs/01-almacenamiento.md) | [`adr/0004-estrategia-almacenamiento.md`](adr/0004-estrategia-almacenamiento.md) |
| 2 | Base de datos relacional | [`docs/02-base-datos-relacional.md`](docs/02-base-datos-relacional.md) | [`adr/0002-eleccion-bd-relacional.md`](adr/0002-eleccion-bd-relacional.md) |
| 3 | Base de datos NoSQL | [`docs/03-base-datos-nosql.md`](docs/03-base-datos-nosql.md) | [`adr/0003-eleccion-bd-nosql.md`](adr/0003-eleccion-bd-nosql.md) |
| 4 | Servicios de cómputo | [`docs/04-computo.md`](docs/04-computo.md) | [`adr/0001-eleccion-computo.md`](adr/0001-eleccion-computo.md) |
| 5 | Red en la nube (VPC) | [`docs/05-red.md`](docs/05-red.md) | [`adr/0005-diseno-red-vpc.md`](adr/0005-diseno-red-vpc.md) |
| 6 | Notificación y mensajería | [`docs/06-mensajeria-notificaciones.md`](docs/06-mensajeria-notificaciones.md) | [`adr/0006-mensajeria-notificaciones.md`](adr/0006-mensajeria-notificaciones.md) |
| 7 | Alojamiento web y CDN | [`docs/07-hosting-web.md`](docs/07-hosting-web.md) | [`adr/0007-hosting-cdn.md`](adr/0007-hosting-cdn.md) |
| 8 | Monitoreo y correlación de incidentes | [`docs/08-monitoreo-incidentes.md`](docs/08-monitoreo-incidentes.md) | [`adr/0008-monitoreo-alarmas.md`](adr/0008-monitoreo-alarmas.md) |

Todos los ADR siguen la plantilla base: [`adr/template.md`](adr/template.md).

## 🖼️ Diagrama de arquitectura

El diagrama general (formato editable `.drawio` y exportado `.png`) se
encuentra en [`diagrams/`](diagrams/) y está incrustado en el documento
consolidado [`docs/00-arquitectura-general.md`](docs/00-arquitectura-general.md).

## 🛠️ Servicios AWS utilizados

| Categoría | Servicios |
|---|---|
| Cómputo | EC2, Lambda, ECS/Fargate |
| Bases de datos | RDS (MySQL), DynamoDB |
| Almacenamiento | S3, S3 Glacier / Glacier Deep Archive |
| Red | VPC, Subredes públicas/privadas, ALB, NAT Gateway, Security Groups, NACL, VPN Site-to-Site |
| Mensajería | SNS, SQS |
| Hosting / CDN | S3 (static hosting), CloudFront, ACM, Route 53 |
| Monitoreo | CloudWatch (Metrics, Logs, Alarms), EventBridge |

## 🧭 Cómo navegar este repositorio

1. Empieza por [`docs/00-arquitectura-general.md`](docs/00-arquitectura-general.md)
   para la visión completa con el diagrama.
2. Profundiza en cada componente desde la tabla de índice de lecciones
   más arriba.
3. Revisa `adr/` para entender **por qué** se tomó cada decisión, no solo
   **qué** se decidió.
4. Consulta `docs/buenas-practicas.md` para el resumen transversal de
   seguridad, escalabilidad y alta disponibilidad.
5. `monitoreo/`, `prototipo/` y `presentacion/` contienen los entregables
   operativos y de cierre del proyecto.

## ✅ Estado del proyecto

- [x] Estructura del repositorio
- [x] Lección 1 — Almacenamiento
- [x] Lección 2 — Base de datos relacional
- [x] Lección 3 — Base de datos NoSQL
- [x] Lección 4 — Cómputo
- [x] Lección 5 — Red
- [x] Lección 6 — Mensajería y notificaciones
- [x] Lección 7 — Alojamiento web y CDN
- [x] Lección 8 — Monitoreo y correlación de incidentes
- [x] Diagrama de arquitectura general
- [x] Documento consolidado de arquitectura general
- [x] Guía de buenas prácticas
- [x] Plan de monitoreo consolidado
- [x] Estimación de costos
- [x] Prototipo / demo (desplegado y validado en AWS Academy Learner Lab)
- [x] Presentación final
- [x] Reflexión de portafolio

**Proyecto completo. ✔**

## 👤 Autor
Fidel Enrique Vera Chourio - Proyecto ABP, Módulo 4: Arquitectura Cloud - SOFOFA