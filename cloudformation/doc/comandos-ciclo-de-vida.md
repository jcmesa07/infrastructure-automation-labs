# Comandos del ciclo de vida de un stack de CloudFormation

Referencia de comandos del AWS CLI para gestionar un stack de principio a fin.
Los ejemplos usan el stack `mi-primer-ecr` y la plantilla `ecr.yaml`.

> Recuerda: los comandos corren contra la region y credenciales que tengas
> configuradas (`aws configure`). Si usas credenciales temporales (Access Key
> que empieza por `ASIA`), necesitas tambien el `aws_session_token`.

Resumen del ciclo:
**validate → deploy (create) → describe/events (observar) → deploy otra vez (update) → delete (destroy)**

---

## 0. Validar la plantilla antes de desplegar

```bash
aws cloudformation validate-template --template-body file://ecr.yaml
```

---

## 1. Crear / Actualizar el stack  (= terraform apply)

Mismo comando para crear y para actualizar. Con el mismo `--stack-name`,
CloudFormation detecta la diferencia y aplica solo lo necesario. Si no hay
cambios, responde `No changes to deploy`.

```bash
aws cloudformation deploy \
  --template-file ecr.yaml \
  --stack-name mi-primer-ecr
```

---

## 2. Previsualizar cambios antes de aplicar  (= terraform plan)

Esto genera un **change set** (el plan) sin ejecutarlo. Luego lo inspeccionas y,
si te convence, repites el `deploy` sin la bandera `--no-execute-changeset`.

```bash
aws cloudformation deploy \
  --template-file ecr.yaml \
  --stack-name mi-primer-ecr \
  --no-execute-changeset
```

---

## 3. Ver el estado del stack

```bash
# Todo el detalle
aws cloudformation describe-stacks --stack-name mi-primer-ecr

# Solo el estado (CREATE_COMPLETE, UPDATE_COMPLETE, etc.)
aws cloudformation describe-stacks \
  --stack-name mi-primer-ecr \
  --query "Stacks[0].StackStatus" --output text
```

---

## 4. Ver los eventos (el "log" de lo que paso)

Clave para depurar cuando algo falla.

```bash
aws cloudformation describe-stack-events \
  --stack-name mi-primer-ecr \
  --query "StackEvents[*].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output table
```

---

## 5. Ver los recursos que gestiona el stack

```bash
aws cloudformation list-stack-resources --stack-name mi-primer-ecr --output table
```

---

## 6. Ver las salidas (Outputs)

Util cuando la plantilla tiene seccion `Outputs`.

```bash
aws cloudformation describe-stacks \
  --stack-name mi-primer-ecr \
  --query "Stacks[0].Outputs" --output table
```

---

## 7. Listar todos tus stacks

```bash
aws cloudformation list-stacks \
  --query "StackSummaries[?StackStatus!='DELETE_COMPLETE'].[StackName,StackStatus]" \
  --output table
```

---

## 8. Borrar el stack  (= terraform destroy)

```bash
aws cloudformation delete-stack --stack-name mi-primer-ecr
```

> Nota: si el repo ECR ya tiene imagenes, el borrado falla porque ECR no elimina
> un repositorio con contenido. Borra las imagenes primero o hazlo desde la consola.

---

## 9. Comandos "wait" (para scripts y pipelines)

Bloquean la terminal hasta que la fase termina. Hay uno por fase:

```bash
aws cloudformation wait stack-create-complete --stack-name mi-primer-ecr
aws cloudformation wait stack-update-complete --stack-name mi-primer-ecr
aws cloudformation wait stack-delete-complete --stack-name mi-primer-ecr
```

---

## Equivalencias rapidas con Terraform

| Terraform            | CloudFormation                                  |
| -------------------- | ----------------------------------------------- |
| `terraform plan`     | `deploy --no-execute-changeset` (change set)    |
| `terraform apply`    | `deploy`                                         |
| `terraform destroy`  | `delete-stack`                                   |
| `terraform state list` | `list-stack-resources`                        |
| `terraform output`   | `describe-stacks --query "Stacks[0].Outputs"`   |
| `terraform validate` | `validate-template`                             |
