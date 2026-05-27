# Guía de Despliegue: SOC Lab AWS + Splunk

> Infraestructura SOC realista en AWS con ingesta de logs en Splunk desde múltiples fuentes.  
> Stack: CloudTrail + S3 + EC2 + VPC + Splunk (Docker) + Fluent Bit + Apache

Atención: Todo el lab está montado en un laboratorio de AWS, explico las limitaciones de este y como debéis actuar si teneis una cuenta sin limitaciones, es por ello que todo no es 100% como me gustaría o de la forma más segura.

---

## Arquitectura

```
Internet
    │
    ▼
[EC2 t2.micro - Apache]          [AWS CloudTrail]
    │                                 │
    │  Fluent Bit → HEC               ▼
    │                           [S3 Bucket]
    |                                  |
    └──────────────────────────────────┐
                                       ▼
                          [EC2 t3.medium - Splunk Docker]
                               Puerto 8000 (UI)
                               Puerto 8088 (HEC)
                               Splunk for AWS (lee S3)
```

---

## Fase 1 — CloudTrail + S3

### 1.1 Crear CloudTrail

`CloudTrail → Trails → Create trail`

**Trail name:** `logs-<tuproyecto>`

**Storage location:** seleccionar **"Create new S3 bucket"** y asignar un nombre.
Dejar que CloudTrail cree el bucket es importante — AWS aplica automáticamente la bucket policy necesaria para que CloudTrail pueda escribir en él. Si usas un bucket existente tendrás que añadir esa policy manualmente.

**Log file SSE-KMS encryption:** desactivado en lab.
En producción se activa para cifrar los logs con una clave KMS gestionada. Requiere permisos IAM adicionales en Splunk para descifrar.

**Log file validation:** activado.
Genera un archivo digest con hash criptográfico de cada log. Permite detectar si los logs han sido modificados o borrados (principio de non-repudiation de CompTIA Security+).

**Additional settings:**
- SNS notification: desactivado — Splunk gestionará las alertas
- CloudWatch Logs: desactivado — Splunk leerá los logs directamente desde S3, activarlo sería duplicar el almacenamiento y añadir coste sin beneficio.

**Tags:**
- `Project`: `soc-<tuproyecto>`
- `Environment`: `lab`

**Choose log events — Management Events (Read + Write):**

- **Write** — captura acciones que modifican recursos: crear EC2, borrar S3, modificar IAM. El más relevante para SOC porque detecta cambios no autorizados.
- **Read** — captura consultas sobre recursos: listar buckets, describir instancias. Permite detectar reconocimiento (atacante enumerando recursos de la cuenta).

Los demás tipos (Data Events, Insights Events, Network Activity) generan mucho volumen y tienen coste adicional — no necesarios para este lab.

### 1.2 Lifecycle Rule en el S3

Una vez creado el trail, configurar retención automática en el bucket:

`S3 → Bucket → Management → Lifecycle rules → Create lifecycle rule`

- **Rule name:** `lifecycle-<tuproyecto>`
- **Scope:** Apply to all objects (el bucket solo contiene logs de CloudTrail)
- **Acciones a marcar:**
  - `Transition current versions of objects between storage classes`
  - `Expire current versions of objects`

**Transiciones:**

| Días | Clase                     | Por qué                                              |
|------|---------------------------|------------------------------------------------------|
|  30  | S3 Standard-IA            | Logs ya ingestados por Splunk, acceso poco frecuente |
|  90  | Glacier Instant Retrieval | Archivado para posibles auditorías                   |
|  180 | Expiración (borrado)      | Evitar acumulación de costes                         |

---

## Fase 2 — Red (VPC)

### 2.1 Crear VPC con "VPC and more"

Usar la opción **"VPC and more"** en `VPC → Create VPC`. Esta opción crea todos los componentes de red en un solo paso de forma guiada.

Configuración:

- **Name tag:** `vpc-splunk-<tuproyecto>`
- **IPv4 CIDR:** `10.0.0.0/16`
- **Number of AZs:** 1
- **Number of public subnets:** 1
- **Number of private subnets:** 1
- **NAT Gateway:** Zonal- In 1 AZ
- **VPC endpoints:** None

AWS creará automáticamente: VPC, Internet Gateway, subnets pública y privada, route tables con las rutas correctas y NAT Gateway con Elastic IP.

> **¿Por qué 10.0.0.0/16?** Un /16 da 65.536 IPs y permite subdividir en subnets /24 cómodamente. Un /24 solo da 254 IPs totales — demasiado pequeño para una VPC.

> **NAT Gateway:** Permite que instancias en la subnet privada salgan a internet (actualizaciones, Docker pull) sin tener IP pública. El tráfico va: subnet privada → NAT → IGW → internet. Tiene coste (~$0.045/h) — apagar instancias cuando no se usen.

