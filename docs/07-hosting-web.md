# Alojamiento Web Estático y CDN (S3 + CloudFront)

## Objetivo
Publicar el sitio corporativo/portal estático de ACME (landing page,
recursos de marketing, documentación pública) con buena performance
global, usando almacenamiento de objetos como origen y una CDN para
distribución.

## Por qué separado de la app en EC2
La aplicación transaccional (ventas/finanzas) vive en EC2 detrás del ALB
(Lección 4 y 5) porque es contenido dinámico. El **sitio web estático**
(landing, recursos públicos, portal informativo) no necesita cómputo
dedicado: se sirve de forma más económica y rápida desde S3 + CloudFront.

## Componentes utilizados
- **Amazon S3** (bucket con Static Website Hosting habilitado)
- **Amazon CloudFront** (CDN — Edge Locations globales)
- **AWS Certificate Manager (ACM)** (certificado SSL/TLS gratuito)
- **Origin Access Control (OAC)** (restringe acceso directo al bucket)

## Diseño propuesto

| Elemento | Configuración |
|---|---|
| Bucket | `acme-frontend-web` |
| Hosting estático | Habilitado, documento índice `index.html`, error `error.html` |
| Acceso directo al bucket | **Bloqueado** — solo CloudFront puede leer (vía OAC) |
| Distribución CloudFront | Origen: `acme-frontend-web`, cacheo de archivos estáticos (HTML, CSS, JS, imágenes) |
| Certificado SSL | Emitido por AWS Certificate Manager (ACM), asociado a la distribución |
| Dominio personalizado | Opcional — `www.acme-infraestructuraviva.com` vía Route 53 (CNAME/Alias hacia CloudFront) |

## Flujo de una petición
1. El usuario solicita `https://www.acme-infraestructuraviva.com`.
2. Route 53 resuelve el dominio hacia la distribución de **CloudFront**.
3. CloudFront revisa su caché en el Edge Location más cercano al usuario:
   - Si el contenido está en caché → lo entrega directamente (baja latencia).
   - Si no está en caché → lo solicita al bucket S3 de origen, lo cachea y
     lo entrega.
4. Toda la conexión viaja cifrada por HTTPS gracias al certificado de ACM.

## Configuraciones de seguridad (hasta 2, según pauta)
1. **SSL/TLS vía ACM:** certificado gratuito asociado a la distribución
   CloudFront, forzando HTTPS (redirect automático desde HTTP).
2. **Permisos de bucket restringidos (OAC):** el bucket S3 **no es público
   directamente**, su política solo permite lecturas desde la identidad de
   CloudFront (Origin Access Control), evitando que alguien acceda al
   contenido saltándose la CDN.

## Justificación
- **Costo:** servir contenido estático desde S3+CloudFront es
  significativamente más económico que mantener un servidor web dedicado
  encendido 24/7 solo para archivos estáticos.
- **Performance global:** los Edge Locations de CloudFront reducen la
  latencia para usuarios en distintas regiones, sin que ACME tenga que
  desplegar infraestructura en cada región.
- **Seguridad:** al bloquear el acceso directo al bucket, se elimina un
  vector de exposición accidental de archivos (mala configuración de
  permisos públicos en S3).

## Métricas cubiertas (según pauta de evaluación)
- ✅ 1 sitio web público en S3 (`acme-frontend-web`, hosting estático)
- ✅ 1 distribución CloudFront implementada
- ✅ Hasta 2 configuraciones de seguridad (SSL/TLS + permisos OAC)

## Referencia
Basado en Manual #7: Servicios simples de alojamiento web y contenidos
(CloudFront como CDN, comparativa con Lightsail, integración con Route 53).