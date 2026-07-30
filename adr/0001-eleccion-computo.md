# ADR-0001: Estrategia de cómputo híbrida (EC2 + Lambda + ECS/Fargate)

## Estado
Aceptado

## Fecha
2026-07-26

## Contexto
ACME necesita migrar una aplicación monolítica on-premise (ventas/finanzas)
y, además, soportar nuevos módulos con patrones de carga distintos
(soporte por eventos, notificaciones). No existe una única opción de
cómputo que sea óptima para los tres escenarios simultáneamente.

## Decisión
Se adoptará una estrategia de cómputo híbrida:
- **Amazon EC2** para la aplicación principal (carga constante).
- **AWS Lambda** para el procesamiento de tickets de soporte (carga por eventos).
- **Amazon ECS/Fargate** para un microservicio de notificaciones (carga variable, orientado a contenedores).

## Alternativas consideradas
| Opción | Pros | Contras | ¿Por qué no se eligió (como única opción)? |
|---|---|---|---|
| Todo en EC2 | Máximo control, migración directa | Se paga 24/7 incluso para cargas esporádicas (soporte, notificaciones); mayor esfuerzo operativo | Ineficiente en costos para módulos de tráfico intermitente |
| Todo serverless (Lambda) | Escalado automático total, sin servidores que administrar | La app monolítica de ventas/finanzas no está diseñada para un modelo de funciones; requeriría reescritura significativa | Alto costo de refactorización a corto plazo, fuera del alcance de esta primera migración |
| Todo en contenedores (ECS/EKS) | Portabilidad y estandarización total | Mayor curva de aprendizaje y complejidad de orquestación para un proyecto inicial con equipo pequeño | Sobrediseño para el alcance actual del proyecto |

## Consecuencias
- **Positivas:** cada carga de trabajo usa el modelo de cómputo más
  adecuado a su patrón de uso, optimizando costo y complejidad operativa.
- **Negativas / trade-offs:** mayor diversidad tecnológica implica más
  superficies a monitorear y asegurar (se aborda en Lección 8 – Monitoreo).
- **Impacto en costos:** favorable — se evita pagar cómputo constante para
  cargas esporádicas, y se aprovechan los niveles gratuitos de EC2 y Lambda.
- **Impacto en escalabilidad:** EC2 escala mediante Auto Scaling Group;
  Lambda escala automáticamente por diseño; ECS/Fargate escala por tareas
  según demanda de la cola SQS (ver ADR-0006, mensajería).