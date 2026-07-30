# Prototipo / Demo: Instancia de Cómputo Conectada a Base de Datos Gestionada

## Objetivo
Demostrar, mediante instrucciones reproducibles, cómo desplegar una
instancia de cómputo (EC2) conectada a una base de datos gestionada
(RDS), incluyendo la configuración del entorno de desarrollo en Visual
Studio Code y la validación de consultas SQL en SQLiteOnline antes de
migrar a producción.

> **Nota de alcance:** este documento es una guía operativa de despliegue,
> no un pipeline de código automatizado. Sigue estos pasos sobre una
> cuenta de **AWS Academy Learner Lab** o **Free Tier** para reproducir el
> prototipo.

---

## Prerrequisitos
- Cuenta de AWS Academy Learner Lab (o AWS Free Tier) activa.
- Visual Studio Code instalado.
- Extensión **AWS Toolkit** para VS Code.
- Extensión **Remote - SSH** para VS Code.
- Cliente MySQL (`mysql` CLI) o extensión **SQLTools** en VS Code.
- Un par de llaves SSH (`.pem`) generado desde la consola de AWS.

---

## Parte A — Desplegar RDS (base de datos gestionada)

1. Consola AWS → **RDS → Create database**.
2. Método de creación: **Standard create**.
3. Motor: **MySQL** (versión 8.0).
4. Plantilla: **Free tier**.
5. Identificador de instancia: `acme-db-demo`.
6. Usuario maestro: `admin` / contraseña autogenerada o definida.
7. Clase de instancia: `db.t3.micro`.
8. Conectividad: seleccionar la **VPC** y la **subred privada de datos**
   diseñadas en la [Lección 5](../docs/05-red.md).
9. Acceso público: **No**.
10. Grupo de seguridad: usar `sg-rds` (solo permite tráfico desde `sg-app`,
    ver [ADR-0005](../adr/0005-diseno-red-vpc.md)).
11. Crear la base de datos y esperar a que el estado sea `Available`.

## Parte B — Desplegar EC2 (instancia de cómputo)

1. Consola AWS → **EC2 → Launch instance**.
2. Nombre: `acme-app-demo`.
3. AMI: **Amazon Linux 2023** (Free tier eligible).
4. Tipo de instancia: `t3.micro`.
5. Par de claves: selecciona o crea uno nuevo (descarga el `.pem`).
6. Red: la **VPC** del proyecto, subred pública (para esta demo puntual;
   en producción la app corre en subred privada detrás del ALB, ver
   [Lección 4](../docs/04-computo.md)).
7. Grupo de seguridad: permitir **SSH (22)** solo desde tu IP, y salida
   libre para conectarse a RDS.
8. Lanzar la instancia.

## Parte C — Conexión a la instancia (implementación real)

> **Nota:** la guía original contemplaba conexión SSH + VS Code Remote-SSH.
> En la implementación real se usó **AWS Systems Manager Session Manager**
> en su lugar, por dos razones: (1) mayor seguridad — no requiere abrir el
> puerto 22 ni gestionar llaves `.pem`, y (2) AWS Academy Learner Lab
> restringe la creación de roles IAM personalizados (`iam:CreateRole`), por
> lo que se usó el rol preconfigurado **`LabRole`** en vez de un rol
> dedicado `acme-ec2-ssm-role`.

### Conexión real utilizada
1. EC2 → Instances → seleccionar instancia → **Connect**.
2. Pestaña **"Maximum security" → SSM Session Manager → Connect**.
3. Esto abre una terminal en el navegador autenticada vía IAM
   (`LabRole`), sin SSH, sin llaves, sin puertos entrantes.
4. Dentro de la terminal: `sudo dnf install -y mariadb105` para instalar
   el cliente MySQL, y `mysql -h <endpoint-rds> -u admin -p` para conectar
   a la base de datos.

### Validación de conectividad realizada
```bash
curl -s https://checkip.amazonaws.com
```
Confirmó que la instancia (en subred privada) puede salir a Internet
correctamente a través del NAT Gateway.

### Conectar EC2 a RDS
Una vez la instancia está `running`, conéctate por SSH y valida la
conexión a la base de datos:

```bash
ssh -i "tu-llave.pem" ec2-user@<IP-PUBLICA-EC2>

# Dentro de la instancia EC2:
sudo yum install -y mysql
mysql -h <endpoint-rds.amazonaws.com> -u admin -p
```

