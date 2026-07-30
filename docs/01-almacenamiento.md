# Almacenamiento en la Nube

## Objetivo
Implementar una estrategia de almacenamiento en AWS que diferencie datos de
uso frecuente de datos de archivado, optimizando costo, durabilidad y
disponibilidad, sin depender de hardware físico on-premise.

## Componentes utilizados
- **Amazon S3** (almacenamiento de objetos, clase Standard)
- **Amazon S3 Glacier / Glacier Deep Archive** (archivado de largo plazo)
- **S3 Lifecycle Policies** (transición automática entre clases)
- **S3 Versioning** (protección ante sobrescritura/eliminación accidental)

## Diseño propuesto

| Bucket | Propósito | Clase inicial | Política de ciclo de vida |
|---|---|---|---|
| `acme-static-content` | Archivos estáticos y multimedia de las apps de ventas/soporte | S3 Standard | Sin transición (acceso frecuente) |
| `acme-backups-datos` | Respaldos de bases de datos y documentos históricos | S3 Standard | Transición a S3 Glacier a los 30 días |
| `acme-logs-monitoreo` | Logs de aplicación y auditoría | S3 Standard-IA | Transición a Glacier Deep Archive a los 90 días |

## Origen de los datos por bucket

| Bucket | Origen de los datos | Mecanismo |
|---|---|---|
| `acme-static-content` | Equipo de desarrollo / apps de ventas y soporte | Carga manual o vía pipeline de despliegue |
| `acme-backups-datos` | **Amazon RDS** (base de datos relacional, Lección 2) | Snapshot automático diario / export programado hacia el bucket |
| `acme-logs-monitoreo` | **Amazon CloudWatch Logs** (Lección 8) | Exportación periódica de logs de aplicación (EC2, Lambda, ECS) y, opcionalmente, VPC Flow Logs de la red (Lección 5) |

Esto cierra el ciclo de vida completo de cada bucket de archivado: los datos
no llegan "de la nada", sino que provienen de un servicio productor
identificado, se almacenan en S3 Standard mientras son recientes, y luego
transicionan automáticamente a Glacier según la política de ciclo de vida
definida más abajo.

## Política de ciclo de vida (lifecycle)
- Día 0–30: los objetos de `acme-backups-datos` permanecen en S3 Standard
  (acceso rápido ante incidentes recientes).
- Día 30+: transición automática a **S3 Glacier** (recuperación en minutos/horas,
  costo significativamente menor).
- Día 90+ (logs): transición a **Glacier Deep Archive** (más económico,
  recuperación en horas, apto para auditoría/cumplimiento).

## Estrategia de backup
- **Versioning** habilitado en los 3 buckets (permite recuperar versiones
  anteriores ante borrado o sobrescritura accidental).
- Backup programado (manual o script) que sube exports de la base de datos
  relacional/NoSQL hacia `acme-backups-datos` de forma periódica.
- Retención definida según política interna (ej. 12 meses en Glacier antes de
  expiración definitiva).

## Justificación (uso, costo, disponibilidad)
- **Uso frecuente → S3 Standard:** los archivos estáticos de las apps se
  consultan constantemente, por lo que requieren baja latencia.
- **Uso esporádico/archivado → Glacier:** los respaldos y logs antiguos rara
  vez se consultan, por lo que priorizar costo sobre velocidad de acceso es
  la decisión correcta.
- **Durabilidad:** S3 y Glacier ofrecen 99.999999999% (11 nueves) de
  durabilidad, superando ampliamente lo que ACME podía garantizar on-premise.
- **Sin hardware físico:** se elimina la necesidad de mantener storage propio,
  reduciendo CAPEX a OPEX.

## Métricas cubiertas (según pauta de evaluación)
- ✅ 1 bucket estándar + 1 archivado (mínimo) → `acme-static-content` +
  `acme-backups-datos`
- ✅ Hasta 3 buckets con políticas diferentes (máximo) → se agregó
  `acme-logs-monitoreo`
- ✅ 1 política de backup implementada → versioning + lifecycle rules

## Referencia
Basado en Manual #1: Almacenamiento en cloud (secciones de tipos de
almacenamiento, estructura de costos y estrategias de uso de tecnologías).