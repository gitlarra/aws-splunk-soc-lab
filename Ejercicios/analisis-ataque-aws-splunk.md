# Experimentos AWS — Detección con CloudTrail y Splunk

Este documento recoge los experimentos realizados sobre la infraestructura AWS para generar eventos de seguridad reales y aprender a detectarlos en Splunk.

> ⚠️ **Aviso sobre costes:** Algunos experimentos como lanzar instancias EC2 de alta capacidad pueden generar costes elevados en cuentas AWS sin restricciones. Revisar siempre los límites de la cuenta antes de ejecutar cualquier acción.

---

## Contexto

CloudTrail registra todas las llamadas a la API de AWS — cada acción que realizas en la consola o por CLI genera un evento. Estos eventos se guardan en S3 y desde ahí Splunk los ingesta para su análisis.

Cada evento CloudTrail contiene siempre:
- **eventName** — qué acción se realizó
- **eventSource** — qué servicio de AWS (iam.amazonaws.com, s3.amazonaws.com...)
- **userIdentity** — quién lo hizo
- **sourceIPAddress** — desde dónde
- **errorCode / errorMessage** — si fue denegado y por qué

---

## Experimento 1 — Creación de usuario IAM con privilegios elevados

### Objetivo
Simular como un atacante que ha conseguido acceso a la consola AWS e intenta crear un usuario administrador para mantener persistencia.

### Acción
`IAM → Users → Create User`
- Nombre del usuario: `admin`

En AWS Academy esto devuelve `AccessDenied`. En una cuenta normal sin políticas restrictivas el usuario se crearía. El siguiente paso de un atacante sería adjuntarle `AdministratorAccess`.

### Evento generado
- `eventName`: `CreateUser`
- `eventSource`: `iam.amazonaws.com`
- `errorCode`: `AccessDenied`
- `errorMessage`: *not authorized to perform: iam:CreateUser on resource: arn:aws:iam::ACCOUNT_ID:user/admin*

### Por qué es relevante
La creación de usuarios IAM es una técnica de persistencia clásica en ataques a AWS (MITRE ATT&CK T1136 — Create Account). Un atacante crea un usuario con acceso permanente aunque se cambien las credenciales originales comprometidas.

### Cómo buscarlo en Splunk

**Paso 1 — Buscar el evento:**
```
index=cloudtrail-logs eventName=CreateUser
```
Esto devuelve todos los intentos de creación de usuario IAM, tanto exitosos como denegados.

**Paso 2 — Ver quién lo intentó y el resultado:**
```
index=cloudtrail-logs eventName=CreateUser
| eval actor=mvindex(split('userIdentity.arn', "/"), -1)
| eval resultado=if(isnotnull(errorCode), errorCode, "Success")
| table eventTime actor sourceIPAddress resultado errorMessage
```

- `mvindex(split(...), -1)` — divide el ARN por `/` y coge el último elemento, que es el nombre real del usuario o rol
- `isnotnull(errorCode)` — si existe errorCode fue denegado, si no existe tuvo éxito

**Paso 3 — Alerta si hay éxito:**

Lo realmente peligroso es un `CreateUser` con `resultado=Success` seguido de `AttachUserPolicy`. Busca:
```
index=cloudtrail-logs eventName IN (CreateUser, AttachUserPolicy)
| eval actor=mvindex(split('userIdentity.arn', "/"), -1)
| eval resultado=if(isnotnull(errorCode), errorCode, "Success")
| where resultado="Success"
| table eventTime eventName actor sourceIPAddress
```

---

## Experimento 2 — Creación de política IAM con privilegios de administrador

### Objetivo
Simular un atacante intentando crear una política personalizada con permisos de administrador para adjuntársela a un usuario o rol existente.

### Acción
`IAM → Policies → Create Policy`
- Nombre: `policyMasterAdmin`
- Permisos: `*:*` sobre todos los recursos

En AWS Academy devuelve `AccessDenied`. En una cuenta normal la política se crearía y podría adjuntarse a cualquier entidad.

