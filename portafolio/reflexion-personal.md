# Reflexión Personal — Proyecto Infraestructura Viva

## Contexto
Este proyecto consistió en diseñar una propuesta completa de migración
cloud para una empresa ficticia (ACME), documentando 8 dominios de
arquitectura AWS, y validando parte del diseño mediante un prototipo real
desplegado en AWS Academy Learner Lab.

## Aprendizajes técnicos

Durante el desarrollo apliqué conceptos de los 8 manuales del módulo de
forma práctica, no solo teórica:

- **Almacenamiento:** entender cuándo usar S3 Standard vs. Glacier no es
  solo una decisión de "barato vs. caro", sino de patrón de acceso real
  a los datos.
- **Bases de datos:** experimenté en carne propia por qué RDS y DynamoDB
  no son intercambiables — modelé el mismo proyecto con datos
  transaccionales estructurados (ventas) y datos semiestructurados
  (tickets de soporte) y las diferencias de diseño fueron evidentes.
- **Red:** diseñar una VPC segmentada en papel es una cosa; verla
  funcionar de verdad (con el tráfico realmente pasando por el NAT
  Gateway, confirmado con un simple `curl checkip.amazonaws.com`) fue
  donde el concepto realmente "hizo clic".
- **Monitoreo:** correlacionar alarmas con EventBridge me hizo pensar en
  el monitoreo no como "alarmas sueltas" sino como un sistema de
  detección con distintos niveles de severidad.

## Desafíos enfrentados (y cómo los resolví)

- **Restricciones reales de AWS Academy:** no pude crear un rol IAM
  personalizado (`iam:CreateRole` denegado) — tuve que adaptar el diseño
  para usar `LabRole`, entendiendo que en un entorno educativo las
  cuentas tienen permisos intencionalmente limitados, distinto a una
  cuenta de producción real.
- **Convenciones de nombres de AWS:** un error simple («`sg-` no puede
  usarse como prefijo de nombre») me obligó a ajustar toda la
  documentación para mantenerla consistente con lo realmente desplegado.
- **Errores de conexión:** un `Access denied` al conectar a RDS resultó
  ser un problema de tipeo de contraseña, no de red, aprendí a
  diagnosticar primero si el error es de "capa de red" (timeout) o de
  "capa de aplicación" (credenciales), que son dos problemas muy
  distintos con síntomas parecidos.
- **Iterar el diagrama:** el diagrama de arquitectura pasó por varias
  rondas de revisión antes de quedar consistente con la documentación,
  entendí que un diagrama "bonito" y un diagrama "correcto" no siempre
  son lo mismo en el primer intento.

## Transición on-premise → nube: costos y seguridad

Antes de este proyecto, asumía que "usar el Free Tier" significaba costo
prácticamente cero. Me sorprendió descubrir que componentes como el NAT
Gateway y el Load Balancer nunca están cubiertos por ningún nivel
gratuito, y que justamente esos son los costos fijos más relevantes de
toda la arquitectura. Eso me hizo entender que "optimizar costos en la
nube" no es evitar todo gasto, sino gastar de forma consciente en lo que
realmente aporta seguridad y disponibilidad. En cuanto a seguridad, pensar
en capas (Security Groups + NACL + roles IAM de menor privilegio) es un
cambio de mentalidad respecto a un firewall único on-premise, cada capa
asume que la anterior podría fallar.

## Herramientas utilizadas
Visual Studio Code (con extensión Remote-SSH y AWS Toolkit), SQLiteOnline
para prototipado de consultas, AWS Academy Learner Lab, GitHub para
control de versiones y documentación, y draw.io para el diagrama de
arquitectura.

## Reflexión final

La parte que más disfruté fue, sin duda, ver el prototipo funcionando de
verdad: conectarme por Session Manager, llegar hasta la base de datos RDS
y ejecutar las mismas consultas que antes solo había probado en
SQLiteOnline, cerró el círculo entre la teoría y la práctica de una forma
que la documentación por sí sola no logra. Lo que más trabajo me costó
fue, paradójicamente, algo que no esperaba: iterar el diagrama de
arquitectura hasta que cada flecha representara correctamente la
dirección real del flujo de datos, es fácil que un diagrama se vea bien
y esté conceptualmente invertido.

Si tuviera que rehacer el proyecto desde cero, dedicaría tiempo al inicio
a revisar las restricciones específicas de AWS Academy (como los permisos
limitados de IAM) y las convenciones de nombres de AWS, en vez de
descubrirlas sobre la marcha, habría ahorrado algunas vueltas.

Este proyecto reforzó mi interés en especializarme en
arquitectura cloud/DevOps

A alguien que está por empezar este módulo le diría que no dé por sentado
que el diseño teórico se traduce 1 a 1 al desplegarlo de verdad, casi
siempre aparece algún ajuste de nombres, permisos o configuración que solo
se descubre al hacerlo, y eso no es un error, es parte del aprendizaje.

---

*Este proyecto forma parte de mi portafolio profesional. El repositorio
completo, incluyendo diagrama, documentación técnica, decisiones de
arquitectura (ADR) y evidencia del prototipo desplegado, está disponible
en: https://github.com/fevc08/infraestructura-viva*