Si la conexión es exitosa, verás el prompt `mysql>` — esto confirma que
el Security Group `sg-rds` está correctamente configurado para aceptar
tráfico únicamente desde la instancia autorizada.

---

## Parte C — Configuración en Visual Studio Code

### 1. Conectarse a la instancia EC2 vía Remote-SSH
Agrega esta entrada a tu archivo `~/.ssh/config`:

```
Host acme-app-demo
    HostName <IP-PUBLICA-EC2>
    User ec2-user
    IdentityFile ~/.ssh/tu-llave.pem
```

Luego, en VS Code: `Ctrl+Shift+P` → **Remote-SSH: Connect to Host** →
selecciona `acme-app-demo`. Esto abre una ventana de VS Code trabajando
directamente sobre el sistema de archivos de la instancia EC2.

### 2. Configurar credenciales de AWS Toolkit
`Ctrl+Shift+P` → **AWS: Create Credentials Profile** → ingresa el
`Access Key ID` y `Secret Access Key` entregados por AWS Academy Learner
Lab. Esto permite explorar recursos (RDS, S3, Lambda) directamente desde
el panel lateral de AWS Toolkit sin salir de VS Code.

### 3. Conectar a RDS desde SQLTools (extensión VS Code)
Instala la extensión **SQLTools + SQLTools MySQL/MariaDB Driver**, y
configura una nueva conexión:

```json
{
  "name": "acme-db-demo",
  "driver": "MySQL",
  "server": "<endpoint-rds.amazonaws.com>",
  "port": 3306,
  "database": "acme",
  "username": "admin"
}
```

Esto permite ejecutar y explorar consultas SQL directamente desde el
editor, con autocompletado de esquema.

---

## Parte D — Demostración de consultas en SQLiteOnline

Antes de ejecutar consultas contra el RDS real, se valida el esquema y la
lógica de las queries en **[sqliteonline.com](https://sqliteonline.com)**
(sin necesidad de instalación ni credenciales):

1. Abre sqliteonline.com → selecciona motor **SQLite** (sintaxis
   compatible para prototipado, aunque el destino final sea MySQL).
2. Crea el esquema base:

```sql
CREATE TABLE clientes (id INTEGER PRIMARY KEY, nombre TEXT, email TEXT, fecha_registro TEXT);
CREATE TABLE productos (id INTEGER PRIMARY KEY, nombre TEXT, precio REAL, stock INTEGER);
CREATE TABLE ventas (id INTEGER PRIMARY KEY, cliente_id INTEGER, producto_id INTEGER, cantidad INTEGER, fecha TEXT, monto_total REAL);
CREATE TABLE facturas (id INTEGER PRIMARY KEY, venta_id INTEGER, estado_pago TEXT, fecha_emision TEXT);
```

3. Inserta datos de prueba (mínimo 3-5 filas por tabla).
4. Ejecuta las 5 consultas documentadas en la
   [Lección 2](../docs/02-base-datos-relacional.md#consultas-sql-validadas-en-sqliteonline)
   y verifica que devuelvan resultados coherentes.
5. Una vez validada la lógica, las mismas consultas se ejecutan contra el
   RDS real desde SQLTools (Parte C) o desde la terminal `mysql` (Parte B).

---

## Evidencia a recolectar
Guardado en [`prototipo/consultas-sql/`](consultas-sql/):

- [x] Instancia RDS operativa — evidenciado indirectamente: las 5 consultas
      se ejecutaron con éxito contra el endpoint real de `acme-db-demo`
- [x] Instancia EC2 en estado `running` — evidenciado por el Instance ID
      (`i-0b4406025fcf512b3`) visible en el encabezado de cada sesión de
      Session Manager
- [x] Conexión Session Manager exitosa
- [x] Conexión `mysql` exitosa desde EC2 hacia RDS
- [x] Las 5 consultas ejecutándose contra RDS real, con resultados
- [ ] Consultas equivalentes en SQLiteOnline (prototipado previo)

> Nota: no se conservaron capturas separadas de la consola AWS mostrando
> los estados `Available`/`Running`, ya que los recursos fueron eliminados
> al finalizar la demo para evitar consumo innecesario del Free Tier. La
> evidencia funcional (consultas ejecutándose exitosamente) demuestra que
> ambos recursos estuvieron operativos.

## Limpieza de recursos
Al finalizar la demo, **detén o elimina** la instancia RDS y EC2 para
evitar consumir horas del Free Tier / AWS Academy innecesariamente:
`RDS → Actions → Delete` y `EC2 → Instance state → Terminate`.