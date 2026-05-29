# Guía de Despliegue: SOC Lab AWS + Splunk con Terraform

> Infraestructura SOC realista en AWS con ingesta de logs en Splunk desde múltiples fuentes, desplegada como código con Terraform.  
> Stack: CloudTrail + S3 + EC2 + VPC + Splunk (Docker) + Fluent Bit + Apache

Esta guía es la versión Terraform de la guía de despliegue manual. A diferencia de aquella, aquí toda la infraestructura se define como código y se despliega con un solo `terraform apply`. Las fases de configuración manual dentro de Splunk (HEC, Add-on for AWS) siguen siendo necesarias y se cubren al final.

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

## Estructura del proyecto

```
mi-laboratorio-terraform/
├── terraform.tf        # Proveedores y credenciales (AWS)
├── network.tf          # Módulo VPC, subredes y NAT Gateway
├── security.tf         # SGs y reglas ingress/egress de Apache y Splunk
├── iam.tf              # Rol IAM, Policy Attachment e Instance Profile para SSM
├── compute.tf          # Instancias EC2 de Apache (victim) y Splunk (siem)
├── cloudtrail.tf       # Trail, Bucket S3, S3 Policy y bloques data dinámicos
└── scripts/
    ├── apache.sh       # Script de aprovisionamiento de Apache + Fluent Bit
    └── splunk.sh       # Script de aprovisionamiento de Splunk en Docker
```

Ya que hay servicios que dependen de otros, hay que montar la infraestructura desde la base hacia arriba: VPC → Security Groups → IAM → EC2 → S3 → CloudTrail.

---

## Requisitos previos

- AWS CLI instalado
- Plugin Session Manager instalado para hacer port forwarding:
  ```bash
  aws ssm start-session --target <INSTANCE-ID> \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["8000"],"localPortNumber":["8000"]}'
  ```

---

## Fase 1 — Provider (`terraform.tf`)

El primer archivo define el provider de AWS y las credenciales. En este laboratorio se usan credenciales de prueba porque el despliegue se realiza contra LocalStack.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.28.0"
    }
  }
}

provider "aws" {
  access_key = "test"
  secret_key = "test"
  region     = "us-east-1"
}
```

> En una cuenta AWS real se eliminan `access_key` y `secret_key` del código y se usan variables de entorno o un perfil de AWS CLI. Nunca se hardcodean credenciales reales.

---

## Fase 2 — Red (`network.tf`)

### 2.1 VPC

Se usa un módulo en vez de un `resource` directo porque el módulo crea en un solo bloque todo lo necesario: VPC, Internet Gateway, subnets pública y privada, route tables y NAT Gateway. Hacerlo con recursos individuales requeriría configurar todo esto a mano.

La información del módulo se puede consultar en: https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "splunk-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a"]
  private_subnets = ["10.0.1.0/24"]
  public_subnets  = ["10.0.101.0/24"]

  enable_nat_gateway = true

  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}
```

> **¿Por qué `10.0.0.0/16`?** Un /16 da 65.536 IPs y permite subdividir en subnets /24 cómodamente. Un /24 solo da 254 IPs totales, demasiado pequeño para una VPC.

> **NAT Gateway:** Permite que las instancias en la subnet privada salgan a internet (actualizaciones, Docker pull) sin tener IP pública. El tráfico va: subnet privada → NAT → IGW → internet. Con `enable_nat_gateway = true` el módulo lo crea y configura las route tables automáticamente.

---

## Fase 3 — Security Groups (`security.tf`)

La documentación de Terraform recomienda no usar los bloques `ingress` y `egress` dentro del propio `aws_security_group`, sino definirlos como recursos separados: `aws_vpc_security_group_ingress_rule` y `aws_vpc_security_group_egress_rule`.

Para referenciar la VPC creada en el módulo anterior se sigue el patrón `module.<nombre_modulo_en_tf>.<argumento>`.

### SG Apache

Abre el puerto 80 desde cualquier origen para servir tráfico web.

