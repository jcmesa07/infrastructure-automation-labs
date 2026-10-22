# Lab: Nested Stacks (módulos reutilizables + varios padres)

Dos ejemplos de nested stacks que **comparten los mismos módulos hijos**. Muestra
el patrón de "librería de módulos" (como los módulos de Terraform): los hijos
reutilizables viven en `modules/`, y cada padre los compone a su manera.

## Estructura

```
nested-stacks/
├── parent-buckets.yml       # PADRE 1: crea 3 buckets reutilizando bucket.yml
├── parent-network.yml       # PADRE 2: VPC + IGW + rutas, cableando outputs
│
├── modules/                 # HIJOS reutilizables (la "librería")
│   ├── bucket.yml           #   un bucket S3 parametrizado
│   ├── vpc.yml              #   una VPC               → Output VpcId
│   ├── internet-gateway.yml #   IGW + attachment      → recibe VpcId, Output IgwId
│   ├── route-table.yml      #   tabla de rutas        → recibe VpcId + IgwId
│   ├── subnet.yml           #   subred pública        → recibe VpcId + RouteTableId
│   └── security-group.yml   #   SG puertos 22 y 80    → recibe VpcId
│
├── params/                  # Valores por entorno (≈ .tfvars), por padre
│   ├── buckets-dev.json
│   ├── buckets-qa.json
│   ├── buckets-prod.json
│   └── network-dev.json
└── README.md
```

Un padre = un stack que despliegas. Los hijos de `modules/` no se despliegan
solos: los invoca el padre con `TemplateURL: modules/<hijo>.yml`.

---

## Padre 1 — `parent-buckets.yml` (reutilizar el MISMO módulo N veces)

Instancia `modules/bucket.yml` tres veces (datos, logs, backups) con parámetros
distintos. Enseña la **reutilización**: un módulo, varias instancias.

```yaml
BucketDatos:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: modules/bucket.yml
    Parameters: { NombreBucket: ..., Versionado: Enabled }
```

---

## Padre 2 — `parent-network.yml` (COMPARTIR outputs entre hijos)

Compone tres módulos distintos y **cablea los outputs de unos como parámetros de
otros**:

```
vpc ──VpcId──────────▶ internet-gateway, route-table, subnet, security-group
internet-gateway ──IgwId──▶ route-table
route-table ──RouteTableId──▶ subnet
```

La red queda completa: **VPC → IGW → route-table → subnet pública** (con salida a
internet) **+ security-group** con los puertos **22 (SSH)** y **80 (HTTP)** abiertos.

```yaml
Igw:
  Properties:
    Parameters:
      VpcId: !GetAtt Vpc.Outputs.VpcId     # output de un hijo -> input de otro
RouteTable:
  Properties:
    Parameters:
      VpcId: !GetAtt Vpc.Outputs.VpcId
      IgwId: !GetAtt Igw.Outputs.IgwId
```

`!GetAtt <Hijo>.Outputs.<X>` pasa el valor **y** crea la dependencia, así que el
orden (`vpc → internet-gateway → route-table`) se calcula solo. No hace falta
Export/Import ni SSM: eso es solo para stacks raíz independientes (ver
`../doc/COMPARTIR-DATOS-ENTRE-STACKS.md`).

### Equivalente en Terraform

```hcl
module "vpc"          { source = "./modules/vpc" }
module "igw"          { source = "./modules/internet-gateway"; vpc_id = module.vpc.vpc_id }
module "route_table"  { source = "./modules/route-table";      vpc_id = module.vpc.vpc_id
                                                               igw_id = module.igw.igw_id }
```

---

## Despliegue (igual para ambos padres)

Nested stacks exige `package` (sube los módulos a S3) antes del `deploy`.

```bash
# bucket de staging para los artefactos (una sola vez)
aws s3 mb s3://MI-BUCKET-ARTEFACTOS-cfn
```

### Padre de buckets

```bash
aws cloudformation package \
  --template-file parent-buckets.yml \
  --s3-bucket MI-BUCKET-ARTEFACTOS-cfn \
  --output-template-file parent-buckets.packaged.yml

aws cloudformation deploy \
  --template-file parent-buckets.packaged.yml \
  --stack-name poc-buckets-dev \
  --parameter-overrides file://params/buckets-dev.json
```

### Padre de red

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

> Cambiar de entorno = cambiar solo el archivo de params y el `--stack-name`
> (ej. `params/buckets-prod.json` + `--stack-name poc-buckets-prod`).

---

## Ver resultados y borrar

```bash
# Outputs del padre (vienen de los hijos)
aws cloudformation describe-stacks --stack-name poc-network-dev \
  --query "Stacks[0].Outputs" --output table

# Borrar el padre borra sus hijos (en orden inverso)
aws cloudformation delete-stack --stack-name poc-network-dev
```

> ⚠️ Los buckets usan `DeletionPolicy: Retain` y **sobreviven** al borrado del
> stack (es un recurso con datos). La red no retiene nada, se elimina completa.

---

## Protecciones del stack (Termination Protection y Stack Policy)

Dos candados a nivel de stack. **No van en la plantilla ni en `deploy`**: se
aplican con comandos aparte (detalle en `../doc/LIFECYCLE-Y-PROTECCIONES.md`).

### Termination Protection — candado contra borrar el stack entero

Mientras está activa, `delete-stack` falla.

```bash
# activar
aws cloudformation update-termination-protection \
  --enable-termination-protection --stack-name poc-network-dev

# desactivar (para poder borrar después)
aws cloudformation update-termination-protection \
  --no-enable-termination-protection --stack-name poc-network-dev
```

### Stack Policy — proteger un recurso concreto durante updates

El archivo `stack-policy.json` de este lab **deniega reemplazar o borrar la VPC**
(`LogicalResourceId/Vpc`) en cualquier update, y permite el resto. Así, un cambio
que recrearía la VPC se bloquea en vez de tumbar la red.

```bash
# aplicar la stack policy a un stack existente
aws cloudformation set-stack-policy \
  --stack-name poc-network-dev \
  --stack-policy-body file://stack-policy.json

# o al crear
aws cloudformation create-stack ... --stack-policy-body file://stack-policy.json
```

> Si alguna vez necesitas hacer ese cambio bloqueado a propósito, se usa
> `--stack-policy-during-update-body` solo para ese update puntual.

| Protección | Alcance | Mecanismo |
|------------|---------|-----------|
| `DeletionPolicy` (en el módulo) | un recurso | atributo en la plantilla |
| Stack Policy | recursos concretos del stack, en updates | `set-stack-policy` + JSON |
| Termination Protection | el stack completo, al borrar | `update-termination-protection` |

---

## Idea para llevarlo a la realidad

`modules/` es tu **librería compartida**: añadir un padre nuevo es solo componer
los hijos que ya tienes. En proyectos grandes, `modules/` se versiona aparte y se
publica en un bucket de S3 común a los equipos.
