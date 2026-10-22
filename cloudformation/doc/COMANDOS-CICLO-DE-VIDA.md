# Comandos del ciclo de vida de un stack de CloudFormation

Referencia de comandos del AWS CLI para gestionar un stack de principio a fin,
**en el orden en que normalmente se usan**.

> Recuerda: los comandos corren contra la region y credenciales que tengas
> configuradas (`aws configure`). Si usas credenciales temporales (Access Key
> que empieza por `ASIA`), necesitas tambien el `aws_session_token`.

Orden del ciclo de vida:
**validate → plan (change set) → apply (deploy) → observar → destroy**

| # | Fase | Comando | Equivalente Terraform |
|---|------|---------|------------------------|
| 1 | Validar | `validate-template` | `terraform validate` |
| 2 | Plan | `deploy --no-execute-changeset` | `terraform plan` |
| 3 | Apply | `deploy` | `terraform apply` |
| 4 | Apply por entorno | `deploy --parameter-overrides file://...` | `apply -var-file` |
| 5-9 | Observar | `describe-stacks`, `describe-stack-events`, ... | `show` / `output` / `state list` |
| 10 | Destroy | `delete-stack` | `terraform destroy` |
| 11 | Utilidad | `wait ...` | (no aplica) |

---

## 1. Validar la plantilla  (= terraform validate)

Siempre primero: comprueba que la sintaxis es correcta antes de nada.

```bash
aws cloudformation validate-template --template-body file://ecr.yaml
```

---

## 2. Previsualizar cambios: el plan  (= terraform plan)

Genera un **change set** (el plan) sin ejecutarlo. Lo inspeccionas y, si te
convence, pasas al apply (paso 3) repitiendo el `deploy` sin la bandera.

```bash
aws cloudformation deploy \
  --template-file s3.yml \
  --stack-name mi-primer-s3 \
  --no-execute-changeset
```

---

## 3. Crear / Actualizar: el apply  (= terraform apply)

Mismo comando para crear y para actualizar. Con el mismo `--stack-name`,
CloudFormation detecta la diferencia y aplica solo lo necesario. Si no hay
cambios, responde `No changes to deploy`.

```bash
aws cloudformation deploy \
  --template-file ecr.yaml \
  --stack-name mi-primer-ecr
```

---

## 4. Apply con parametros por entorno  (= terraform apply -var-file)

Valores distintos por entorno con la misma plantilla, usando un archivo de
parametros por cada uno (equivalente a los `.tfvars`). Detalle en
`PARAMETROS-POR-ENTORNO.md`.

Archivo de parametros (formato lista):

```json
// params/dev.json
[
  { "ParameterKey": "Entorno", "ParameterValue": "dev" }
]
```

Desplegar pasando el archivo del entorno (AWS CLI v2):

```bash
# dev  (= terraform apply -var-file=dev.tfvars)
aws cloudformation deploy \
  --template-file app.yml \
  --stack-name app-dev \
  --parameter-overrides file://params/dev.json

# prod = mismo template, otro archivo y otro stack
aws cloudformation deploy \
  --template-file app.yml \
  --stack-name app-prod \
  --parameter-overrides file://params/prod.json
```

Inline, sin archivo (= `-var "x=y"`):

```bash
aws cloudformation deploy ... --parameter-overrides Entorno=dev TipoInstancia=t3.micro
```

Con `create-stack` / `update-stack` la misma lista se pasa con `--parameters`:

```bash
aws cloudformation create-stack \
  --stack-name app-dev \
  --template-body file://app.yml \
  --parameters file://params/dev.json
```

> OJO: CloudFormation **nunca** carga un archivo de params solo (no hay
> equivalente al `terraform.tfvars` automatico). Siempre lo indicas de forma
> explicita, como un `-var-file` obligatorio en cada despliegue.

### Nested stacks con params (el lab de `nested-stacks/`)

Primero `package` (sube los modulos a S3) y luego `deploy` con el archivo:

```bash
aws cloudformation package \
  --template-file parent-network.yml \
  --s3-bucket MI-BUCKET-ARTEFACTOS-cfn \
  --output-template-file parent-network.packaged.yml

aws cloudformation deploy \
  --template-file parent-network.packaged.yml \
  --stack-name poc-network-dev \
  --parameter-overrides file://params/network-dev.json
```

---

## 5. Ver el estado del stack

```bash
# Todo el detalle
aws cloudformation describe-stacks --stack-name mi-primer-s3

# Solo el estado (CREATE_COMPLETE, UPDATE_COMPLETE, etc.)
aws cloudformation describe-stacks \
  --stack-name mi-primer-s3 \
  --query "Stacks[0].StackStatus" --output text
```

---

## 6. Ver los eventos (el "log" de lo que paso)

Clave para depurar cuando algo falla.