```hcl
resource "aws_security_group" "SGapache" {
  name        = "SGApache"
  description = "Allow 80"
  vpc_id      = module.vpc.vpc_id

  tags = {
    Name = "allow_80"
  }
}

resource "aws_vpc_security_group_ingress_rule" "http" {
  security_group_id = aws_security_group.SGapache.id
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 80
  ip_protocol       = "tcp"
  to_port           = 80
}

resource "aws_vpc_security_group_egress_rule" "todoapache" {
  security_group_id = aws_security_group.SGapache.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```
> El egress con `ip_protocol = "-1"` permite todo el tráfico saliente. En producción habría que limitarlo.

### SG Splunk

Solo abre el puerto 8088 (HEC) y únicamente desde instancias que tengan el SG de Apache. Esto se consigue con `referenced_security_group_id` en lugar de un CIDR.

No se abre el puerto 22 porque la instancia está en subred privada y se accede a ella mediante SSM. No se abre el 8000 porque se accede a la UI de Splunk mediante port forwarding, simulando estar dentro de la red de AWS.

```hcl
resource "aws_security_group" "SGsplunk" {
  name        = "SGsplunk"
  description = "Allow 8088 for HEC"
  vpc_id      = module.vpc.vpc_id

  tags = {
    Name = "allow_8088"
  }
}

resource "aws_vpc_security_group_ingress_rule" "HEC" {
  security_group_id             = aws_security_group.SGsplunk.id
  referenced_security_group_id = aws_security_group.SGapache.id
  from_port                     = 8088
  ip_protocol                   = "tcp"
  to_port                       = 8088
}

resource "aws_vpc_security_group_egress_rule" "todosplunk" {
  security_group_id = aws_security_group.SGsplunk.id
  cidr_ipv4         = "0.0.0.0/0"
  ip_protocol       = "-1"
}
```

> El egress con `ip_protocol = "-1"` permite todo el tráfico saliente. En producción habría que limitarlo.

---

## Fase 4 — IAM (`iam.tf`)

Para que la instancia de Splunk sea accesible mediante SSM sin necesidad de SSH ni bastión, necesita un IAM Role con la policy `AmazonSSMManagedInstanceCore`.

Esta policy incluye:
- **Session Manager** — acceso interactivo a la consola sin puertos SSH abiertos.
- **Ejecución de SSM** — Run Command, Fleet Manager, State Manager.
- **Inventario y parches** — recopilación de metadatos y aplicación de parches.

Se necesitan tres recursos: 
- Crear el rol --> https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role
- Adjuntar la policy --> https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role_policy_attachment
- Crear el instance profile --> https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_instance_profile

```hcl
resource "aws_iam_role" "SSMEC2Role" {
  name = "SSMEC2Role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Environment = "dev"
  }
}

resource "aws_iam_role_policy_attachment" "attach-policy-role-SSMEC2Role" {
  role       = aws_iam_role.SSMEC2Role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "SSMEC2Profile" {
  name = "SSMEC2Profile"
  role = aws_iam_role.SSMEC2Role.name
}
```

---

## Fase 5 — EC2 (`compute.tf`)

### Instancia Apache (victim)

Va en la subnet pública, tiene IP pública y el SG de Apache. Se aprovisiona automáticamente con `user_data` usando el script `apache.sh` que debeis de guardar en un directorio llamado scripts, dentro del directorio de terraform.

```hcl
resource "aws_instance" "apache" {
  ami           = "ami-0fc0d6e8d70ab2d42"
  instance_type = "t2.micro"

  subnet_id                   = module.vpc.public_subnets[0]
  associate_public_ip_address = true

  vpc_security_group_ids = [aws_security_group.SGapache.id]
  iam_instance_profile   = aws_iam_instance_profile.SSMEC2Profile.name
  user_data              = file("scripts/apache.sh")

  tags = {
    Name = "victim"
  }
}
```

### Instancia Splunk (siem)

Va en la subnet privada, sin IP pública. El módulo VPC ya configura las route tables para que el tráfico saliente vaya a través del NAT Gateway, por lo que no hay que hacer nada adicional en el recurso.