### Evento generado
- `eventName`: `CreatePolicy`
- `eventSource`: `iam.amazonaws.com`
- `errorCode`: `AccessDenied`

### Por qué es relevante
Crear políticas con `*:*` es escalada de privilegios. Un atacante con permisos limitados puede intentar crear una política más permisiva y adjuntársela a sí mismo.

### Cómo buscarlo en Splunk

```
index=cloudtrail-logs eventName IN (CreatePolicy, AttachUserPolicy, AttachRolePolicy)
| eval actor=mvindex(split('userIdentity.arn', "/"), -1)
| eval resultado=if(isnotnull(errorCode), errorCode, "Success")
| table eventTime eventName actor sourceIPAddress resultado errorMessage
| sort -eventTime
```

El campo `requestParameters.policyDocument` contendría el JSON de la política si la acción tuvo éxito — ahí se vería si incluía `"Action": "*"`.

---

## Experimento 3 — Creación de bucket S3

### Objetivo
Simular la creación de un bucket S3 que podría usarse para exfiltración de datos o como almacenamiento de herramientas de ataque.

### Acción
`S3 → Create Bucket`
- Nombre: `s3-para-hacer-maldades`

Este experimento tiene éxito en AWS Academy. Un atacante con acceso a S3 puede crear buckets para exfiltrar datos de otros servicios.

### Evento generado
- `eventName`: `CreateBucket`
- `eventSource`: `s3.amazonaws.com`
- `requestParameters.bucketName`: nombre del bucket creado
- Sin errorCode — acción exitosa

### Por qué es relevante
La creación de buckets S3 es una técnica de exfiltración (MITRE ATT&CK T1537 — Transfer Data to Cloud Account). Un atacante copia datos sensibles a un bucket propio y los descarga desde fuera.

### Cómo buscarlo en Splunk

**Paso 1 — Ver todos los buckets creados:**
```
index=cloudtrail-logs eventName=CreateBucket
| eval actor=mvindex(split('userIdentity.arn', "/"), -1)
| table eventTime actor requestParameters.bucketName sourceIPAddress
| sort -eventTime
```

**Paso 2 — Detectar si el bucket se hizo público (señal de alerta máxima):**
```
index=cloudtrail-logs eventName IN (PutBucketAcl, PutBucketPolicy) 
| eval actor=mvindex(split('userIdentity.arn', "/"), -1)
| table eventTime eventName actor requestParameters.bucketName sourceIPAddress
```

Un bucket que se crea y al momento se hace público es una señal clara de exfiltración.

---

## Experimento 4 — Lanzar instancia EC2 de alta capacidad

### Objetivo
Simular un atacante intentando lanzar instancias grandes para minería de criptomonedas o procesamiento masivo (cryptojacking).

### Acción
`EC2 → Launch Instance`
- Tipo: `c8i.flex.8xlarge` (96 vCPUs, muy cara)

En AWS Academy devuelve `Client.UnauthorizedOperation`. En una cuenta normal **la instancia se lanzaría y generaría costes inmediatos** — una `c8i.8xlarge` cuesta ~$1.36/hora, pero instancias más grandes pueden llegar a $30/hora o más.

> ⚠️ **Si tienes una cuenta AWS sin restricciones, no lances instancias grandes sin límites de billing configurados. Activa AWS Budgets con alerta antes de hacer este experimento.**

### Evento generado
- `eventName`: `RunInstances`
- `eventSource`: `ec2.amazonaws.com`
- `requestParameters.instanceType`: tipo de instancia solicitada
- `errorCode`: `Client.UnauthorizedOperation` (en lab) o sin error (en cuenta normal)

### Por qué es relevante
El cryptojacking en AWS es uno de los ataques más comunes cuando se filtran credenciales. Un atacante lanza decenas de instancias grandes para minar criptomonedas — la víctima recibe una factura de miles de dólares.

### Cómo buscarlo en Splunk

