# AWS CloudFormation — Cuestionario y Ejercicios (con Respuestas)

> Práctica tipo examen para la sección **AWS CloudFormation** (4 preguntas, mayor peso).
> Mezcla opción múltiple, preguntas cortas, "completa el template", "escribe de memoria"
> y "arregla el template" (formato hands-on de HackerRank).
> Las respuestas están al final de cada sección.

---

## Índice
- [A. Opción múltiple (MCQ)](#a-opción-múltiple)
- [B. Preguntas cortas](#b-preguntas-cortas)
- [C. Completa el template](#c-completa-el-template)
- [D. Escribe de memoria](#d-escribe-de-memoria)
- [E. Arregla el template](#e-arregla-el-template)
- [Respuestas A](#respuestas-a)
- [Respuestas B](#respuestas-b)
- [Respuestas C](#respuestas-c)
- [Respuestas D](#respuestas-d)
- [Respuestas E](#respuestas-e)

---

## A. Opción múltiple

**A1.** ¿Cuál es la única sección obligatoria de un template de CloudFormation?
- a) `Parameters`
- b) `Resources`
- c) `Outputs`
- d) `AWSTemplateFormatVersion`

**A2.** Para un recurso `AWS::IAM::Role`, `!Ref MiRol` devuelve:
- a) El ARN del rol
- b) El nombre del rol
- c) El ID de la política
- d) El trust policy

**A3.** ¿Qué capability necesitas para desplegar un template que crea un rol IAM **con nombre propio** (`RoleName`)?
- a) `CAPABILITY_IAM`
- b) `CAPABILITY_NAMED_IAM`
- c) `CAPABILITY_AUTO_EXPAND`
- d) Ninguna

**A4.** ¿Qué función usarías para interpolar variables dentro de un string?
- a) `Fn::Join`
- b) `Fn::Select`
- c) `Fn::Sub`
- d) `Fn::FindInMap`

**A5.** En un change set, `Replacement: True` significa que:
- a) El recurso se modifica en caliente
- b) El recurso se recrea (se reemplaza)
- c) El recurso se exporta
- d) No habrá cambios

**A6.** ¿Qué `DeletionPolicy` conserva el recurso al borrar el stack?
- a) `Delete`
- b) `Retain`
- c) `Remove`
- d) `Keep`

**A7.** ¿Qué pseudo-parámetro, al asignarse a una propiedad, hace que esa propiedad se elimine?
- a) `AWS::Region`
- b) `AWS::StackName`
- c) `AWS::NoValue`
- d) `AWS::AccountId`

**A8.** ¿Qué `Transform` habilita `Fn::ForEach`?
- a) `AWS::Serverless-2016-10-31`
- b) `AWS::Include`
- c) `AWS::LanguageExtensions`
- d) `AWS::CodeDeploy`

**A9.** ¿Cómo lees un output de un nested stack desde el stack padre?
- a) `!ImportValue NestedStack.Output`
- b) `!GetAtt NestedStack.Outputs.NombreOutput`
- c) `!Ref NestedStack.NombreOutput`
- d) `!FindInMap [NestedStack, Outputs, Nombre]`

**A10.** Para una cola `AWS::SQS::Queue`, `!Ref MiCola` devuelve:
- a) El ARN de la cola
- b) La URL de la cola
- c) El nombre de la cola
- d) El RedrivePolicy

**A11.** ¿Qué atributo espera señales (`cfn-signal`) antes de marcar un recurso como creado?
- a) `UpdatePolicy`
- b) `DeletionPolicy`
- c) `CreationPolicy`
- d) `DependsOn`

**A12.** ¿Qué sección define tablas fijas clave→valor (ej. AMI por región)?
- a) `Mappings`
- b) `Conditions`
- c) `Metadata`
- d) `Rules`

**A13.** Una dynamic reference para leer un SecureString cifrado de SSM es:
- a) `{{resolve:ssm:/ruta}}`
- b) `{{resolve:ssm-secure:/ruta}}`
- c) `{{resolve:secretsmanager:/ruta}}`
- d) `!Ref /ruta`

