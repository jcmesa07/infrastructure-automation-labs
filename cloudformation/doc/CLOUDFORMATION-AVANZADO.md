# AWS CloudFormation — Parte 2: Avanzado

> Complemento de `CLOUDFORMATION-REFERENCIA-COMPLETA.md`. Cubre los temas de nivel
> **Advanced** que suelen diferenciar en la prueba: nested stacks, StackSets, change
> sets, drift, custom resources, macros, bootstrapping de EC2, dynamic references,
> stack policies y herramientas.

---

## Índice
1. [Nested Stacks](#1-nested-stacks)
2. [Cross-stack references (export/import)](#2-cross-stack-references)
3. [Nested vs Cross-stack: cuándo cada uno](#3-nested-vs-cross-stack)
4. [Change Sets a fondo](#4-change-sets-a-fondo)
5. [Drift detection](#5-drift-detection)
6. [Custom Resources](#6-custom-resources)
7. [Macros y Transforms](#7-macros-y-transforms)
8. [Fn::ForEach y language extensions](#8-fnforeach-y-language-extensions)
9. [Bootstrapping de EC2: CreationPolicy, WaitCondition y helper scripts](#9-bootstrapping-de-ec2)
10. [UpdatePolicy (rolling updates en ASG)](#10-updatepolicy)
11. [Dynamic references (SSM y Secrets Manager)](#11-dynamic-references)
12. [Stack policies](#12-stack-policies)
13. [Termination protection y rollback](#13-termination-protection-y-rollback)
14. [Resource import](#14-resource-import)
15. [StackSets](#15-stacksets)
16. [Modules, Registry y Hooks](#16-modules-registry-y-hooks)
17. [SAM y CDK (relación con CFN)](#17-sam-y-cdk)
18. [Herramientas del ecosistema](#18-herramientas-del-ecosistema)
19. [Límites y cuotas](#19-límites-y-cuotas)
20. [Ejercicios](#20-ejercicios)
21. [Soluciones](#21-soluciones)

---

## 1. Nested Stacks

Un stack puede crear otros stacks como recursos usando `AWS::CloudFormation::Stack`. Sirve para **modularizar** y **reutilizar** plantillas (ej. una plantilla de VPC, otra de app).

```yaml
Resources:
  RedStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/mi-bucket/plantillas/vpc.yml
      Parameters:
        CidrBlock: 10.0.0.0/16
      TimeoutInMinutes: 20
      Tags:
        - Key: Modulo
          Value: red
```

- El **TemplateURL** debe apuntar a un template en **S3** (o un paquete generado con `aws cloudformation package`).
- El stack padre pasa `Parameters` al hijo y puede leer sus `Outputs` con `!GetAtt RedStack.Outputs.NombreDelOutput`.
- Borrar el padre borra los hijos (anidados).

```yaml
# Leer un output de un nested stack:
VpcId: !GetAtt RedStack.Outputs.VpcId
```

> `aws cloudformation package --template-file app.yml --s3-bucket mi-bucket --output-template-file packaged.yml` sube artefactos locales (código Lambda, sub-templates) a S3 y reescribe las rutas automáticamente.

---

## 2. Cross-stack references

Alternativa a los nested: stacks **independientes** que comparten valores vía `Export`/`ImportValue`.

```yaml
# Stack A (red) exporta:
Outputs:
  VpcId:
    Value: !Ref MiVpc
    Export:
      Name: red-VpcId      # nombre único por región

# Stack B (app) importa:
Resources:
  MiSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !ImportValue red-VpcId
```

Reglas clave:
- El nombre del export es **único por región y cuenta**.
- **No puedes borrar ni modificar** un export que otro stack está importando (hay que romper la dependencia primero).
- No se pueden usar `ImportValue` con nombres que dependan de parámetros de forma dinámica en todos los casos (hay restricciones).

---

## 3. Nested vs Cross-stack

| | Nested Stacks | Cross-stack (export/import) |
|---|---|---|
| Acoplamiento | Fuerte (padre controla hijos) | Débil (stacks independientes) |
| Ciclo de vida | Compartido (se despliegan juntos) | Separado |
| Reutilización | Alta (plantillas parametrizadas) | Media |
| Cuándo | Componentes que siempre van juntos | Recursos compartidos por muchos stacks (ej. VPC central) |

---

## 4. Change Sets a fondo

Un change set es la **preview** de los cambios antes de aplicarlos (como `terraform plan`).

```bash
# Crear el change set
aws cloudformation create-change-set \
  --stack-name mi-stack \
  --change-set-name cambios-1 \
  --template-body file://t.yml

# Ver qué va a cambiar
aws cloudformation describe-change-set \
  --stack-name mi-stack --change-set-name cambios-1

# Ejecutarlo (aplicar)
aws cloudformation execute-change-set \
  --stack-name mi-stack --change-set-name cambios-1

# Descartarlo
aws cloudformation delete-change-set \
  --stack-name mi-stack --change-set-name cambios-1
```

Lo importante del describe: te dice si cada recurso será **Add**, **Modify** o **Remove**, y si un Modify requiere **Replacement** (recrear el recurso, con posible pérdida de datos/IP).

> `Replacement: True` es la señal de alarma: ese cambio **recrea** el recurso.

---

## 5. Drift detection

Detecta si alguien cambió recursos **por fuera** de CloudFormation (consola, CLI directa).

```bash
aws cloudformation detect-stack-drift --stack-name mi-stack
aws cloudformation describe-stack-resource-drifts --stack-name mi-stack
```

Estados: `IN_SYNC`, `MODIFIED`, `DELETED`, `NOT_CHECKED`. CFN no corrige el drift automáticamente; solo lo reporta. La buena práctica es no editar recursos gestionados por CFN.

---

## 6. Custom Resources

Para crear/configurar cosas que CFN **no soporta nativamente** o lógica personalizada. CFN invoca una **Lambda** (o un topic SNS) en create/update/delete y espera una respuesta.

```yaml
Resources:
  MiRecursoCustom:
    Type: Custom::MiLogica          # Custom::<lo-que-quieras> o AWS::CloudFormation::CustomResource
    Properties:
      ServiceToken: !GetAtt MiLambdaCustom.Arn   # la Lambda que maneja el evento
      ParametroExtra: valor
```

La Lambda recibe un evento con `RequestType` (`Create`/`Update`/`Delete`) y **debe** responder al `ResponseURL` con `SUCCESS` o `FAILED`. Si no responde, el stack se queda colgado hasta el timeout.

Casos típicos: poblar una tabla, generar un recurso en un servicio sin soporte CFN, lookups dinámicos.

---

## 7. Macros y Transforms

`Transform` aplica **macros** que procesan/expanden el template antes del despliegue.

- **`AWS::Serverless-2016-10-31`** → SAM. Habilita `AWS::Serverless::Function`, `::Api`, `::Table`, etc.
- **`AWS::Include`** → inserta un snippet de template guardado en S3.
  ```yaml
  Transform:
    Name: AWS::Include
    Parameters:
      Location: s3://mi-bucket/snippets/comun.yml
  ```
- **`AWS::LanguageExtensions`** → añade funciones como `Fn::ForEach`, `Fn::Length`, `Fn::ToJsonString`.
- **Macros propias** → una Lambda que transforma el template a tu gusto (requiere `CAPABILITY_AUTO_EXPAND` al desplegar).

---

## 8. Fn::ForEach y language extensions

Con `Transform: AWS::LanguageExtensions` puedes hacer **loops** (antes imposible en CFN):

```yaml
Transform: AWS::LanguageExtensions
Resources:
  Fn::ForEach::Buckets:
    - NombreLogico                       # variable de iteración
    - [alpha, beta, gamma]               # lista de valores
    - "Bucket${NombreLogico}":           # plantilla del recurso por iteración
        Type: AWS::S3::Bucket
        Properties:
          BucketName: !Sub "datos-${NombreLogico}"
```

Esto genera `Bucketalpha`, `Bucketbeta`, `Bucketgamma`. Otras funciones: `Fn::Length` (tamaño de lista) y `Fn::ToJsonString`.

---

## 9. Bootstrapping de EC2

Cómo CFN sabe que una instancia terminó de configurarse (no solo de arrancar).

### CreationPolicy + cfn-signal
El recurso no se marca `CREATE_COMPLETE` hasta recibir N señales de éxito.

```yaml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    CreationPolicy:
      ResourceSignal:
        Count: 1
        Timeout: PT10M            # formato ISO8601: 10 minutos
    Metadata:
      AWS::CloudFormation::Init:  # instrucciones para cfn-init
        config:
          packages:
            yum:
              httpd: []
          services:
            sysvinit:
              httpd:
                enabled: true
                ensureRunning: true
    Properties:
      # ... ImageId, InstanceType, etc.
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          # cfn-init lee el Metadata AWS::CloudFormation::Init y lo aplica
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServer --region ${AWS::Region}
          # cfn-signal avisa a CFN que todo salió bien
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource WebServer --region ${AWS::Region}
```

Helper scripts (vienen en Amazon Linux):
- **`cfn-init`** → aplica el `Metadata: AWS::CloudFormation::Init` (paquetes, archivos, servicios, comandos).
- **`cfn-signal`** → envía la señal de éxito/fallo al `CreationPolicy`.
- **`cfn-hup`** → demonio que detecta cambios en el Metadata y re-ejecuta cfn-init.
- **`cfn-get-metadata`** → lee metadata de un recurso.

### WaitCondition (alternativa clásica)
```yaml
  EsperarApp:
    Type: AWS::CloudFormation::WaitCondition
    DependsOn: WebServer
    Properties:
      Handle: !Ref WaitHandle
      Timeout: "600"
      Count: 1
  WaitHandle:
    Type: AWS::CloudFormation::WaitConditionHandle
```

> `CreationPolicy` + `cfn-signal` es el enfoque moderno recomendado sobre WaitCondition.

---

## 10. UpdatePolicy

Controla **cómo** se actualiza un AutoScalingGroup, un dominio ElastiCache o una Lambda alias. Lo más común: rolling updates en ASG.

```yaml
Resources:
  MiASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 2
        PauseTime: PT5M
        WaitOnResourceSignals: true   # espera cfn-signal de las nuevas instancias
    Properties:
      # ...
```

Diferencia clave: `CreationPolicy` = al **crear**; `UpdatePolicy` = al **actualizar**; `UpdateReplacePolicy` = qué pasa con el recurso viejo si lo **reemplazan**.

---

## 11. Dynamic references

Insertan valores de SSM Parameter Store o Secrets Manager **en tiempo de despliegue**, sin ponerlos en el template.

```yaml
# SSM Parameter Store (texto plano)
ImageId: '{{resolve:ssm:/mi-app/ami-id}}'
ImageId: '{{resolve:ssm:/mi-app/ami-id:2}}'        # versión 2

# SSM SecureString (cifrado con KMS)
Password: '{{resolve:ssm-secure:/mi-app/db-pass}}'

# Secrets Manager (una clave del JSON del secreto)
MasterUserPassword: '{{resolve:secretsmanager:mi-secreto:SecretString:password}}'
```

Formato Secrets Manager: `{{resolve:secretsmanager:ID-o-ARN:SecretString:clave-json:etiqueta-version}}`.

> Diferencia con `Parameters` tipo SSM: las dynamic references se resuelven **cada vez** que se despliega; un parámetro SSM se resuelve al pasar el parámetro.

---

## 12. Stack policies

Protegen recursos de **updates accidentales**. Es un JSON asociado al stack (distinto de las IAM policies).

```json
{
  "Statement": [
    { "Effect": "Allow", "Action": "Update:*", "Principal": "*", "Resource": "*" },
    { "Effect": "Deny", "Action": "Update:Replace", "Principal": "*",
      "Resource": "LogicalResourceId/MiBaseDeDatos" }
  ]
}
```

```bash
aws cloudformation set-stack-policy --stack-name s --stack-policy-body file://policy.json
```

Acciones: `Update:Modify`, `Update:Replace`, `Update:Delete`, `Update:*`. Útil para blindar bases de datos.

---

## 13. Termination protection y rollback

```bash
# Protección contra borrado del stack
aws cloudformation update-termination-protection \
  --stack-name s --enable-termination-protection

# Configuración de rollback (deshacer si fallan ciertas alarmas)
aws cloudformation deploy ... \
  --rollback-configuration MonitoringTimeInMinutes=5,RollbackTriggers=[...]
```

- Por defecto, si un recurso falla al crear/actualizar, CFN hace **rollback** de todo el stack.
- `--disable-rollback` deja el stack en el estado fallido para depurar.
- **Rollback triggers**: alarmas de CloudWatch que, si se disparan durante el despliegue, provocan rollback.

---

## 14. Resource import

Importa recursos **ya existentes** (creados a mano) para que CFN los gestione, sin recrearlos.

```bash
aws cloudformation create-change-set \
  --stack-name s --change-set-name importar \
  --change-set-type IMPORT \
  --resources-to-import file://recursos.json \
  --template-body file://t.yml
```

El template debe incluir el recurso con un `DeletionPolicy` definido. Útil para "adoptar" infraestructura legada en IaC.

---

## 15. StackSets

Despliegan el **mismo template en muchas cuentas y regiones** de una sola operación (gobierno centralizado en AWS Organizations).

- **Administrator account** → donde defines el StackSet.
- **Target accounts** → donde se crean las instancias del stack.
- Dos modelos de permisos: **self-managed** (roles IAM que creas tú) o **service-managed** (integrado con Organizations).

```bash
aws cloudformation create-stack-set --stack-set-name base --template-body file://t.yml
aws cloudformation create-stack-instances --stack-set-name base \
  --accounts 111111111111 222222222222 --regions us-east-1 us-east-2
```

---

## 16. Modules, Registry y Hooks

- **CloudFormation Registry** → catálogo de tipos de recurso, incluidos de **terceros** (Datadog, MongoDB...) y propios.
- **Modules** (`AWS::CloudFormation::ModuleVersion`) → empaquetan un fragmento reutilizable de recursos como un "tipo" propio (`::MODULE`).
- **Resource types propios** → defines un recurso custom con su schema y handlers (create/read/update/delete/list) usando el CFN CLI.
- **Hooks** → validan/bloquean cambios **antes** de aprovisionar (ej. "ningún S3 sin cifrado"). Policy-as-code preventivo.

---

## 17. SAM y CDK

- **SAM (Serverless Application Model)** → extensión de CFN para serverless. `Transform: AWS::Serverless-2016-10-31`. Tipos cortos: `AWS::Serverless::Function`, `::Api`, `::HttpApi`, `::Table`. `sam build`, `sam deploy`, `sam local`. Se expande a CFN normal.
- **CDK (Cloud Development Kit)** → defines infra en código (TypeScript, Python...). `cdk synth` **genera un template CloudFormation** y `cdk deploy` lo despliega. Es decir, CDK **compila a CFN** por debajo.

---

## 18. Herramientas del ecosistema

| Herramienta | Para qué | Local / AWS |
|---|---|---|
| `cfn-lint` | Lint y validación de sintaxis/propiedades | Local |
| `aws cloudformation validate-template` | Validación básica (sintaxis) | AWS |
| `cfn-guard` | Policy as code (reglas de cumplimiento) | Local |
| `taskcat` | Testing: despliega en varias regiones y reporta | AWS |
| `rain` | CLI moderna para desplegar/formatear templates | AWS |
| `aws cloudformation package` | Sube artefactos a S3 y reescribe rutas | AWS |
| `cfn-flip` | Convierte entre YAML y JSON | Local |

> `cfn-lint` es el que más vale la pena dominar: atrapa errores antes de desplegar, sin tocar AWS.

---

## 19. Límites y cuotas

Valores que a veces caen en preguntas "advanced" (pueden cambiar con el tiempo):

- Tamaño de template: **1 MB** (al subir a la API; 460 800 bytes si va inline en el request).
- Recursos por stack: **500**.
- Parámetros por template: **200**.
- Outputs por template: **200**.
- Mappings por template: **200**.
- Resources anidados: los nested stacks cuentan como un recurso en el padre.
- `Description`: máx **1024 bytes**.

---

## 20. Ejercicios

### Conceptuales
1. ¿Diferencia entre nested stacks y cross-stack references?
2. ¿Cómo lees un output de un nested stack desde el padre?
3. ¿Qué indica `Replacement: True` en un change set?
4. ¿Qué hace la detección de drift y corrige algo automáticamente?
5. ¿Para qué sirve un Custom Resource y qué debe hacer su Lambda sí o sí?
6. ¿Qué `Transform` habilita `Fn::ForEach`?
7. ¿Diferencia entre `CreationPolicy`, `UpdatePolicy` y `UpdateReplacePolicy`?
8. ¿Qué hacen `cfn-init` y `cfn-signal`?
9. Escribe una dynamic reference que lea la clave `password` del secreto `mi-secreto`.
10. ¿Para qué sirve una stack policy y en qué se diferencia de una IAM policy?
11. ¿Qué capability necesitas para desplegar un template con una macro propia?
12. ¿Qué relación hay entre CDK/SAM y CloudFormation?

### Escenario
13. Tienes una base de datos crítica en un stack. Quieres: (a) que no se borre si eliminas el stack, (b) que un update jamás la reemplace. ¿Qué dos mecanismos usas?

---

## 21. Soluciones

> Intenta responder primero sin mirar.

1. Nested = stacks hijos controlados por un padre, ciclo de vida compartido. Cross-stack = stacks independientes que comparten valores con `Export`/`ImportValue`.
2. Con `!GetAtt NombreDelNestedStack.Outputs.NombreOutput`.
3. Que ese cambio **recrea** (reemplaza) el recurso, con posible pérdida de datos/IP/ID.
4. Reporta si hay cambios hechos fuera de CFN (`MODIFIED`, `DELETED`...). **No** corrige nada automáticamente.
5. Ejecutar lógica o crear recursos no soportados nativamente. Su Lambda **debe responder** al `ResponseURL` con SUCCESS/FAILED, o el stack se cuelga.
6. `AWS::LanguageExtensions`.
7. `CreationPolicy` = al crear (espera señales). `UpdatePolicy` = cómo actualizar (rolling en ASG). `UpdateReplacePolicy` = qué pasa con el recurso viejo si un update lo reemplaza (Delete/Retain/Snapshot).
8. `cfn-init` aplica el `Metadata: AWS::CloudFormation::Init` (paquetes, archivos, servicios). `cfn-signal` avisa al `CreationPolicy` que la config terminó OK (o falló).
9. `'{{resolve:secretsmanager:mi-secreto:SecretString:password}}'`.
10. Protege recursos del stack de updates accidentales (acciones `Update:*`). La IAM policy controla permisos de usuarios/roles; la stack policy controla qué se puede modificar **dentro** del stack.
11. `CAPABILITY_AUTO_EXPAND`.
12. Ambos **generan/compilan a templates CloudFormation**: SAM se expande vía Transform; CDK sintetiza (`cdk synth`) un template y lo despliega. CFN es la capa de ejecución por debajo.

**Escenario**
13. (a) `DeletionPolicy: Retain` (y `UpdateReplacePolicy: Retain`) en el recurso de la BD. (b) Una **stack policy** con `Effect: Deny` sobre `Update:Replace` para el Logical ID de la base de datos.