> **Private subnet:** En este laboratorio no tengo SSM para conectarme al EC2 con Splunk en la subred privada, por lo que yo usaré solo la subred pública, pero si tenéis esta opción, la VPC esta montada para hacer una red mas segura y aislada, colocando el Ec2 splunk en la subred privada.

### 2.2 Security Groups

**SG-Apache** (`sg-apache-victim`):

| Puerto | Protocolo |   Fuente  | Por qué      |
|--------|-----------|-----------|--------------|
|   80   |    TCP    | 0.0.0.0/0 | Web pública  |
|   22   |    TCP    | Tu IP/32  | SSH          |

**SG-Splunk** (`sg-splunk-siem`):

| Puerto | Protocolo |   Fuente   | Por qué              |
|--------|-----------|------------|----------------------|
|  8000  |    TCP    |  Tu IP/32  | Splunk UI            |
|  8088  |    TCP    |  Apache SG | HEC desde Fluent Bit |
|   22   |    TCP    |  Tu IP/32  | SSH                  |


> El outbound por defecto (todo permitido) es suficiente. Fluent Bit puede salir al puerto 8088 de Splunk sin reglas adicionales en el SG de Apache.
> En la VPC se ha creado el subnet privado porque es donde deberia de ir el Splunk, pero como no tengo acceso a SSM para trabajar con el EC2 del splunk lo pondre en la subnet pública, pero esto se tiene que evitar a toda costa, por ello se pone la IP public en los SG mencionados arriba.

---

## Fase 3 — EC2 Splunk

### 3.1 Lanzar instancia

| Campo                 | Valor                          |
|-----------------------|--------------------------------|
| Name                  | `ec2-splunk-siem`              |
| AMI                   | Ubuntu 24.04 LTS               |
| Instance type         | `t3.medium`                    |
| Key pair              | Crear o usar existente         |
| VPC                   | La creada                      |
| Subnet                | `subnet-public`                |
| Auto-assign public IP | Enabled                        |
| Security Group        | `sg-splunk-siem`               |
| Storage               | 30GB gp3                       |
| IAM Instance Profile  | Rol con permisos S3 (ver nota) |

> **⚠️ IAM Role:** Asignar el rol en el momento de crear la instancia, no después. Asignarlo post-creación puede no surtir efecto sin reinicio y causar problemas con el metadata service. En mi caso es un rol prehecho llamado myS3role y tiene acceso total a S3. Este mismo rol debe tener en Trusted relationships:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

### 3.2 Instalar Docker y lanzar Splunk

```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu

# Reconectar SSH para que tome efecto el grupo docker
exit
```

Reconectar y lanzar Splunk:

```bash
# ⚠️ Mapear los tres puertos explícitamente
# ⚠️ Usar volúmenes para persistir configuración entre recreaciones del contenedor
docker run -d \
  --name splunk \
  --restart always \
  -p 8000:8000 \
  -p 8088:8088 \
  -p 9997:9997 \
  -v splunk-var:/opt/splunk/var \
  -v splunk-etc:/opt/splunk/etc \
  -e SPLUNK_START_ARGS='--accept-license' \
  -e SPLUNK_GENERAL_TERMS='--accept-sgt-current-at-splunk-com' \
  -e SPLUNK_PASSWORD='TuPasswordSegura123!' \
  splunk/splunk:latest
```

Verificar estado con watch docker ps (esperar 1 minuto aprox. hasta STATUS = healthy )

> **Error común — puertos no mapeados:** Si en `docker ps` ves `8088-8089/tcp` sin `0.0.0.0:8088->8088/tcp`, el puerto no está publicado en el host. Fluent Bit recibirá `Connection refused`. Solución: recrear el contenedor con `-p 8088:8088` explícito.

> **Error común — configuración perdida al recrear:** Sin volúmenes, al recrear el contenedor se pierden tokens HEC, índices y toda la configuración. Los volúmenes `splunk-var` y `splunk-etc` persisten todo.

> **Error común — contraseña:** Si el contenedor se queda en "health: starting" todo el rato puede ser que la contraseña no cumpla los requisitos. Podéis verlo con docker logs splunk

Acceder a UI: `http://IP-PUBLICA-EC2-SPLUNK:8000`

### 3.3 Activar HEC y crear token

1. `Settings → Data Inputs → HTTP Event Collector`
2. `Global Settings` → All Tokens Enable → Enable SSL NO (Solo porque estamos en lab) → Puerto 8088 → Save
3. `New Token`:
   - Name: `fluent-bit-apache`
   - Source type: Automatic
   - Default Index: `apache-logs`
4. Copiar el token generado

---

## Fase 4 — EC2 Apache

### 4.1 Lanzar instancia

