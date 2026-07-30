# Infraestructura Viva
## Propuesta de Arquitectura Cloud para Soluciones Digitales ACME

**Proyecto ABP - Módulo 4: Arquitectura Cloud — SOFOFA**
**Autor:** Fidel Vera Chourio
**Fecha:** Julio 2026

---

## 1. Introducción

**Soluciones Digitales ACME** es una empresa que opera actualmente con
infraestructura on-premise para sus sistemas de ventas, finanzas y soporte.
Este documento resume la propuesta de migración hacia **Amazon Web
Services (AWS)**, diseñada para resolver seis problemáticas centrales de
la organización mediante ocho componentes de arquitectura, cada uno
respaldado por una decisión técnica documentada (ADR) y validado, en
varios casos, mediante un prototipo real desplegado en AWS Academy
Learner Lab.

El proyecto completo, documentación técnica, decisiones de arquitectura,
diagrama, prototipo y evidencia, está disponible en el repositorio:
**[Repositorio Infraestructura Viva](https://github.com/fevc08/infraestructura-viva)**

---

## 2. Problemática y objetivo

ACME enfrenta seis limitaciones producto de su infraestructura física:
cómputo que no escala ante demanda variable, datos transaccionales y
semiestructurados sin optimización diferenciada, almacenamiento sin
distinción de costo por frecuencia de uso, red sin segmentación ni
aislamiento de servicios críticos, ausencia de monitoreo proactivo, y
comunicación entre sistemas sin garantía de entrega.

**Objetivo del proyecto:** diseñar una arquitectura en AWS que resuelva
estas seis problemáticas de forma escalable, segura y económicamente
viable, aprovechando el Free Tier de AWS y AWS Academy Learner Lab.

---

## 3. Arquitectura propuesta — visión general

La arquitectura se organiza en una **VPC segmentada** (`10.0.0.0/16`) con
subredes pública y privadas distribuidas en dos zonas de disponibilidad,
separando la capa de entrada (ALB, CloudFront), la capa de aplicación
(EC2, Lambda, ECS/Fargate) y la capa de datos (RDS, DynamoDB).

![Diagrama de arquitectura general](../diagrams/arquitectura-general.png)

*(Diagrama completo disponible en `diagrams/arquitectura-general.png` del
repositorio; si este documento se presenta de forma aislada, se adjunta
la imagen junto con esta entrega.)*

**Recorrido de la arquitectura:**
1. Los usuarios acceden al sitio corporativo estático vía **Route 53 →
   CloudFront → S3**, y a la aplicación transaccional vía **Internet
   Gateway → ALB → Auto Scaling Group (EC2)**.
2. La app se conecta a **RDS Multi-AZ** para los datos de ventas/finanzas.
3. Los tickets de soporte fluyen por **API Gateway → Lambda → DynamoDB**,
   con notificación asíncrona vía **SQS → ECS/Fargate**.
4. **SNS** distribuye alertas a email, SMS y procesamiento automático.
5. **CloudWatch** monitoriza EC2, RDS y Lambda; sus alarmas notifican y
   disparan Auto Scaling; **EventBridge** correlaciona eventos para crear
   tickets automáticos ante incidentes de severidad alta.
6. Los buckets de archivado transicionan a **Glacier** según su ciclo de
   vida.

---

## 4. Resumen por dominio de arquitectura

| # | Dominio | Servicio principal | Resuelve |
|---|---|---|---|
| 1 | Almacenamiento | S3 + Glacier, lifecycle policies | Diferenciación de costo por frecuencia de acceso |
| 2 | Base de datos relacional | RDS MySQL, Multi-AZ | Datos transaccionales de ventas/finanzas, alta disponibilidad |
| 3 | Base de datos NoSQL | DynamoDB, GSI, On-Demand | Datos semiestructurados de soporte (tickets) |
| 4 | Cómputo | EC2 (ASG), Lambda, ECS/Fargate | Cómputo adaptado a 3 patrones de carga distintos |
| 5 | Red | VPC segmentada, ALB, Security Groups | Aislamiento de servicios críticos, punto único de entrada |
| 6 | Mensajería | SNS + SQS + DLQ | Comunicación desacoplada y con reintentos garantizados |
| 7 | Hosting web | S3 + CloudFront + ACM | Contenido estático performante y seguro |
| 8 | Monitoreo | CloudWatch + EventBridge | Detección proactiva y remediación automática de incidentes |

Cada dominio cuenta con documentación técnica completa
(`docs/0X-*.md`) y su decisión de arquitectura correspondiente
(`adr/000X-*.md`) en el repositorio.

---

## 5. Decisiones clave de arquitectura

- **Cómputo híbrido (ADR-0001):** se combinó EC2 (carga constante),
  Lambda (carga por eventos) y ECS/Fargate (microservicio), en vez de un
  único modelo, optimizando costo según el patrón de tráfico de cada
  módulo.
- **RDS sobre Aurora (ADR-0002):** se priorizó RDS MySQL estándar por
  ajustarse al Free Tier y al volumen inicial de ACME, dejando Aurora
  como alternativa de escalamiento futuro.
- **DynamoDB para datos semiestructurados (ADR-0003):** se evitó forzar
  los tickets de soporte a un esquema relacional rígido, aprovechando la
  flexibilidad y el escalado automático de DynamoDB.
- **Red segmentada en 3 capas (ADR-0005):** ningún servicio de datos es
  alcanzable directamente desde Internet; todo el tráfico pasa por un
  punto de entrada controlado (ALB o CloudFront).
- **SNS + SQS con Dead Letter Queue (ADR-0006):** se garantiza que
  ninguna notificación o ticket se pierda ante fallos temporales de un
  consumidor.
- **CloudWatch + EventBridge (ADR-0008):** se prioriza distinguir
  incidentes puntuales de incidentes sistémicos mediante correlación de
  alarmas, evitando tanto la sub-reacción como el ruido excesivo de
  notificaciones.

---

## 6. Prototipo desplegado

A diferencia de un ejercicio puramente teórico, parte de esta arquitectura
fue **desplegada y validada realmente** en AWS Academy Learner Lab:

- VPC completa (3 subredes, Internet Gateway, NAT Gateway, route tables)
  replicando el diseño de la Lección 5.
- Instancia RDS MySQL (`acme-db-demo`) operativa, con base de datos y
  esquema real (4 tablas: clientes, productos, ventas, facturas).
- Instancia EC2 en subred privada, conectada exitosamente a RDS,
  accedida mediante **AWS Systems Manager Session Manager** (sin SSH
  expuesto, sin llaves, sin puertos entrantes — alineado con el principio
  de seguridad por capas del diseño).
- Las 5 consultas SQL documentadas en la Lección 2 se ejecutaron
  exitosamente contra la base de datos real, con evidencia capturada.
- Recursos eliminados al finalizar la prueba para evitar consumo
  innecesario del Free Tier / créditos de AWS Academy.

Detalle completo y evidencia en `prototipo/README.md` y
`prototipo/consultas-sql/`.

---

## 7. Buenas prácticas aplicadas

| Categoría | Prácticas clave |
|---|---|
| **Seguridad** | Defensa en profundidad (Security Groups + NACL), roles IAM de menor privilegio, cifrado en tránsito y en reposo, sin exposición pública de bases de datos |
| **Escalabilidad** | Auto Scaling Group (EC2), capacidad On-Demand (DynamoDB, Lambda), escalado automático de CloudFront |
| **Alta disponibilidad** | Multi-AZ en RDS, Auto Scaling Group multi-AZ, replicación nativa en S3/DynamoDB/SNS/SQS |
| **Costo** | Instancias en Free Tier, capacidad On-Demand, transición automática a almacenamiento frío |

Detalle completo en `docs/buenas-practicas.md`.

---

## 8. Estimación de costos

| Escenario | Estimado mensual |
|---|---|
| Durante Free Tier (primeros 12 meses) | ~$50-55 USD/mes (principalmente NAT Gateway y ALB, no cubiertos por Free Tier) |
| Post-Free Tier | ~$90-110 USD/mes según volumen real |

El detalle línea por línea, con precios de referencia de AWS (us-east-1,
julio 2026), está en `docs/costos-estimados.md`.

---

## 9. Conclusiones

La arquitectura propuesta resuelve las seis problemáticas de ACME
combinando servicios gestionados que reducen la carga operativa del
equipo de Innovación Tecnológica, manteniendo el costo dentro de rangos
predecibles y ajustados al Free Tier durante la fase inicial. El
prototipo desplegado confirma que el diseño de red y la conexión
cómputo-base de datos son **viables y funcionales en la práctica**, no
solo en el papel.

**Próximos pasos recomendados para ACME:**
1. Evaluar Read Replicas en RDS si crece la carga de reportería.
2. Sustituir el NAT Gateway por VPC Gateway Endpoints (gratuitos) para
   tráfico hacia S3/DynamoDB, reduciendo el costo fijo mensual.
3. Evaluar migración a Aurora o capacidad reservada una vez que el
   volumen de uso de ACME sea predecible.
4. Completar el retiro gradual de la infraestructura on-premise una vez
   validada la migración en producción.

---

## 10. Enlaces del proyecto

- **Repositorio completo:** `https://github.com/fevc08/infraestructura-viva`
- **Diagrama de arquitectura:** `diagrams/arquitectura-general.drawio` / `.png`
- **Documentación técnica:** carpeta `docs/`
- **Decisiones de arquitectura (ADR):** carpeta `adr/`
- **Prototipo y evidencia:** carpeta `prototipo/`