**Paso 1 — Ver todas las instancias lanzadas:**
```
index=cloudtrail-logs eventName=RunInstances
| eval actor=mvindex(split('userIdentity.arn', "/"), -1)
| eval resultado=if(isnotnull(errorCode), errorCode, "Success")
| table eventTime actor requestParameters.instanceType sourceIPAddress resultado
| sort -eventTime
```

**Paso 2 — Detectar instancias grandes (posible cryptojacking):**
```
index=cloudtrail-logs eventName=RunInstances
| eval actor=mvindex(split('userIdentity.arn', "/"), -1)
| eval resultado=if(isnotnull(errorCode), errorCode, "Success")
| where NOT match(requestParameters.instanceType, "^t[23]\.")
| table eventTime actor requestParameters.instanceType sourceIPAddress resultado
```

Este filtro excluye instancias `t2.*` y `t3.*` (las pequeñas normales) y muestra solo las grandes o inesperadas.

---

## Dashboard AWS CloudTrail

El dashboard agrupa los cuatro experimentos en una vista única para el analista SOC.

### Cómo construirlo en Splunk

`Dashboards → Create New Dashboard`
- Title: `SOC - AWS CloudTrail Monitor`
- Type: Classic Dashboard

Añadir panels en este orden:

**Panel 1 — Tabla: Acciones IAM Sospechosas**
- Visualización: Table
- Query: experimento 1 y 2 combinados con `eventName IN (CreateUser, CreatePolicy, AttachUserPolicy, AttachRolePolicy)`
- Campos: `eventTime`, `eventName`, `actor`, `sourceIPAddress`, `resultado`, `errorMessage`

**Panel 2 — Tabla: Buckets S3 Creados**
- Visualización: Table
- Query: experimento 3
- Campos: `eventTime`, `actor`, `requestParameters.bucketName`, `sourceIPAddress`, `resultado`

**Panel 3 — Tabla: Instancias EC2 Lanzadas**
- Visualización: Table
- Query: experimento 4
- Campos: `eventTime`, `actor`, `requestParameters.instanceType`, `sourceIPAddress`, `resultado`

**Panel 4 — Gráfico: Actividad IAM por tipo de evento**
- Visualización: Bar chart
- Query: `index=cloudtrail-logs eventSource="iam.amazonaws.com" | stats count by eventName | sort -count`
- Muestra qué operaciones IAM son más frecuentes — útil para detectar reconocimiento (muchos List/Get antes de un Create)

**Panel 5 — Gráfico: IPs con más actividad**
- Visualización: Pie chart
- Query: `index=cloudtrail-logs NOT sourceIPAddress IN ("ec2.amazonaws.com", "cloudtrail.amazonaws.com", "s3.amazonaws.com") | stats count by sourceIPAddress | sort -count | head 10`
- Filtra IPs de servicios internos de AWS para ver solo IPs humanas

### Consideraciones del dashboard

- **Time range:** Last 24h por defecto — ajustar a Last 7 days para ver patrones
- **Auto-refresh:** cada 5 minutos si el lab está activo
- **Color coding:** en producción se usarían colores para distinguir Success (verde) de AccessDenied (rojo) — requiere configuración adicional de tokens en el XML

---

## Resumen de queries SPL

| Caso de uso                | Query base                                                         |
|----------------------------|--------------------------------------------------------------------|
| Todos los eventos IAM      | `index=cloudtrail-logs eventSource="iam.amazonaws.com"`            |
| Intentos de crear usuarios | `index=cloudtrail-logs eventName=CreateUser`                       |
| Políticas creadas          | `index=cloudtrail-logs eventName=CreatePolicy`                     |
| Buckets S3 creados         | `index=cloudtrail-logs eventName=CreateBucket`                     |
| Instancias EC2 lanzadas    | `index=cloudtrail-logs eventName=RunInstances`                     |
| Solo acciones denegadas    | Añadir `errorCode=AccessDenied` a cualquier query                  |
| Solo acciones exitosas     | Añadir `NOT errorCode=*` a cualquier query                         |
| Inventario de eventos      | `index=cloudtrail-logs \| stats count by eventName \| sort -count` |
