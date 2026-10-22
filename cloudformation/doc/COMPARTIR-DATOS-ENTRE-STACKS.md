# Compartir datos entre stacks en CloudFormation (comparado con Terraform)

Guía de cómo pasar valores (un VPC ID, un ARN, un endpoint) de una pieza de
infraestructura a otra, y cuándo usar cada mecanismo. En cada sección se compara
con el equivalente en **Terraform**.

---

## 1. La idea central: ¿dónde está la frontera?

Lo que define la dificultad de pasar datos no es la herramienta, es **dónde pone
cada una la frontera de estado**:

| | Unidad de estado | Pasar datos *dentro* | Pasar datos *fuera* |
|---|---|---|---|
| **CloudFormation** | el **stack** | fácil (nested, `!GetAtt`) | explícito (Export / SSM) |
| **Terraform** | el **state** | fácil (módulos, en memoria) | explícito (`remote_state`, `data`) |

Dentro de la frontera: el paso de valores es directo.
Cruzar la frontera: ambas herramientas exigen un **contrato explícito**.

Terraform parece "más fácil" solo porque, por defecto, sus módulos viven en un
**mismo state**. En cuanto divides en states separados (recomendado a escala),
Terraform recae en la misma fricción que CloudFormation.

---

## 2. Dentro de una misma unidad: nested stacks vs módulos

### CloudFormation: nested stacks + `!GetAtt`

El padre lee los `Outputs` de un hijo directamente:

```yaml
# Padre
Resources:
  Red:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: modules/network.yml

  App:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: modules/app.yml
      Parameters:
        VpcId: !GetAtt Red.Outputs.VpcId   # output del hijo -> input de otro hijo
```

### Terraform: módulos

```hcl
module "network" {
  source = "./modules/network"
}

module "app" {
  source = "./modules/app"
  vpc_id = module.network.vpc_id   # output de un módulo -> input de otro
}
```

**Equivalencias:**

| CloudFormation (nested) | Terraform (módulos) |
|---|---|
| `Parameters:` del hijo | `variables.tf` |
| `Outputs:` del hijo | `outputs.tf` |
| `!GetAtt Red.Outputs.VpcId` | `module.red.vpc_id` |
| recurso `AWS::CloudFormation::Stack` | bloque `module "x"` |

En ambos casos la dependencia es **automática** y el valor se resuelve en una
sola ejecución (ver sección 5).

---

## 3. Entre stacks independientes — Opción A: Export / `!ImportValue`

Cuando son **dos stacks raíz distintos**, `!GetAtt` ya no sirve. Primera opción:
exportar un valor con nombre único e importarlo.

```yaml
# ---- Stack A (red) ----
Outputs:
  VpcId:
    Value: !Ref MiVpc
    Export:
      Name: red-vpc-id        # único por cuenta + región
```

```yaml
# ---- Stack B (app) ----
Resources:
  MiSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !ImportValue red-vpc-id
```

**Características:**
- Acople **fuerte**: mientras B importe, **no puedes borrar ni cambiar** ese
  output en A. (Es un candado de seguridad, bueno para infra base.)
- `Export.Name` único por cuenta y región.
- ❌ No funciona entre cuentas ni entre regiones.

### En Terraform no hay equivalente directo

Terraform no tiene "exports bloqueados". Lo más parecido a cruzar states es leer
el state remoto de otra configuración:

```hcl
data "terraform_remote_state" "red" {
  backend = "s3"
  config = {
    bucket = "mi-tfstate"
    key    = "red/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_subnet" "app" {
  vpc_id = data.terraform_remote_state.red.outputs.vpc_id
}
```

Diferencia clave: `terraform_remote_state` **lee**, pero **no bloquea** al
productor. No existe el "candado" del Export de CloudFormation.

---

## 4. Entre stacks independientes — Opción B: SSM Parameter Store

El stack productor **escribe** un parámetro; el consumidor lo **lee**.

```yaml
# ---- Stack A (red) ----
Resources:
  VpcIdParam:
    Type: AWS::SSM::Parameter
    Properties:
      Name: /red/vpc-id
      Type: String
      Value: !Ref MiVpc
```

```yaml
# ---- Stack B (app) ----  — forma 1: parámetro de plantilla
Parameters:
  VpcId:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /red/vpc-id
Resources:
  MiSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VpcId
```

