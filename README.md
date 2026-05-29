# SOC Lab: AWS + Splunk

Tutorial para aprender a montar un laboratorio SOC en AWS con ingesta de logs en Splunk desde múltiples fuentes. El objetivo es replicar un entorno de monitorización de seguridad que un analista SOC encontraría en producción.

> ⚠️ Este lab está montado sobre AWS Academy (Vocareum), que tiene limitaciones de permisos IAM. En la guía se indica qué pasos cambian en una cuenta de AWS sin restricciones.

---

## Stack

| Componente   | Tecnología                          |
|--------------|-------------------------------------|
| Cloud        | AWS (EC2, S3, VPC, CloudTrail, IAM) |
| SIEM         | Splunk Enterprise (Docker)          |
| Web server   | Apache2 (apt install)               |
| Ingesta Logs | Fluent Bit                          |
| OS           | Ubuntu 24.04 LTS                    |
| IaC          | Terraform                           |

---

## Terraform

En la carpeta `terraform/` se encuentra toda la infraestructura definida como código, lista para desplegar con un solo `terraform apply`.

```
terraform/
├── infraestructura/     # Archivos .tf y scripts de aprovisionamiento
└── guia/                # Guía paso a paso del despliegue con Terraform
```

---

## Fuentes de logs

| Fuente                   | Método                     | Index Splunk    |
|--------------------------|----------------------------|-----------------|
| Apache access/error logs | Fluent Bit → HEC           | apache-logs     |
| AWS CloudTrail           | Splunk Add-on for AWS → S3 | cloudtrail-logs |

---

## Casos de uso SOC

### AWS CloudTrail
- Detección de intentos de creación de usuarios IAM no autorizados
- Monitorización de buckets S3 creados
- Control de instancias EC2 lanzadas y tipos de instancia
- Detección de escalada de privilegios (AttachUserPolicy, AttachRolePolicy)

### Apache
- Monitorización de accesos a rutas sospechosas
- Detección de fuerza bruta en páginas protegidas (401)
- Tráfico total

---

## Requisitos

- Cuenta AWS (con permisos completos para reproducirlo sin limitaciones)
- Splunk Enterprise (free trial o licencia)

---

## Conceptos cubiertos

CloudTrail S3 Lifecycle VPC Security Groups IAM Roles Docker Fluent Bit Splunk HEC SPL SOC Log Analysis Threat Detection Terraform