| Campo                 | Valor                   |
|-----------------------|-------------------------|
| Name                  | `ec2-apache-victim`     |
| AMI                   | Ubuntu 24.04 LTS        |
| Instance type         | `t2.micro`              |
| Key pair              |  Crear o usar existente |
| Subnet                | `subnet-public`         |
| Auto-assign public IP | Enabled                 |
| Security Group        | `sg-apache-victim`      |
| Storage               | 10GB gp3                |

### 4.2 Instalar Apache

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apache2
sudo systemctl enable apache2
sudo systemctl start apache2
```

Verificar: `http://IP-PUBLICA-EC2-APACHE` → página por defecto de Ubuntu/Apache.

---

## Fase 5 — Fluent Bit

### 5.1 Instalar

```bash
curl https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh
sudo systemctl enable fluent-bit
```

### 5.2 Configurar

```bash
sudo nano /etc/fluent-bit/fluent-bit.conf
```

Reemplazar todo el contenido:

```ini
[SERVICE]
    Flush        5
    Daemon       Off
    Log_Level    info
    parsers_file parsers.conf
    plugins_file plugins.conf

[INPUT]
    Name    tail
    Path    /var/log/apache2/access.log
    Tag     apache.access
    DB      /var/log/fluent-bit-apache-access.db
    Parser  apache2

[INPUT]
    Name    tail
    Path    /var/log/apache2/error.log
    Tag     apache.error
    DB      /var/log/fluent-bit-apache-error.db

[OUTPUT]
    Name          splunk
    Match         *
    Host          <IP-PRIVADA-EC2-SPLUNK>
    Port          8088
    Splunk_Token  <TU-TOKEN-HEC>
    TLS           Off
    TLS.Verify    Off
```

> **Error común — `interval_sec` en INPUT:** La propiedad `interval_sec` no es válida en versiones recientes de Fluent Bit. Eliminarla del bloque INPUT si aparece.
> **TLS off:** Solo en laboratorio.

```bash
sudo systemctl restart fluent-bit
sudo systemctl status fluent-bit

# Ver logs en tiempo real para debug
sudo journalctl -u fluent-bit -f
```

---

## Fase 6 — Configurar Splunk para AWS

### 6.1 Instalar Splunk Add-on for AWS

`Apps (Menú lateral izquierdo) → Find More Apps` → buscar `Splunk Add-on for Amazon Web Services`

**Requiere cuenta en Splunkbase** (gratuita). 

Alternativa: descargar `.tgz` de splunkbase.splunk.com e instalar via `Apps → Manage Apps → Install app from file`.

### 6.2 Configurar cuenta AWS en el Add-on

`Splunk Add-on for AWS → Configuration → IAM Role → Add`

| Campo        | Valor                                      |
|--------------|--------------------------------------------|
| Name         | `myS3Role`                                 |
| IAM Role ARN | `arn:aws:iam::TU-ACCOUNT-ID:role/myS3Role` |

> **En AWS Academy/Vocareum:** No se pueden crear usuarios IAM con access keys. Usar IAM Role asignado a la EC2. El Add-on lo detecta automáticamente via instance metadata.

> **Error común — AssumeRole sobre sí mismo:** No poner el mismo rol en "AWS Account" y "Assume Role". Si el rol ya está en el Instance Profile, dejar "Assume Role" vacío.

### 6.3 Crear índice para CloudTrail

`Settings → Indexes → New Index`

| Campo      | Valor             |
|------------|-------------------|
| Index name | `cloudtrail-logs` |
| Max size   | 5GB               |

### 6.4 Crear input S3 para CloudTrail

`Splunk Add-on for AWS → Inputs → Create New Input → CloudTrail → Incremental S3`

Lo recomendado es usar la opción de SQS, en mi laboratorio no puedo, pero si vosotros podéis escoger esta opción.

| Campo           | Valor                           |
|-----------------|---------------------------------|
| Name            | `cloudtrail-s3-<tuproyecto>`    |
| AWS Account     | `myS3Role`                      |
| Assume Role     | Vacío                           |
| AWS Region      | Donde tengáis el S3             |
| S3 Bucket       | El S3 donde se guarden los logs |
| Log File Prefix | `AWSLogs/`                      |
| Source Type     | `aws:cloudtrail`                |
| Index           | `cloudtrail-logs`               |
| Start Date/Time | Inicio del lab o la que queráis |

---

## Fase 7 — Verificación

### Verificar logs Apache

```
index=apache-logs
```

Genera tráfico para poblar:

```bash
curl http://IP-PUBLICA-EC2-APACHE
curl http://IP-PUBLICA-EC2-APACHE/pagina-no-existe
curl http://IP-PUBLICA-EC2-APACHE/admin
```

### Verificar logs CloudTrail

```
index=cloudtrail-logs
```

Genera actividad en la consola AWS (navegar por servicios, crear recursos) y esperar 10-15 minutos.

---