```yaml
# ---- Stack B (app) ----  — forma 2: dynamic reference inline
Resources:
  MiSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: "{{resolve:ssm:/red/vpc-id}}"
```

**Características:**
- Acople **suelto**: puedes actualizar/borrar A sin bloqueos.
- ✅ Cross-account y cross-region (con permisos).
- Versionado e historial del valor.
- ❌ No hay dependencia automática: si A cambia el valor, B no se entera hasta
  que lo **redespliegues**.

### Equivalente en Terraform: `data source`

Leer el valor directamente desde SSM (ambas herramientas comparten el mismo SSM):

```hcl
data "aws_ssm_parameter" "vpc_id" {
  name = "/red/vpc-id"
}

resource "aws_subnet" "app" {
  vpc_id = data.aws_ssm_parameter.vpc_id.value
}
```

SSM es, de hecho, un punto de encuentro común: un stack de CloudFormation puede
escribir el parámetro y una config de Terraform leerlo, o al revés.

---

## 5. Cuándo se resuelve el valor (timing)

| | Dentro de la unidad | Entre unidades |
|---|---|---|
| **CloudFormation** | en el `deploy`, por orden de dependencias (nested) | Export: al desplegar B; SSM: al desplegar B |
| **Terraform** | en el `apply`, por orden del grafo (DAG) | `remote_state`/`data`: al `apply` de la config consumidora |

En Terraform, durante `plan`, un valor que aún no existe (lo asigna AWS al crear
el recurso) se muestra como **`(known after apply)`** y se concreta en el
`apply`, creando primero el productor y luego inyectándolo en el consumidor.

En CloudFormation el equivalente es el **orden de dependencias**: los nested
stacks o los recursos que dependen de un `!GetAtt`/`!Ref` se crean después del
recurso del que dependen.

---

## 6. Tabla de decisión (CloudFormation)

| Factor | Export / ImportValue | SSM Parameter Store |
|--------|----------------------|---------------------|
| Acople | **Fuerte** (bloquea al productor) | **Suelto** |
| Cross-account / cross-region | ❌ No | ✅ Sí |
| Dependencia automática | ✅ Sí | ❌ No |
| Cambiar el valor estando en uso | ❌ Bloqueado | ✅ Libre |
| Versionado / historial | ❌ No | ✅ Sí |
| Protección contra borrado accidental | ✅ Fuerte | ⚠️ Débil |

**Regla mental:**
- "Esto no debería poder borrarse mientras alguien lo use" (VPC, KMS, rol base,
  misma cuenta/región) → **Export / Import**.
- "Esto va a cambiar" o "cruza cuentas/regiones" o "hay muchos stacks
  evolucionando" → **SSM**.

A escala, muchos equipos usan **SSM para casi todo** para evitar el bloqueo de
los exports, aceptando a cambio tener que redesplegar el consumidor cuando el
valor cambia.

---

## 7. El gotcha que más duele

El dolor nº1 de Export/Import: una vez que un stack **importa** tu export, **no
puedes actualizar ni borrar** el stack que lo exporta hasta romper esa
importación. Con decenas de stacks esto genera bloqueos y dependencias de
despliegue molestas. Por eso:

- Pocos stacks, infra base estable → **Export/Import** está bien y es más simple.
- Muchos stacks, cambios frecuentes, multi-cuenta → **SSM** casi siempre gana.

---

## 8. Resumen de equivalencias CloudFormation ↔ Terraform

| Caso | CloudFormation | Terraform |
|------|----------------|-----------|
| Dentro de una unidad | nested stacks + `!GetAtt X.Outputs.Y` | módulos + `module.x.y` |
| Entre unidades, acople fuerte + candado | `Export` / `!ImportValue` | *(no hay equivalente con bloqueo)* |
| Entre unidades, acople suelto | `AWS::SSM::Parameter` + `{{resolve:ssm:...}}` | `data "aws_ssm_parameter"` / `terraform_remote_state` |
| Momento de resolución | `deploy`, por dependencias | `apply`, por el grafo (`known after apply`) |

**Conclusión:** ninguna de las dos "es más compleja". Ambas convergen a escala en
lo mismo: **muchas unidades independientes + un contrato explícito para cruzar
fronteras**. Cambia el *default*: Terraform empieza todo junto (fácil al inicio,
tú decides cuándo partir); CloudFormation te empuja antes a partir en stacks (más
fricción al inicio, pero el desacople viene de fábrica).
