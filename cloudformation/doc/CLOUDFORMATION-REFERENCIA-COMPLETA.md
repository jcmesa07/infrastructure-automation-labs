# AWS CloudFormation — Referencia Completa + Ejercicios

> Guía de estudio para la sección **AWS CloudFormation** de la prueba Bancolombia (4 preguntas, la de mayor peso).
> Cubre toda la anatomía de un template, secciones, atributos de recursos, funciones intrínsecas, pseudo-parámetros y políticas.

---

## Índice
1. [Qué es CloudFormation](#1-qué-es-cloudformation)
2. [Anatomía completa de un template](#2-anatomía-completa-de-un-template)
3. [Las 9 secciones del template](#3-las-9-secciones-del-template)
4. [Resources a fondo (lo más importante)](#4-resources-a-fondo)
5. [Atributos de un recurso](#5-atributos-de-un-recurso)
6. [DeletionPolicy y UpdateReplacePolicy](#6-deletionpolicy-y-updatereplacepolicy)
7. [Parameters a fondo](#7-parameters-a-fondo)
8. [Funciones intrínsecas (todas)](#8-funciones-intrínsecas)
9. [Pseudo-parámetros](#9-pseudo-parámetros)
10. [Conditions](#10-conditions)
11. [Mappings](#11-mappings)
12. [Outputs y exportar valores](#12-outputs-y-exportar-valores)
13. [Comandos del ciclo de vida](#13-comandos-del-ciclo-de-vida)
14. [Buenas prácticas y errores típicos](#14-buenas-prácticas-y-errores-típicos)
15. [Ejercicios](#15-ejercicios)
16. [Soluciones](#16-soluciones)

---

## 1. Qué es CloudFormation

CloudFormation (CFN) es el servicio de **infraestructura como código (IaC)** de AWS. Describes tus recursos en un **template** (YAML o JSON) y CFN los crea, actualiza y borra como una unidad llamada **stack**.

Conceptos clave:
- **Template** → el archivo YAML/JSON con la descripción de la infraestructura.
- **Stack** → la instancia desplegada de un template (todos los recursos juntos).
- **Change set** → la "preview" de los cambios antes de aplicarlos (como `terraform plan`).
- **Drift** → cuando alguien cambia un recurso por fuera de CFN y el estado real difiere del template.
- **Nested stacks** → stacks dentro de stacks (modularidad).
- **StackSets** → desplegar el mismo stack en varias cuentas/regiones.

Ventaja: es **declarativo** (dices el *qué*, no el *cómo*) e **idempotente** (aplicar el mismo template no duplica recursos).

---

## 2. Anatomía completa de un template

Un template YAML tiene esta estructura. **La única sección obligatoria es `Resources`.**

```yaml
AWSTemplateFormatVersion: "2010-09-09"   # opcional (única versión válida)
Description: "Texto describiendo el template"   # opcional

Metadata: {}        # opcional - datos extra sobre el template

Parameters: {}      # opcional - valores de entrada al desplegar

Rules: {}           # opcional - validaciones sobre los parámetros

Mappings: {}        # opcional - tablas de búsqueda clave-valor

Conditions: {}      # opcional - condiciones booleanas

Transform: []       # opcional - macros (ej. AWS::Serverless para SAM)

Resources: {}       # OBLIGATORIO - los recursos de AWS a crear

Outputs: {}         # opcional - valores de salida del stack
```

---

## 3. Las 9 secciones del template

| Sección | Obligatoria | Para qué sirve |
|---|---|---|
| `AWSTemplateFormatVersion` | No | Versión del formato. Solo existe `"2010-09-09"`. |
| `Description` | No | Texto descriptivo (máx 1024 bytes). Debe ir **justo después** de la versión. |
| `Metadata` | No | Datos adicionales (ej. config de la consola, `AWS::CloudFormation::Interface`). |
| `Parameters` | No | Valores de entrada que das al desplegar (hace el template reutilizable). |
| `Rules` | No | Validan combinaciones de parámetros (ej. "si env=prod, el tipo debe ser grande"). |
| `Mappings` | No | Tablas fijas clave→valor (ej. AMI por región). Se consultan con `Fn::FindInMap`. |
| `Conditions` | No | Expresiones booleanas que deciden si crear un recurso o asignar un valor. |
| `Transform` | No | Aplica macros. `AWS::Serverless-2016-10-31` convierte el template en SAM. |
| `Resources` | **SÍ** | Declara los recursos de AWS. Es el corazón del template. |
| `Outputs` | No | Devuelve valores (ej. la URL de un bucket) y permite exportarlos a otros stacks. |

> Pregunta típica de examen: **¿cuál es la única sección obligatoria?** → `Resources`.

---

## 4. Resources a fondo

Es la sección más importante. Sintaxis general:

```yaml
Resources:
  NombreLogico:                     # ID lógico que TÚ inventas (alfanumérico, único)
    Type: AWS::Servicio::Recurso    # QUÉ recurso creas. Formato: AWS::Service::Resource
    Properties:                     # configuración específica de ese recurso
      Clave1: valor1
      Clave2: valor2
```

- **ID lógico (Logical ID)**: nombre interno del template. Solo alfanumérico (`MiBucket`, `WebServer01`). Con él te refieres al recurso usando `Ref` o `Fn::GetAtt`. NO es el nombre real del recurso en AWS.
- **`Type`**: identifica el tipo de recurso. Formato `AWS::Servicio::Recurso`. Ejemplos:
  - `AWS::S3::Bucket`
  - `AWS::EC2::Instance`
  - `AWS::IAM::Role`
  - `AWS::Lambda::Function`
  - `AWS::SQS::Queue`
  - `AWS::SecretsManager::Secret`
- **`Properties`**: la configuración. Cada `Type` tiene sus propias propiedades (obligatorias y opcionales). Algunas pueden ir vacías si el recurso tiene todo por defecto (como el ECR de tu ejemplo).

> **Physical ID vs Logical ID**: el *lógico* es el nombre en el template; el *físico* es el nombre/ARN real que AWS asigna tras crearlo. Si no defines un nombre (ej. `BucketName`), CFN genera uno único automáticamente.

---

## 5. Atributos de un recurso

Además de `Type` y `Properties`, un recurso acepta **atributos** (van al mismo nivel que `Type`):

```yaml
Resources:
  MiRecurso:
    Type: AWS::S3::Bucket
    Properties: {}
    DependsOn: OtroRecurso          # fuerza orden de creación
    Condition: EsProduccion         # solo se crea si la condición es true
    DeletionPolicy: Retain          # qué hacer al borrar el stack
    UpdateReplacePolicy: Retain     # qué hacer si un update reemplaza el recurso
    Metadata: {}                    # datos extra del recurso
    CreationPolicy: {}              # espera señales antes de marcar "creado" (EC2/ASG)
    UpdatePolicy: {}                # cómo actualizar (rolling update en ASG)
```

| Atributo | Para qué |
|---|---|
| `DependsOn` | Define orden explícito: crea este recurso **después** del indicado. CFN resuelve dependencias solo, pero a veces hay que forzarlo. |
| `Condition` | Asocia el recurso a una condición de la sección `Conditions`. Si es false, no se crea. |
| `DeletionPolicy` | Qué pasa al eliminar el stack (ver sección 6). |
| `UpdateReplacePolicy` | Qué pasa si un update **reemplaza** el recurso (lo recrea). |
| `Metadata` | Información arbitraria asociada al recurso. |
| `CreationPolicy` | CFN espera N señales de éxito antes de dar el recurso por creado (típico con `cfn-signal` en EC2/AutoScaling). |
| `UpdatePolicy` | Controla cómo se actualiza un AutoScalingGroup, Lambda alias o similar (rolling updates). |

---

## 6. DeletionPolicy y UpdateReplacePolicy

Pregunta frecuente. Controlan qué pasa con los datos al borrar/reemplazar:

**`DeletionPolicy`** — al eliminar el stack:
- `Delete` (default en la mayoría) → borra el recurso.
- `Retain` → conserva el recurso (no lo borra). Útil para datos críticos.
- `Snapshot` → crea un snapshot antes de borrar (solo recursos que soportan snapshot: RDS, EBS, ElastiCache, Redshift...).
- `RetainExceptOnCreate` → conserva, salvo si el recurso se creó en un stack que luego hizo rollback.

**`UpdateReplacePolicy`** — cuando un update obliga a **recrear** el recurso:
- `Delete` (default), `Retain`, `Snapshot`.

> Clave: para bases de datos y buckets con datos, pon `DeletionPolicy: Retain` o `Snapshot` para no perder información al borrar el stack.

---

## 7. Parameters a fondo

Hacen el template reutilizable. Se pasan al desplegar.

```yaml
Parameters:
  NombreParametro:
    Type: String                   # tipo (obligatorio)
    Default: valor                 # valor por defecto
    Description: "qué es"
    AllowedValues: [dev, qa, prod] # lista cerrada de valores válidos
    AllowedPattern: "^[a-z]+$"     # regex de validación (solo String)
    MinLength: 1
    MaxLength: 20
    MinValue: 1                    # para Number
    MaxValue: 100
    NoEcho: true                   # oculta el valor en consola/logs (passwords)
    ConstraintDescription: "mensaje si falla la validación"
```

### Tipos de parámetro
- **Básicos**: `String`, `Number`, `List<Number>`, `CommaDelimitedList`.
- **Específicos de AWS** (CFN valida que existan en tu cuenta):
  - `AWS::EC2::KeyPair::KeyName`
  - `AWS::EC2::VPC::Id`
  - `AWS::EC2::Subnet::Id`
  - `AWS::EC2::SecurityGroup::Id`
  - `AWS::EC2::Image::Id`
  - `List<AWS::EC2::Subnet::Id>`
- **SSM**: `AWS::SSM::Parameter::Value<String>` → toma el valor de Parameter Store.

Se usan con `!Ref NombreParametro`.

> Pasar parámetros al desplegar:
> `aws cloudformation deploy --template-file t.yml --stack-name s --parameter-overrides Env=prod InstanceType=t3.micro`

---

## 8. Funciones intrínsecas

Funciones que CFN resuelve en tiempo de despliegue. Tienen **forma larga** (`Fn::Nombre`) y **forma corta** (`!Nombre`, solo en YAML).

| Función | Forma corta | Qué hace |
|---|---|---|
| `Ref` | `!Ref` | Devuelve el valor de un parámetro, o el ID físico de un recurso. |
| `Fn::GetAtt` | `!GetAtt` | Devuelve un atributo de un recurso (ej. ARN, DNS). |
| `Fn::Sub` | `!Sub` | Sustituye variables `${}` dentro de un string. |
| `Fn::Join` | `!Join` | Une una lista con un separador. |
| `Fn::Split` | `!Split` | Parte un string en lista según un separador. |
| `Fn::Select` | `!Select` | Toma un elemento de una lista por índice. |
| `Fn::FindInMap` | `!FindInMap` | Busca un valor en `Mappings`. |
| `Fn::GetAZs` | `!GetAZs` | Lista las zonas de disponibilidad de una región. |
| `Fn::ImportValue` | `!ImportValue` | Importa un valor exportado por otro stack. |
| `Fn::Base64` | `!Base64` | Codifica a Base64 (típico en EC2 UserData). |
| `Fn::Cidr` | `!Cidr` | Genera bloques CIDR de subred. |
| `Fn::If` | `!If` | Devuelve un valor u otro según una condición. |
| `Fn::And` / `Or` / `Not` / `Equals` | `!And` etc. | Lógica booleana (en `Conditions`). |
| `Fn::Length` | — | Longitud de una lista (requiere `AWS::LanguageExtensions`). |
| `Fn::ToJsonString` | — | Convierte un objeto a JSON string (con language extensions). |
| `Fn::Transform` | — | Invoca una macro. |

### Ejemplos prácticos

```yaml
# Ref: ID físico de un recurso o valor de un parámetro
BucketName: !Ref MiParametroNombre
RoleArn: !Ref MiRol            # para un recurso, Ref suele dar su nombre/ID

# GetAtt: un atributo concreto (ARN, DNS, etc.)
Arn: !GetAtt MiRol.Arn
Endpoint: !GetAtt MiBaseDeDatos.Endpoint.Address

# Sub: interpolar variables (más legible que Join)
QueueUrl: !Sub "https://sqs.${AWS::Region}.amazonaws.com/${AWS::AccountId}/mi-cola"
Nombre: !Sub "${Env}-mi-recurso"

# Join: unir con separador
Arn: !Join [":", ["arn:aws:s3", "", "", !Ref MiBucket]]

# GetAtt + Select + GetAZs: primera AZ de la región
AvailabilityZone: !Select [0, !GetAZs ""]

# FindInMap: buscar en una tabla
ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]

# ImportValue: traer un export de otro stack
VpcId: !ImportValue red-compartida-VpcId

# If: valor condicional
InstanceType: !If [EsProd, t3.large, t3.micro]
```

> **Regla de oro:** `!Ref` y `!GetAtt` son las dos que más caen. `Ref` da el "identificador principal" del recurso; `GetAtt` da un **atributo específico** (como el ARN). Qué devuelve exactamente cada `Ref`/`GetAtt` depende del tipo de recurso (lo dice la doc de cada `Type`).

> **Ojo con anidar formas cortas:** no puedes poner `!GetAtt` dentro de `!Sub` en forma corta directa. Usa `${Recurso.Atributo}` dentro del `!Sub` o pasa a forma larga.

---

## 9. Pseudo-parámetros

Variables que AWS provee automáticamente (no las declaras). Se usan con `!Ref` o dentro de `!Sub`:

| Pseudo-parámetro | Devuelve |
|---|---|
| `AWS::AccountId` | El ID de la cuenta (12 dígitos). |
| `AWS::Region` | La región del stack (ej. `us-east-2`). |
| `AWS::StackName` | El nombre del stack. |
| `AWS::StackId` | El ARN completo del stack. |
| `AWS::Partition` | La partición (`aws`, `aws-cn`, `aws-us-gov`). |
| `AWS::URLSuffix` | Sufijo del dominio (`amazonaws.com`). |
| `AWS::NotificationARNs` | Lista de ARNs de notificación del stack. |
| `AWS::NoValue` | "Sin valor": si una propiedad resuelve a esto, se **elimina**. Útil con `!If`. |

```yaml
# Construir un ARN de forma portable (sirve en cualquier región/partición)
Resource: !Sub "arn:${AWS::Partition}:s3:::${AWS::StackName}-datos"

# Quitar una propiedad condicionalmente
KmsKeyId: !If [UsaKms, !Ref MiKey, !Ref "AWS::NoValue"]
```

---

## 10. Conditions

Permiten crear recursos o asignar valores según condiciones. Dos pasos: **definir** la condición y **usarla**.

```yaml
Parameters:
  Env:
    Type: String
    AllowedValues: [dev, prod]

Conditions:
  EsProd: !Equals [!Ref Env, prod]
  NoEsProd: !Not [!Equals [!Ref Env, prod]]
  EsProdEnUsEast1: !And
    - !Equals [!Ref Env, prod]
    - !Equals [!Ref "AWS::Region", us-east-1]

Resources:
  BackupBucket:
    Type: AWS::S3::Bucket
    Condition: EsProd            # solo se crea en prod
  MiInstancia:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !If [EsProd, t3.large, t3.micro]
```

Operadores: `Fn::Equals`, `Fn::And`, `Fn::Or`, `Fn::Not`, y `Fn::If` para elegir valores.

---

## 11. Mappings

Tablas fijas de dos niveles (clave → clave → valor). Clásico: AMI por región.

```yaml
Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0abc111
    us-east-2:
      AMI: ami-0def222
Resources:
  Server:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", AMI]
```

`!FindInMap [NombreMapa, ClaveNivel1, ClaveNivel2]`. Los valores son fijos (no se calculan en runtime).

---

## 12. Outputs y exportar valores

Devuelven información del stack y permiten compartir valores entre stacks.

```yaml
Outputs:
  BucketArn:
    Description: "ARN del bucket"
    Value: !GetAtt MiBucket.Arn
  VpcIdExport:
    Value: !Ref MiVpc
    Export:                       # exporta para que otros stacks lo importen
      Name: red-compartida-VpcId
```

- `Value` → el valor a devolver (puede usar `!Ref`, `!GetAtt`, etc.).
- `Export.Name` → nombre **único por región** para que otro stack lo use con `!ImportValue`.
- No puedes borrar un stack cuyo export esté siendo importado por otro.

---

## 13. Comandos del ciclo de vida

(Ya los tienes en `COMANDOS-CICLO-DE-VIDA.md`. Resumen rápido.)

```bash
aws cloudformation validate-template --template-body file://t.yml   # validar
aws cloudformation deploy --template-file t.yml --stack-name s       # crear/actualizar
aws cloudformation deploy ... --no-execute-changeset                 # preview (plan)
aws cloudformation describe-stacks --stack-name s                    # estado
aws cloudformation describe-stack-events --stack-name s              # eventos (debug)
aws cloudformation delete-stack --stack-name s                       # borrar
```

Con parámetros y capacidades IAM:
```bash
aws cloudformation deploy \
  --template-file t.yml \
  --stack-name s \
  --parameter-overrides Env=prod \
  --capabilities CAPABILITY_NAMED_IAM
```

> **`--capabilities`**: obligatorio cuando el template crea **recursos IAM**. 
> - `CAPABILITY_IAM` → recursos IAM sin nombre personalizado.
> - `CAPABILITY_NAMED_IAM` → recursos IAM **con nombre propio** (ej. `RoleName`).
> - `CAPABILITY_AUTO_EXPAND` → templates con macros/transforms.
> Es una pregunta de examen muy común.

---

## 14. Buenas prácticas y errores típicos

1. **`Resources` es obligatoria**; todo lo demás es opcional.
2. **No hardcodees ARNs ni cuenta/región**: usa `!Sub` con `${AWS::AccountId}`, `${AWS::Region}`, `${AWS::Partition}`.
3. **IAM necesita `--capabilities`** (`CAPABILITY_IAM` o `CAPABILITY_NAMED_IAM`).
4. **`DeletionPolicy: Retain`/`Snapshot`** para datos que no quieres perder.
5. **`!Ref` vs `!GetAtt`**: Ref = identificador principal; GetAtt = atributo concreto (ARN, DNS...).
6. **`DependsOn`** solo cuando CFN no detecta la dependencia solo (normalmente no hace falta si usas `!Ref`/`!GetAtt`).
7. **`NoEcho: true`** en parámetros sensibles (passwords).
8. Si pones un **nombre fijo** (ej. `RoleName`, `BucketName`) pierdes flexibilidad: no puedes tener dos stacks iguales y algunos updates obligan a recrear. A veces conviene dejar que CFN autogenere el nombre.
9. **Rollback**: si un recurso falla al crear, CFN revierte todo el stack por defecto.
10. **Drift**: no edites recursos por consola si los gestiona CFN; genera drift.

---

## 15. Ejercicios

> Responde sin mirar. Soluciones en la sección 16.

### Conceptuales
1. ¿Cuál es la única sección obligatoria de un template?
2. ¿Diferencia entre `!Ref` y `!GetAtt`?
3. ¿Qué hace `DeletionPolicy: Retain`?
4. ¿Para qué sirve `--capabilities CAPABILITY_NAMED_IAM`?
5. ¿Qué devuelve el pseudo-parámetro `AWS::NoValue` y para qué se usa?
6. ¿Diferencia entre Logical ID y Physical ID?
7. ¿Qué función usas para interpolar variables en un string? Da un ejemplo.
8. ¿Cómo compartes un valor de un stack a otro?
9. ¿Qué hace `Transform: AWS::Serverless-2016-10-31`?
10. ¿Cuándo usarías `DependsOn` si CFN resuelve dependencias solo?
11. ¿Qué función elige la primera zona de disponibilidad de una región?
12. ¿Qué sección usas para una tabla de AMIs por región y con qué función la consultas?

### Escribe de memoria
13. Escribe un parámetro `Env` que solo acepte `dev`, `qa` o `prod`, con default `dev`.
14. Escribe una `Condition` llamada `EsProd` que sea true si `Env` es `prod`.
15. Escribe un `Output` que devuelva el ARN de un recurso llamado `MiCola` y lo exporte como `MiColaArn`.
16. Construye el ARN de una cola SQS usando `!Sub` y pseudo-parámetros.

### Arregla el template (tipo examen)
17.
```yaml
Resources:
  MiRol:
    Type: AWS::IAM::Role
    Properties:
      RoleName: mi-rol
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
Outputs:
  Arn:
    Value: !Ref MiRol
```
Hay dos problemas: (a) algo que falta al desplegar por el tipo de recurso, y (b) el Output no devuelve el ARN. Encuéntralos.

18.
```yaml
Resources:
  MiBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "${AWS::AccountId}-datos"
  MiCola:
    Type: AWS::SQS::Queue
    DependsOn: MiBucket
    Properties:
      QueueName: mi-cola
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt MiDLQ.Arn
        maxReceiveCount: 3
```
¿Qué recurso falta declarar y es ese `DependsOn` necesario?

---

## 16. Soluciones

> Intenta responder primero sin mirar esta sección.

**Conceptuales**
1. `Resources`.
2. `!Ref` devuelve el identificador principal del recurso (o el valor de un parámetro); `!GetAtt` devuelve un **atributo específico** como el ARN, DNS, endpoint, etc.
3. Al borrar el stack, **conserva** el recurso en vez de eliminarlo (no pierdes los datos).
4. Autoriza a CFN a crear recursos IAM **con nombre personalizado** (ej. `RoleName`). Sin esta capability, el deploy falla.
5. "Sin valor": si una propiedad resuelve a `AWS::NoValue`, esa propiedad **se elimina** del recurso. Se usa con `!If` para incluir/omitir propiedades.
6. Logical ID = nombre del recurso dentro del template (lo inventas tú). Physical ID = nombre/ARN real que AWS asigna tras crearlo.
7. `Fn::Sub` (`!Sub`). Ej: `!Sub "${Env}-mi-bucket"`.
8. Con `Outputs` + `Export.Name` en el stack origen, y `!ImportValue nombre` en el stack destino.
9. Convierte el template en **AWS SAM** (Serverless Application Model): habilita tipos como `AWS::Serverless::Function` que se expanden a recursos normales.
10. Cuando hay una dependencia que CFN **no detecta** por no haber un `!Ref`/`!GetAtt` entre los recursos (ej. una política IAM que debe existir antes de que algo la use, sin referencia directa).
11. `!Select [0, !GetAZs ""]`.
12. La sección `Mappings`, y se consulta con `!FindInMap [NombreMapa, Clave1, Clave2]`.

**Escribe de memoria**
13.
```yaml
Parameters:
  Env:
    Type: String
    Default: dev
    AllowedValues: [dev, qa, prod]
```
14.
```yaml
Conditions:
  EsProd: !Equals [!Ref Env, prod]
```
15.
```yaml
Outputs:
  MiColaArn:
    Value: !GetAtt MiCola.Arn
    Export:
      Name: MiColaArn
```
16.
```yaml
!Sub "arn:${AWS::Partition}:sqs:${AWS::Region}:${AWS::AccountId}:mi-cola"
```

**Arregla el template**
17. (a) Como crea un `AWS::IAM::Role` **con nombre** (`RoleName: mi-rol`), el deploy necesita `--capabilities CAPABILITY_NAMED_IAM`. (b) El Output usa `!Ref MiRol`, que para un rol devuelve el **nombre**, no el ARN; debe ser `!GetAtt MiRol.Arn`.
```yaml
Outputs:
  Arn:
    Value: !GetAtt MiRol.Arn
```
18. Falta declarar el recurso `MiDLQ` (otra `AWS::SQS::Queue`) que se referencia en `deadLetterTargetArn`. El `DependsOn: MiBucket` **no es necesario**: no hay relación real entre la cola y el bucket, y además CFN no necesita ese orden. En cambio sí hay dependencia implícita con `MiDLQ` por el `!GetAtt` (CFN la resuelve sola, sin `DependsOn`).
