# Ciclo de vida y protecciones (CloudFormation vs `lifecycle` de Terraform)

CloudFormation **no tiene un bloque único `lifecycle`**. Reparte esa
funcionalidad en dos niveles:

1. **Resource Attributes** — atributos por recurso (al mismo nivel que `Type` y
   `Properties`).
2. **Protecciones a nivel de stack** — se aplican con comandos aparte, NO en la
   plantilla.

---

## 1. Resource Attributes (el equivalente al `lifecycle { }`)

| Terraform `lifecycle { }` | CloudFormation | Qué hace |
|---|---|---|
| `prevent_destroy` | `DeletionPolicy: Retain` | No borra el recurso al borrar el stack. |
| (retención con copia) | `DeletionPolicy: Snapshot` | Al borrar, deja un snapshot (RDS, EBS...). |
| (qué pasa al reemplazar) | `UpdateReplacePolicy` | Si un cambio obliga a recrear, conserva/snapshotea el viejo. |
| `depends_on` | `DependsOn` | Orden explícito cuando no hay referencia entre recursos. |
| `create_before_destroy` | `UpdatePolicy` (solo ASG y algunos) | Comportamiento de actualización/rolling. No hay toggle general. |
| (esperar señal de arranque) | `CreationPolicy` | Espera N señales (cfn-signal) antes de marcar "creado". |
| `ignore_changes` | ❌ no existe equivalente | Ver sección drift abajo. |
| `replace_triggered_by` | ❌ no existe directo | Se simula metiendo un valor en el nombre/logical ID. |

### Ejemplo (ya usado en `nested-stacks/modules/bucket.yml`)

```yaml
Bucket:
  Type: AWS::S3::Bucket
  DeletionPolicy: Retain         # = prevent_destroy (no lo borra con el stack)
  UpdateReplacePolicy: Retain    # si hay que recrearlo, conserva el viejo
  Properties:
    ...
```

### Valores de `DeletionPolicy` / `UpdateReplacePolicy`

- `Delete` (por defecto): borra el recurso.
- `Retain`: lo deja (sale del control del stack pero no se destruye).
- `Snapshot`: deja una copia antes de borrar (solo recursos que soportan snapshot).

---

## 2. El hueco: no hay `ignore_changes`

Terraform usa `ignore_changes` para no pelear con cambios hechos por fuera
(ej. un tag que pone otro sistema). CloudFormation **no** tiene eso:

- Si algo cambia fuera del stack, es **drift**.
- Puedes detectarlo (`detect-stack-drift`), pero CloudFormation no lo ignora: al
  siguiente update intentará devolver el recurso al estado de la plantilla.

```bash
# detectar drift de un stack
aws cloudformation detect-stack-drift --stack-name poc-network-dev
aws cloudformation describe-stack-resource-drifts --stack-name poc-network-dev
```

---

## 3. Protecciones a nivel de STACK (no van en la plantilla)

Estas dos NO son atributos de recurso ni parámetros del template, y **tampoco se
configuran con `deploy`**. Se aplican con comandos propios.

### 3.1. Termination Protection — candado contra borrar el stack entero

Mientras está activada, `delete-stack` falla. Es el candado de "no me borres
esto por accidente".

```bash
# activar
aws cloudformation update-termination-protection \
  --enable-termination-protection \
  --stack-name poc-network-dev

# desactivar (para poder borrar luego)
aws cloudformation update-termination-protection \
  --no-enable-termination-protection \
  --stack-name poc-network-dev

# tambien se puede activar al crear
aws cloudformation create-stack ... --enable-termination-protection
```

> Diferencia con `DeletionPolicy`: `DeletionPolicy` protege **un recurso**;
> Termination Protection protege **el stack completo**.

### 3.2. Stack Policies — bloquear updates sobre recursos concretos

Un documento JSON que dice qué updates se permiten o deniegan **durante un
update del stack**. Útil para que nadie reemplace/borre por accidente un recurso
crítico (una VPC, una base de datos) al actualizar.

`stack-policy.json`:
```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/Vpc"
    },
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*"
    }
  ]
}
```

Aplicarlo:
```bash
# al crear
aws cloudformation create-stack ... --stack-policy-body file://stack-policy.json

# a un stack existente
aws cloudformation set-stack-policy \
  --stack-name poc-network-dev \
  --stack-policy-body file://stack-policy.json
```

**Acciones posibles**: `Update:Modify`, `Update:Replace`, `Update:Delete`,
`Update:*`. El orden importa: un `Deny` específico gana sobre un `Allow` general.

> Para hacer un cambio puntual saltándote la policy, se usa un
> `--stack-policy-during-update-body` solo para ese update.

---

## 4. Resumen de equivalencias

| Necesidad | Terraform | CloudFormation |
|-----------|-----------|----------------|
| No borrar un recurso | `prevent_destroy` | `DeletionPolicy: Retain` |
| Snapshot al borrar | (manual) | `DeletionPolicy: Snapshot` |
| Orden explícito | `depends_on` | `DependsOn` |
| Ignorar cambios externos | `ignore_changes` | ❌ (hay drift detection, no ignore) |
| Forzar recreación | `replace_triggered_by` | ❌ (truco con el nombre/logical ID) |
| Candado al destruir todo | (backend/policies) | Termination Protection |
| Bloquear updates puntuales | (policies IAM) | Stack Policies |

**Idea clave**: en CloudFormation el "lifecycle" no es un bloque, son **atributos
por recurso** (`DeletionPolicy`, `UpdateReplacePolicy`, `DependsOn`,
`CreationPolicy`, `UpdatePolicy`) + **protecciones de stack** (Termination
Protection, Stack Policies) que se aplican con comandos, no con la plantilla.
