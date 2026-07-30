# Servicios de Cómputo

## Objetivo
Desplegar las aplicaciones de ACME (ventas/finanzas y soporte) usando
recursos de cómputo escalables, combinando máquinas virtuales, serverless y
contenedores según el tipo de carga de trabajo, aprovechando Free Tier /
AWS Academy.

## Componentes utilizados (hasta 3 servicios de cómputo)

| Servicio | Uso en la arquitectura | Tipo de carga |
|---|---|---|
| **Amazon EC2** (t3.micro, Free Tier) | Aplicación web principal de ventas/finanzas, conectada a RDS | Carga constante, 24/7 |
| **AWS Lambda** | Procesamiento de tickets de soporte (API Gateway → Lambda → DynamoDB, ver Lección 3) | Carga por eventos, intermitente |
| **Amazon ECS con Fargate** | Microservicio de notificaciones internas (consume mensajes de SQS, ver Lección 6) | Microservicio, escalado por demanda |

## Por qué esta combinación (no un solo modelo)
- **EC2** se eligió para la app principal porque es una migración directa
  desde el entorno on-premise actual (monolito de ventas/finanzas),
  minimizando el esfuerzo de re-arquitectura inicial.
- **Lambda** se eligió para el módulo de soporte porque su tráfico es
  esporádico (picos ante incidentes), y el modelo "pago por ejecución" es
  más económico que mantener una instancia encendida 24/7 para eso.
- **ECS/Fargate** se eligió para el microservicio de notificaciones porque
  se beneficia de despliegues ágiles tipo contenedor sin gestionar
  servidores subyacentes (a diferencia de EC2 puro).

## Configuración EC2 (aplicación principal)
| Parámetro | Valor |
|---|---|
| Tipo de instancia | t3.micro (Free Tier) |
| AMI | Amazon Linux 2023 |
| Subred | Privada (solo accesible vía balanceador de carga, ver Lección 5) |
| Auto Scaling Group | Mínimo 1, deseado 2, máximo 4 instancias |
| Política de escalado | Añadir instancia si CPU > 80% durante 10 min (CloudWatch Alarm, ver Lección 8) |

## Prueba de escalabilidad documentada
Escenario de prueba:
1. Se genera carga sintética sobre el endpoint de la app (ej. con Apache
   Bench o un script de peticiones concurrentes) durante 15 minutos.
2. CloudWatch registra el uso de CPU de la instancia EC2.
3. Al superar el umbral de 80% de CPU sostenido por 10 minutos, el Auto
   Scaling Group lanza una segunda instancia automáticamente.
4. El balanceador de carga redistribuye el tráfico entre ambas instancias,
   normalizando el uso de CPU.
5. Al bajar la carga, el Auto Scaling Group reduce nuevamente a la cantidad
   mínima de instancias.

> Evidencia: capturas del gráfico de CloudWatch (CPU antes/durante/después
> de la prueba) y del evento de scale-out se documentan en
> `monitoreo/evidencias/`.

## Justificación de costos
- EC2 t3.micro y Lambda están dentro de los límites del Free Tier de AWS
  para cargas de prueba/portafolio.
- Fargate cobra por vCPU/memoria reservada por segundo — se usa en un
  microservicio liviano y de bajo uso para mantener el costo controlado.

## Métricas cubiertas (según pauta de evaluación)
- ✅ 1 instancia o función serverless desplegada (mínimo) → EC2 + Lambda
- ✅ Hasta 3 servicios de cómputo utilizados (máximo) → EC2, Lambda, ECS/Fargate
- ✅ 1 prueba de escalabilidad documentada (Auto Scaling Group + CloudWatch)

## Referencia
Basado en Manual #4: Servicios de cómputo (comparativa EC2 vs.
contenedores vs. serverless, y estructura de costos por tipo de servicio).