**A14.** La detección de drift:
- a) Corrige automáticamente los cambios manuales
- b) Solo reporta diferencias, no corrige
- c) Borra los recursos con drift
- d) Impide editar por consola

**A15.** ¿Qué comando genera la "preview" de cambios sin aplicarlos?
- a) `aws cloudformation deploy`
- b) `aws cloudformation describe-stacks`
- c) `aws cloudformation create-change-set`
- d) `aws cloudformation detect-stack-drift`

---

## B. Preguntas cortas

**B1.** Explica la diferencia entre `!Ref` y `!GetAtt`.

**B2.** ¿Qué diferencia hay entre Logical ID y Physical ID?

**B3.** ¿Cuándo usarías `DependsOn` si CloudFormation resuelve dependencias solo?

**B4.** ¿Qué hace `Fn::Base64` y en qué recurso es típico usarla?

**B5.** ¿Qué diferencia hay entre una inline policy y una managed policy en IAM dentro de CFN?

**B6.** ¿Qué pasa si intentas borrar un stack cuyo `Export` está siendo importado por otro stack?

**B7.** ¿Qué diferencia hay entre `CreationPolicy`, `UpdatePolicy` y `UpdateReplacePolicy`?

**B8.** ¿Para qué sirve un Custom Resource y qué obligación tiene su Lambda?

---

## C. Completa el template

**C1.** Completa para que el `InstanceType` sea `t3.large` en prod y `t3.micro` en otros entornos (ya existe la condición `EsProd`):
```yaml
Properties:
  InstanceType: !If [ _____ , _____ , _____ ]
```

**C2.** Completa el Output para devolver el ARN del rol `MiRol` y exportarlo como `dev-MiRolArn`:
```yaml
Outputs:
  RolArn:
    Value: _______________
    Export:
      Name: _______________
```

**C3.** Completa el `!FindInMap` para obtener la AMI de la región actual desde el mapa `RegionAmiMap` (clave de segundo nivel `AMI`):
```yaml
ImageId: !FindInMap [ _____ , _____ , _____ ]
```

**C4.** Completa el `RedrivePolicy` de una cola SQS para enviar a `MiDLQ` tras 3 fallos:
```yaml
RedrivePolicy:
  deadLetterTargetArn: _______________
  maxReceiveCount: _____
```

**C5.** Completa el ARN portable de un bucket usando `!Sub` y pseudo-parámetros:
```yaml
Resource: !Sub "arn:_______:s3:::${_______}-datos"
```

---

## D. Escribe de memoria

**D1.** Escribe un parámetro `Env` que solo acepte `dev`, `qa`, `prod`, con default `dev` y descripción.

**D2.** Escribe un rol IAM cuya trust policy permita al servicio Lambda asumirlo, con la managed policy `AWSLambdaBasicExecutionRole`.

**D3.** Escribe una `Condition` llamada `EsProd` (true si `Env == prod`) y otra `UsaKms` (true si el parámetro `KmsKeyArn` NO está vacío).

**D4.** Escribe el `UserData` (en Base64) de una EC2 que instale y arranque httpd, usando `!Sub`.

**D5.** Escribe un secreto de Secrets Manager que autogenere una password de 20 caracteres sobre `{"username":"admin"}`.

---

## E. Arregla el template

**E1.** Encuentra los errores (hay 3):
```yaml
Resources:
  MiRol:
    Type: AWS::IAM::Role
    Properties:
      RoleName: app-rol
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

**E2.** Encuentra el error de referencia y de dependencia:
```yaml
Resources:
  MiCola:
    Type: AWS::SQS::Queue
    DependsOn: MiBucket
    Properties:
      QueueName: cola
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt MiDLQ.Arn
        maxReceiveCount: 3
  MiBucket:
    Type: AWS::S3::Bucket
```

**E3.** Este template no despliega bien el secreto en la Lambda. Arréglalo:
```yaml
Resources:
  MiFuncion:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: f
      Runtime: python3.12
      Handler: index.handler
      Role: !Ref LambdaRol          # <-- aquí
      Environment:
        Variables:
          DB_PASS: '{{resolve:secretsmanager:mi-secreto}}'   # <-- y aquí
      Code:
        ZipFile: "def handler(e,c): return 1"
  LambdaRol:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal: { Service: lambda.amazonaws.com }
            Action: sts:AssumeRole