```bash
aws cloudformation describe-stack-events \
  --stack-name mi-primer-s3 \
  --query "StackEvents[*].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output table
```

---

## 7. Ver los recursos que gestiona el stack  (= terraform state list)

```bash
aws cloudformation list-stack-resources --stack-name mi-primer-s3 --output table
```

---

## 8. Ver las salidas (Outputs)  (= terraform output)

Util cuando la plantilla tiene seccion `Outputs`.

```bash
aws cloudformation describe-stacks \
  --stack-name mi-primer-s3 \
  --query "Stacks[0].Outputs" --output table
```

---

## 9. Listar todos tus stacks

```bash
aws cloudformation list-stacks \
  --query "StackSummaries[?StackStatus!='DELETE_COMPLETE'].[StackName,StackStatus]" \
  --output table
```

---

## 10. Borrar el stack: el destroy  (= terraform destroy)

```bash
aws cloudformation delete-stack --stack-name mi-primer-s3
```

> Nota: si el repo ECR ya tiene imagenes, el borrado falla porque ECR no elimina
> un repositorio con contenido. Borra las imagenes primero o hazlo desde la consola.

---

## 11. Utilidad: comandos "wait" (para scripts y pipelines)

Bloquean la terminal hasta que la fase termina. Hay uno por fase; se usan dentro
de automatizaciones, intercalados con los comandos de arriba.

```bash
aws cloudformation wait stack-create-complete --stack-name mi-primer-ecr
aws cloudformation wait stack-update-complete --stack-name mi-primer-ecr
aws cloudformation wait stack-delete-complete --stack-name mi-primer-ecr
```

---

## 12. Protecciones del stack

Detalle conceptual en `LIFECYCLE-Y-PROTECCIONES.md`. Estas NO van en la plantilla
ni en `deploy`: se aplican con comandos propios.

### 12.1. Termination Protection (candado contra borrar el stack entero)

Mientras esta activa, `delete-stack` falla.

```bash
# activar
aws cloudformation update-termination-protection \
  --enable-termination-protection --stack-name poc-network-dev

# desactivar (para poder borrar luego)
aws cloudformation update-termination-protection \
  --no-enable-termination-protection --stack-name poc-network-dev

# tambien al crear
aws cloudformation create-stack ... --enable-termination-protection
```

### 12.2. Stack Policy (bloquear updates sobre recursos concretos)

Solo actua en los UPDATE (no en create ni en delete). Protege recursos criticos
de que un update los reemplace o borre por accidente.

```bash
# adjuntar/reemplazar la policy de un stack existente
aws cloudformation set-stack-policy \
  --stack-name poc-network-dev \
  --stack-policy-body file://stack-policy.json

# ver la policy actual
aws cloudformation get-stack-policy --stack-name poc-network-dev

# aplicarla al crear
aws cloudformation create-stack ... --stack-policy-body file://stack-policy.json

# saltarsela SOLO para un update puntual (sin cambiar la policy permanente)
aws cloudformation update-stack \
  --stack-name poc-network-dev \
  --template-body file://plantilla.yml \
  --stack-policy-during-update-body file://permitir-una-vez.json
```

> Una stack policy NO se puede eliminar, solo reemplazar por otra (mas permisiva).
> Un `Deny` especifico gana sobre un `Allow` general.

---

## 13. Drift: detectar cambios hechos fuera del stack

CloudFormation no tiene `ignore_changes`. Si algo cambio por fuera, se detecta
como drift (pero no se ignora: el proximo update lo revierte).

```bash
# lanzar la deteccion (devuelve un StackDriftDetectionId)
aws cloudformation detect-stack-drift --stack-name poc-network-dev

# ver el estado de la deteccion
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <ID>

# ver que recursos estan "drifted"
aws cloudformation describe-stack-resource-drifts \
  --stack-name poc-network-dev --output table
```

---

## Equivalencias rapidas con Terraform

| Terraform            | CloudFormation                                  |
| -------------------- | ----------------------------------------------- |
| `terraform validate` | `validate-template`                             |
| `terraform plan`     | `deploy --no-execute-changeset` (change set)    |
| `terraform apply`    | `deploy`                                         |
| `terraform apply -var-file=dev.tfvars` | `deploy --parameter-overrides file://params/dev.json` |
| `terraform apply -var "x=y"` | `deploy --parameter-overrides X=Y`        |
| `terraform state list` | `list-stack-resources`                        |
| `terraform output`   | `describe-stacks --query "Stacks[0].Outputs"`   |
| `terraform destroy`  | `delete-stack`                                   |
| `lifecycle { prevent_destroy }` | `DeletionPolicy: Retain` / `update-termination-protection` |
| drift (`plan`/`refresh`) | `detect-stack-drift` + `describe-stack-resource-drifts` |
