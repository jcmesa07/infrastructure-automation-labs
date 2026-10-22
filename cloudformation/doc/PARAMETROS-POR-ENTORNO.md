# Parámetros por entorno en CloudFormation (equivalente a los `.tfvars`)

Cómo manejar valores distintos por entorno (dev / qa / prod) con una sola
plantilla, y cómo se compara con los `.tfvars` de Terraform.

---

## 1. El mapeo mental

| Terraform | CloudFormation |
|-----------|----------------|
| bloque `variable "x" {}` | `Parameters:` en la plantilla |
| `dev.tfvars` / `prod.tfvars` | `params/dev.json` / `params/prod.json` |
| `terraform apply -var-file=dev.tfvars` | `deploy --parameter-overrides file://params/dev.json` |
| `-var "x=y"` (suelto) | `--parameter-overrides X=Y` (inline) |
| auto-carga de `terraform.tfvars` / `*.auto.tfvars` | ❌ no existe: **siempre** explícito |

La diferencia más importante: en CloudFormation **nunca** se carga un archivo de
parámetros automáticamente. Siempre indicas cuál usar. Es como si en Terraform
estuvieras obligado a pasar `-var-file` en cada `apply`.

---

## 2. La plantilla (define los parámetros)

Igual que los `variable` de Terraform, con tipos, valores por defecto y
validación (`AllowedValues`):

```yaml
Parameters:
  Entorno:
    Type: String
    AllowedValues: [dev, qa, prod]
    Description: Entorno; se usa como prefijo de los nombres.
  TipoInstancia:
    Type: String
    Default: t3.micro
```

---

## 3. Los archivos de parámetros (los "tfvars")

Un archivo por entorno. **El formato depende del comando** que uses.

### Formato lista (recomendado, sirve para todo)

```json
// params/dev.json
[
  { "ParameterKey": "Entorno",       "ParameterValue": "dev" },
  { "ParameterKey": "TipoInstancia", "ParameterValue": "t3.micro" }
]
```

```json
// params/prod.json
[
  { "ParameterKey": "Entorno",       "ParameterValue": "prod" },
  { "ParameterKey": "TipoInstancia", "ParameterValue": "m5.large" }
]
```

Comparación con Terraform:

```hcl
# dev.tfvars
entorno        = "dev"
tipo_instancia = "t3.micro"
```

---

## 4. Desplegar con el archivo del entorno

```bash
# equivalente a: terraform apply -var-file=dev.tfvars
aws cloudformation deploy \
  --template-file app.yml \
  --stack-name app-dev \
  --parameter-overrides file://params/dev.json

# otro entorno = solo cambias el archivo y el nombre del stack
aws cloudformation deploy \
  --template-file app.yml \
  --stack-name app-prod \
  --parameter-overrides file://params/prod.json
```

> `--parameter-overrides file://...` con el formato lista requiere **AWS CLI v2**.
> Con `create-stack` / `update-stack` se usa la misma lista pero con la bandera
> `--parameters file://params/dev.json`.

Inline, sin archivo (útil para un valor puntual):

```bash
aws cloudformation deploy ... --parameter-overrides Entorno=dev TipoInstancia=t3.micro
```

---

## 5. Un detalle clave: un stack por entorno

En CloudFormation cada entorno es **su propio stack** (`app-dev`, `app-qa`,
`app-prod`), normalmente también en **cuentas separadas**. El archivo de params
solo cambia los valores; el aislamiento lo da el `--stack-name` (y la cuenta).

Equivalencia con Terraform:

| Terraform | CloudFormation |
|-----------|----------------|
| `workspace` o directorio por entorno | `--stack-name` distinto por entorno |
| un `state` por entorno | un `stack` por entorno |
| `-var-file=<env>.tfvars` | `--parameter-overrides file://params/<env>.json` |

---

## 6. Alternativas a los archivos de params

No siempre necesitas un archivo externo:

- **`Mappings`** dentro de la plantilla: para diferencias pequeñas y fijas por
  entorno o región (ej. región → AMI ID). Se resuelve con `!FindInMap`. Es como
  un `lookup()` sobre un `map` local en Terraform.

  ```yaml
  Mappings:
    PorEntorno:
      dev:  { Tipo: t3.micro }
      prod: { Tipo: m5.large }
  Resources:
    Servidor:
      Type: AWS::EC2::Instance
      Properties:
        InstanceType: !FindInMap [PorEntorno, !Ref Entorno, Tipo]
  ```

- **SSM Parameter Store**: cuando el valor es dinámico o compartido entre stacks
  (ver `COMPARTIR-DATOS-ENTRE-STACKS.md`). Equivalente a un `data source` en TF.

- **Orquestadores** (Sceptre, Rain, o SAM con `samconfig.toml`): dan un manejo de
  entornos más parecido a los workspaces de Terraform, con un archivo de config
  por stack/entorno.

---

## 7. ¿Y si quiero el archivo de params en YAML en vez de JSON?

El archivo de parámetros **nativo** (`--parameters` / `--parameter-overrides
file://`) es **solo JSON**: el AWS CLI parsea ese `file://` como JSON y falla si
le pasas YAML. Pero hay formas de trabajar en YAML.

> Nota: Terraform tampoco usa YAML para los `.tfvars` (usa HCL o `.tfvars.json`).
> Que el archivo de valores tenga su propio formato es normal en ambas.

### Opción A — `--cli-input-yaml` (lo más nativo)

Las banderas globales `--cli-input-json` / `--cli-input-yaml` reciben **toda** la
entrada del comando en YAML. Funciona limpio con `create-stack`/`update-stack`:

```yaml
# deploy-dev.yaml
StackName: app-dev
TemplateURL: https://mi-bucket.s3.amazonaws.com/app.yml
Parameters:
  - ParameterKey: Entorno
    ParameterValue: dev
  - ParameterKey: TipoInstancia
    ParameterValue: t3.micro
```
```bash
aws cloudformation create-stack --cli-input-yaml file://deploy-dev.yaml
```

### Opción B — YAML simple + `yq` en el script (práctico)

Mantienes un YAML plano y lo conviertes a `Key=Value` al vuelo:

```yaml
# params/dev.yaml
Entorno: dev
TipoInstancia: t3.micro
```
```bash
aws cloudformation deploy --template-file app.yml --stack-name app-dev \
  --parameter-overrides $(yq -r 'to_entries|map("\(.key)=\(.value)")|.[]' params/dev.yaml)
```

### Opción C — orquestadores basados en YAML

Si quieres que **toda** la config por entorno sea YAML, **Sceptre** o **taskcat**
usan YAML nativo para parámetros por stack/entorno. Es el camino cuando el
proyecto crece.

### Recomendación

Para este lab, deja el **JSON** (nativo, sin dependencias). Si el JSON te
estorba, la opción B con `yq` es la más cómoda sin añadir herramientas pesadas.

---

## 8. Resumen

- `Parameters:` = tus `variable`.
- `params/<env>.json` = tus `<env>.tfvars`.
- `--parameter-overrides file://...` = `-var-file=...` (pero **siempre explícito**).
- Un **stack por entorno** (y a menudo una cuenta por entorno) es el aislamiento,
  equivalente a un `state`/`workspace` por entorno en Terraform.
- Para diferencias fijas y pequeñas, `Mappings` evita el archivo externo.