```

**E4.** ¿Por qué este bucket pierde los datos al borrar el stack y cómo lo blindas para que (a) no se borre y (b) un update nunca lo reemplace?
```yaml
Resources:
  DatosBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: datos-criticos
```

---

# RESPUESTAS

## Respuestas A
| # | Resp | Por qué |
|---|---|---|
| A1 | **b** | `Resources` es la única sección obligatoria. |
| A2 | **b** | Para un IAM Role, `Ref` devuelve el **nombre**; el ARN se obtiene con `!GetAtt MiRol.Arn`. |
| A3 | **b** | Nombre propio en IAM → `CAPABILITY_NAMED_IAM`. |
| A4 | **c** | `Fn::Sub` interpola `${variables}`. |
| A5 | **b** | El recurso se **recrea** (reemplaza). |
| A6 | **b** | `Retain` conserva el recurso. |
| A7 | **c** | `AWS::NoValue` elimina la propiedad. |
| A8 | **c** | `AWS::LanguageExtensions`. |
| A9 | **b** | `!GetAtt NestedStack.Outputs.Nombre`. |
| A10 | **b** | Para SQS, `Ref` devuelve la **URL**; el ARN con `!GetAtt .Arn`. |
| A11 | **c** | `CreationPolicy` + `cfn-signal`. |
| A12 | **a** | `Mappings`. |
| A13 | **b** | `{{resolve:ssm-secure:/ruta}}`. |
| A14 | **b** | Drift solo **reporta**, no corrige. |
| A15 | **c** | `create-change-set` genera la preview. |

## Respuestas B
**B1.** `!Ref` devuelve el identificador principal del recurso (o el valor de un parámetro); varía por tipo (nombre, URL, ID...). `!GetAtt` devuelve un **atributo concreto** del recurso (ARN, DNS, endpoint, GroupId...).

**B2.** Logical ID = el nombre que tú le das al recurso dentro del template (solo alfanumérico). Physical ID = el nombre/ARN real que AWS asigna cuando lo crea.

**B3.** Cuando hay una dependencia real que CFN **no puede inferir** porque no existe un `!Ref`/`!GetAtt` entre los recursos (ej. una política que debe aplicarse antes de que otro recurso la use sin referencia directa).

**B4.** Codifica un string a Base64. Típico en `UserData` de EC2 (AWS exige el script de arranque en Base64).

**B5.** Inline policy vive **dentro** del rol/usuario (relación 1:1, no reutilizable). Managed policy es un recurso **independiente** (`AWS::IAM::ManagedPolicy` o una de AWS) que puedes adjuntar a varios roles/usuarios.

**B6.** Falla: CloudFormation **no permite** borrar (ni modificar) un export que otro stack está importando. Hay que romper la dependencia primero (quitar el `ImportValue` del otro stack).

**B7.** `CreationPolicy` = al **crear** (espera N señales de éxito). `UpdatePolicy` = **cómo** se actualiza (rolling updates en ASG). `UpdateReplacePolicy` = qué pasa con el recurso **viejo** si un update lo reemplaza (Delete/Retain/Snapshot).

**B8.** Ejecutar lógica o crear recursos que CFN no soporta nativamente. Su Lambda **debe responder** al `ResponseURL` del evento con `SUCCESS` o `FAILED`; si no, el stack se queda colgado hasta el timeout.

## Respuestas C
**C1.**
```yaml
InstanceType: !If [EsProd, t3.large, t3.micro]
```
**C2.**
```yaml
Value: !GetAtt MiRol.Arn
Export:
  Name: dev-MiRolArn
```
**C3.**
```yaml
ImageId: !FindInMap [RegionAmiMap, !Ref "AWS::Region", AMI]
```
**C4.**
```yaml
RedrivePolicy:
  deadLetterTargetArn: !GetAtt MiDLQ.Arn
  maxReceiveCount: 3
