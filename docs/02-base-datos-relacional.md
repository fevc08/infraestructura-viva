# Base de Datos Relacional (Amazon RDS)

## Objetivo
Desplegar una base de datos relacional gestionada para los datos
transaccionales de los departamentos de ventas y finanzas, con alta
disponibilidad y backups automáticos, validando consultas antes de migrar
mediante SQLiteOnline.

## Componentes utilizados
- **Amazon RDS** (motor MySQL, instancia `db.t3.micro` — free tier eligible)
- **Multi-AZ Deployment** (alta disponibilidad)
- **Backups automáticos** (retención configurable, snapshot diario)
- **Security Group** dedicado (acceso solo desde la subred privada de la app)
- **SQLiteOnline** (prototipado y validación de consultas antes del despliegue)

## Diseño propuesto

| Elemento | Configuración |
|---|---|
| Motor | MySQL 8.0 |
| Clase de instancia | db.t3.micro (Free Tier) |
| Almacenamiento | 20 GB GP2/GP3 |
| Multi-AZ | Habilitado (réplica sincrónica en otra zona de disponibilidad) |
| Backups automáticos | Habilitados, ventana diaria, retención 7 días |
| Acceso | Privado — solo desde subred de aplicación vía Security Group |
| Read Replica | Opcional a futuro, si crece la carga de lectura (reportería finanzas) |

## Modelo de datos (ejemplo simplificado)
Tablas propuestas para el dominio de ventas/finanzas:
- `clientes` (id, nombre, email, fecha_registro)
- `productos` (id, nombre, precio, stock)
- `ventas` (id, cliente_id, producto_id, cantidad, fecha, monto_total)
- `facturas` (id, venta_id, estado_pago, fecha_emision)

## Consultas SQL validadas en SQLiteOnline
Antes de desplegar en RDS, se prototipa el esquema y se prueban consultas en
SQLiteOnline (sin necesidad de instalación local):

1. `SELECT * FROM ventas WHERE fecha BETWEEN '2026-01-01' AND '2026-06-30';`
2. `SELECT c.nombre, SUM(v.monto_total) AS total_comprado FROM ventas v JOIN clientes c ON v.cliente_id = c.id GROUP BY c.nombre ORDER BY total_comprado DESC;`
3. `SELECT p.nombre, p.stock FROM productos p WHERE p.stock < 10;`
4. `UPDATE productos SET stock = stock - 1 WHERE id = 1;`
5. `SELECT f.estado_pago, COUNT(*) FROM facturas f GROUP BY f.estado_pago;`

> Evidencia: capturas de pantalla de estas 5 consultas ejecutándose en
> SQLiteOnline se guardan en `prototipo/consultas-sql/`.

## Alta disponibilidad y backups
- **Multi-AZ:** RDS mantiene una réplica sincrónica en otra zona de
  disponibilidad; ante una falla, AWS promueve automáticamente la réplica
  como instancia primaria (failover automático).
- **Backups automáticos:** snapshots diarios + transaction logs, permitiendo
  restauración point-in-time dentro de la ventana de retención.
- **Mantenimiento gestionado:** parches y actualizaciones aplicados por AWS,
  eliminando la carga operativa que hoy tiene el equipo on-premise.

## Justificación
- **Por qué RDS y no una VM con MySQL manual:** delega instalación, parches,
  backups y monitoreo a AWS, liberando al equipo de Innovación Tecnológica
  de tareas operativas repetitivas.
- **Por qué MySQL:** motor ampliamente soportado, buena documentación,
  compatible con las herramientas gratuitas usadas en el proyecto
  (SQLiteOnline para prototipado, VS Code para scripts).
- **Costo:** clase `db.t3.micro` se ajusta al Free Tier de AWS/AWS Academy,
  ideal para una prueba de concepto antes de escalar verticalmente si el
  volumen de ventas crece.

## Escalabilidad futura
- **Vertical:** cambiar de clase de instancia (ej. a `db.t3.small`) si crece
  la carga transaccional.
- **Horizontal:** agregar Read Replicas para reportes de finanzas de fin de
  mes, sin afectar el rendimiento de las transacciones de ventas.

## Métricas cubiertas (según pauta de evaluación)
- ✅ 1 instancia RDS con backups configurados
- ✅ 3–5 consultas SQL probadas en SQLiteOnline (5 incluidas arriba)
- ✅ 1 documentación de configuración RDS (este documento)

## Referencia
Basado en Manual #2: Servicios de bases de datos relacionales (Partes 1 y 2):
motores disponibles, escalabilidad vertical/horizontal, Multi-AZ y modelo de
costos de RDS.