```hcl
resource "aws_instance" "splunk" {
  ami           = "ami-0fc0d6e8d70ab2d42"
  instance_type = "t3.medium"

  subnet_id                   = module.vpc.private_subnets[0]
  private_ip                  = "10.0.1.10"
  associate_public_ip_address = false

  vpc_security_group_ids = [aws_security_group.SGsplunk.id]
  iam_instance_profile   = aws_iam_instance_profile.SSMEC2Profile.name
  user_data              = file("scripts/splunk.sh")

  tags = {
    Name = "siem"
  }
}
```

### Scripts de aprovisionamiento

Los scripts se almacenan en la carpeta `scripts/` y se pasan a las instancias mediante `user_data`. No se incluyen inline en Terraform para mantener el código limpio.

**`scripts/splunk.sh`** — instala Docker y lanza Splunk con los puertos y volúmenes necesarios:

```bash
#!/bin/bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu

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
  -e SPLUNK_PASSWORD='<COLOCA-LA-CONTRASEÑA>' \
  splunk/splunk:latest
```

**`scripts/apache.sh`** — instala Apache y Fluent Bit, y escribe la configuración de Fluent Bit directamente con `tee`:

```bash
#!/bin/bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apache2
sudo systemctl enable apache2
sudo systemctl start apache2

curl https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh
sudo systemctl enable fluent-bit

sudo tee /etc/fluent-bit/fluent-bit.conf > /dev/null << 'EOF'
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
EOF

sudo systemctl restart fluent-bit
```

> En `<IP-PRIVADA-EC2-SPLUNK>`, debes colocar la Ip privada que hemos asignado en el resource AWS_instance para Splunk 
> En `<TU-TOKEN-HEC>`, una vez se levante la infraestructura, debeis de ir a Splunk, crear un HEC y colocarlo ahí, de lo contraro, no habrá conexión.

---

## Fase 6 — S3 y CloudTrail (`cloudtrail.tf`)

CloudTrail necesita un bucket S3 con una bucket policy específica que le permita escribir logs. Como el bucket debe existir antes que el trail, se usa `depends_on` en el recurso `aws_cloudtrail` para asegurar el orden correcto.

La policy se genera con `data "aws_iam_policy_document"`. El bloque `data` no crea infraestructura, solo genera el JSON de la policy que luego consume `aws_s3_bucket_policy`. Los valores de cuenta, región y partición se obtienen dinámicamente con data sources para no hardcodearlos.

```hcl
resource "aws_cloudtrail" "trail" {
  depends_on = [aws_s3_bucket_policy.s3iampolicy]

  name                          = "trail-logs"
  s3_bucket_name                = aws_s3_bucket.bucket.id
  include_global_service_events = true
}

resource "aws_s3_bucket" "bucket" {
  bucket        = "bucket-logs"
  force_destroy = true
}

data "aws_iam_policy_document" "iampolicy" {
  statement {
    sid    = "AWSCloudTrailAclCheck"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }

    actions   = ["s3:GetBucketAcl"]
    resources = [aws_s3_bucket.bucket.arn]

    condition {
      test     = "StringEquals"
      variable = "aws:SourceArn"
      values   = ["arn:${data.aws_partition.current.partition}:cloudtrail:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:trail/trail-logs"]
    }
  }

  statement {
    sid    = "AWSCloudTrailWrite"
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }

    actions   = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.bucket.arn}/AWSLogs/${data.aws_caller_identity.current.account_id}/*"]

    condition {
      test     = "StringEquals"
      variable = "s3:x-amz-acl"
      values   = ["bucket-owner-full-control"]
    }

    condition {
      test     = "StringEquals"
      variable = "aws:SourceArn"
      values   = ["arn:${data.aws_partition.current.partition}:cloudtrail:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:trail/trail-logs"]
    }
  }
}

resource "aws_s3_bucket_policy" "s3iampolicy" {
  bucket = aws_s3_bucket.bucket.id
  policy = data.aws_iam_policy_document.iampolicy.json
}

data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}
data "aws_region" "current" {}
```