```
**C5.**
```yaml
Resource: !Sub "arn:${AWS::Partition}:s3:::${AWS::StackName}-datos"
```

## Respuestas D
**D1.**
```yaml
Parameters:
  Env:
    Type: String
    Default: dev
    AllowedValues: [dev, qa, prod]
    Description: Entorno de despliegue.
```
**D2.**
```yaml
LambdaRol:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```
**D3.**
```yaml
Conditions:
  EsProd: !Equals [!Ref Env, prod]
  UsaKms: !Not [!Equals [!Ref KmsKeyArn, ""]]
```
**D4.**
```yaml
UserData:
  Fn::Base64: !Sub |
    #!/bin/bash
    yum install -y httpd
    systemctl enable --now httpd
```
**D5.**
```yaml
MiSecreto:
  Type: AWS::SecretsManager::Secret
  Properties:
    GenerateSecretString:
      SecretStringTemplate: '{"username":"admin"}'
      GenerateStringKey: password
      PasswordLength: 20
```

## Respuestas E
**E1.** Errores:
1. Crea un rol IAM con **nombre** (`RoleName`), así que al desplegar necesitas `--capabilities CAPABILITY_NAMED_IAM`.
2. El Output usa `!Ref MiRol`, que para un rol devuelve el **nombre**, no el ARN. Debe ser `!GetAtt MiRol.Arn`.
3. (Buena práctica) Falta `AWSTemplateFormatVersion`/`Description` y una política de permisos; no rompe, pero el rol no hace nada útil sin policies.
Corrección del Output:
```yaml
Outputs:
  Arn:
    Value: !GetAtt MiRol.Arn
```

**E2.** 
- **Referencia rota**: se usa `!GetAtt MiDLQ.Arn` pero `MiDLQ` **no está declarado**. Hay que crear la cola `MiDLQ`.
- **DependsOn innecesario**: `DependsOn: MiBucket` no tiene sentido (no hay relación cola↔bucket). Sobra. La dependencia real (con la DLQ) la resuelve CFN sola por el `!GetAtt`.
Corrección:
```yaml
Resources:
  MiDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: cola-dlq
  MiCola:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: cola
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt MiDLQ.Arn
        maxReceiveCount: 3
```

**E3.** Dos errores:
1. `Role: !Ref LambdaRol` → `Ref` de un rol da el **nombre**, pero Lambda espera el **ARN**. Debe ser `Role: !GetAtt LambdaRol.Arn`.
2. La dynamic reference de Secrets Manager está incompleta: falta indicar la clave del JSON. Formato correcto: `'{{resolve:secretsmanager:mi-secreto:SecretString:password}}'`.
Corrección:
```yaml
      Role: !GetAtt LambdaRol.Arn
      Environment:
        Variables:
          DB_PASS: '{{resolve:secretsmanager:mi-secreto:SecretString:password}}'
```

**E4.** Pierde los datos porque sin `DeletionPolicy`, el default es `Delete`: al borrar el stack, borra el bucket. Para blindarlo:
```yaml
Resources:
  DatosBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain          # (a) no se borra con el stack
    UpdateReplacePolicy: Retain     # conserva si un update lo reemplaza
    Properties:
      BucketName: datos-criticos
```
Y para que **un update nunca lo reemplace**, añade una **stack policy** que deniegue `Update:Replace` sobre el Logical ID `DatosBucket`:
```json
{
  "Statement": [
    { "Effect": "Allow", "Action": "Update:*", "Principal": "*", "Resource": "*" },
    { "Effect": "Deny", "Action": "Update:Replace", "Principal": "*",
      "Resource": "LogicalResourceId/DatosBucket" }
  ]
}
```

---

## Autoevaluación

| Sección | Preguntas | Mi puntaje |
|---|---|---|
| A — Opción múltiple | 15 | ___ / 15 |
| B — Preguntas cortas | 8 | ___ / 8 |
| C — Completa el template | 5 | ___ / 5 |
| D — Escribe de memoria | 5 | ___ / 5 |
| E — Arregla el template | 4 | ___ / 4 |
| **Total** | **37** | **___ / 37** |

> Meta antes de la prueba: **33/37 o más** sin mirar la referencia.