> `include_global_service_events = true` hace que CloudTrail capture también eventos de servicios globales como IAM, lo que es relevante para un SOC porque permite detectar cambios no autorizados en roles y políticas.

---

## Fase 7 — Despliegue

```bash
# Inicializar y descargar módulos y providers
terraform init

# Revisar el plan antes de aplicar
terraform plan

# Desplegar la infraestructura
terraform apply
```

---

## Fase 8 — Configuración manual de Splunk

Una vez desplegada la infraestructura, hay que acceder a Splunk mediante SSM port forwarding y completar la configuración manualmente.

### 8.1 Acceder a Splunk via SSM

Previamente debe estar configurado AWS CLI e instalado el Plugin Session Manager
```bash
aws ssm start-session --target <INSTANCE-ID> \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8000"],"localPortNumber":["8000"]}'
```

Acceder a la UI en: `http://localhost:8000`

### 8.2 Activar HEC y crear token

1. `Settings → Data Inputs → HTTP Event Collector`
2. `Global Settings` → All Tokens: Enable → SSL: No (solo en lab) → Puerto 8088 → Save
3. `New Token`:
   - Name: `fluent-bit-apache`
   - Source type: Automatic
   - Default Index: `apache-logs`
4. Copiar el token generado y actualizar `apache.sh` con el valor real

> Ahora debeis de ir al EC2 de apache y en la ruta /etc/fluent-bit/fluent-bit.conf sustituir `<TU-TOKEN-HEC>` por el HEC que acabais de crear.

### 8.3 Instalar Splunk Add-on for AWS

`Apps → Find More Apps` → buscar `Splunk Add-on for Amazon Web Services`

Requiere cuenta en Splunkbase (gratuita). Alternativa: descargar `.tgz` de splunkbase.splunk.com e instalar via `Apps → Manage Apps → Install app from file`.

### 8.4 Configurar cuenta AWS en el Add-on

`Splunk Add-on for AWS → Configuration → IAM Role → Add`

| Campo        | Valor                                           |
|--------------|-------------------------------------------------|
| Name         | `SSMEC2Role`                                    |
| IAM Role ARN | `arn:aws:iam::<ACCOUNT-ID>:role/SSMEC2Role`     |

> El Add-on detecta automáticamente el rol vía instance metadata. Dejar "Assume Role" vacío para evitar el error de AssumeRole sobre sí mismo.

### 8.5 Crear índice para CloudTrail

`Settings → Indexes → New Index`

| Campo      | Valor             |
|------------|-------------------|
| Index name | `cloudtrail-logs` |
| Max size   | 5 GB              |

### 8.6 Crear input S3 para CloudTrail

`Splunk Add-on for AWS → Inputs → Create New Input → CloudTrail → Incremental S3`

> Se recomienda usar la opción SQS si está disponible en tu cuenta.

| Campo           | Valor                        |
|-----------------|------------------------------|
| Name            | `cloudtrail-s3-lab`          |
| AWS Account     | `SSMEC2Role`                 |
| Assume Role     | Vacío                        |
| AWS Region      | `us-east-1`                  |
| S3 Bucket       | `bucket-logs`                |
| Log File Prefix | `AWSLogs/`                   |
| Source Type     | `aws:cloudtrail`             |
| Index           | `cloudtrail-logs`            |
| Start Date/Time | Inicio del lab               |

---

## Fase 9 — Verificación

### Logs Apache

```
index=apache-logs
```

Generar tráfico para poblar:

```bash
curl http://IP-PUBLICA-EC2-APACHE
curl http://IP-PUBLICA-EC2-APACHE/pagina-no-existe
curl http://IP-PUBLICA-EC2-APACHE/admin
```

### Logs CloudTrail

```
index=cloudtrail-logs
```

Generar actividad en la consola AWS (navegar por servicios, crear recursos) y esperar 10-15 minutos.

---
