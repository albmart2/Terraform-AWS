# Terraform + AWS — Guía del temario desarrollado

## PARTE I - Fundamentos

### 1. Introducción a Infrastructure as Code (IaC)

Durante años, la infraestructura (servidores, redes, bases de datos) se gestionaba **manualmente**: un administrador entraba en la consola de AWS, pulsaba botones, creaba una VPC, lanzaba una instancia EC2, configuraba un Security Group... Este enfoque tiene problemas graves a medida que crece un proyecto:

- **No es repetible:** si quieres el mismo entorno en "staging" que en "producción", tienes que repetir manualmente todos los pasos, y es fácil que se te olvide algo.
- **vNo es auditable:** no hay un historial claro de quién cambió qué y cuándo, más allá de los logs de CloudTrail (que no explican el "por qué").
- **No es versionable:** no puedes hacer ```git diff``` de la infraestructura ni volver atrás fácilmente si algo se rompe.
- **Propenso a "configuration drift":** con el tiempo, el entorno real se desvía de lo que "se supone" que debería ser, porque alguien hizo un cambio a mano y no lo documentó.

**Infrastructure as Code** resuelve esto tratando la infraestructura igual que tratamos el código de una aplicación:

- Se define en ficheros de texto plano.
- Se versiona en Git (con historial, PRs, revisiones).
- Se aplica de forma automatizada y repetible.
- Se puede probar, revisar y auditar como cualquier otro código.

Existen dos grandes enfoques dentro de IaC:

|Enfoque|Qué significa|Ejemplo|
|-------|-------------|-------|
|Declarativo|Describes el "estado final" que quieres, y la herramienta calcula cómo llegar ahí|Terraform, CloudFormation|
|Imperativo|Describes los pasos exactos a ejecutar, en orden|Scripts bash, algunos usos de Ansible|

Terraform es fundamentalmente **declarativo**: tú escribes "quiero un bucket S3 llamado ```mi-app-logs```", y Terraform decide si tiene que crearlo, modificarlo o dejarlo tal cual, comparando lo que pides con lo que ya existe.

#### Ejemplo mental (sin código aún)

Imagina que describes esto en un fichero:

```
Quiero:
- 1 VPC con CIDR 10.0.0.0/16
- 2 subnets públicas
- 1 instancia EC2 tipo t3.micro en la primera subnet
```

Con IaC, ese fichero es la **única fuente de verdad**. Si alguien borra la instancia a mano desde la consola, la siguiente vez que apliques el fichero, la herramienta detectará la diferencia y la volverá a crear (o te avisará).

### 2. ¿Qué es Terraform?

Terraform es una herramienta de **HashiCorp**, escrita en Go, que permite definir, planificar y aplicar infraestructura como código usando un lenguaje propio llamado **HCL** (HashiCorp Configuration Language).

Características clave:

- **Multi-cloud / multi-plataforma:** no está atado a un proveedor. El mismo motor de Terraform puede gestionar AWS, Azure, GCP, Kubernetes, GitHub, Datadog, Cloudflare... cualquier cosa que tenga un "provider".
- **Declarativo:** describes el resultado deseado, no los pasos.
- **Basado en estado:** Terraform mantiene un fichero de estado que representa lo que "cree" que existe en el mundo real, y lo usa para calcular diferencias.
- **Ciclo de vida claro:** ```init``` → ```plan``` → ```apply``` → (más adelante) ```destroy```.

#### Ejemplo mínimo de código Terraform

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}

resource "aws_s3_bucket" "logs" {
  bucket = "mi-app-logs-2026"
}
```

Con solo esto, ejecutando:

```bash
bashterraform init
terraform plan
terraform apply
```

Terraform crea un bucket S3 real en tu cuenta de AWS. Este ejemplo, aunque simple, ya contiene los tres bloques fundamentales que verás en todo el curso: ```terraform {}``` (configuración del propio Terraform), ```provider {}``` (a qué nube conectas) y ```resource {}``` (qué quieres crear).

### 3. Historia de Terraform


- **2014:** HashiCorp lanza Terraform como proyecto open source, en un momento en que la gestión de infraestructura cloud empezaba a explotar en complejidad (más regiones, más servicios, más cuentas).
- **2014–2020:** Terraform se convierte en el estándar de facto de IaC multi-cloud, superando en adopción a alternativas propietarias de cada nube, gracias a su enfoque agnóstico y su ecosistema de providers.
- **Terraform 0.12 (2019):** reescritura importante del lenguaje HCL, con mejor soporte de tipos y expresiones — la sintaxis que usarás en este curso es la posterior a esa versión.
- **Terraform 1.0 (2021):**b primera versión con garantía de estabilidad a largo plazo en la API de configuración.
- **Agosto 2023:** HashiCorp anuncia el cambio de licencia de Terraform (y otros productos) de **MPL 2.0 (open source)** a **BSL (Business Source License)**, restringiendo su uso comercial en productos competidores.
- **Como respuesta**, un conjunto de empresas (incluyendo Gruntwork, Harness, Spacelift, entre otras) crean **OpenTofu**, un fork 100% open source de Terraform, hoy bajo la Linux Foundation.

Este contexto es relevante hoy: al buscar documentación o módulos, te encontrarás referencias tanto a Terraform como a OpenTofu, que en la práctica siguen siendo casi idénticos en sintaxis.

### 4. Terraform vs CloudFormation

**CloudFormation** es el servicio nativo de IaC de AWS.

|Aspecto|Terraform|CloudFormation|
|-------|---------|--------------|
|Alcance|Multi-cloud|Solo AWS|
|Lenguaje|HCL (propio, legible)|YAML / JSON (más verboso)|
|Estado|Fichero de estado gestionado por el usuario (o remoto)|Gestionado internamente por AWS, no lo ves directamente|
|Soporte de nuevos servicios AWS|A veces con retraso (depende del provider)|Inmediato, al ser de AWS|
|Curva de aprendizaje|Media (HCL es sencillo pero hay que aprenderlo)|Media-alta (YAML extenso, "intrínsecas" como ```!Ref```, ```!GetAtt```)|
|Rollback automático|No (a menos que lo gestiones tú)|Sí, integrado|
|Coste|Gratuito (CLI)|Gratuito (el servicio en sí no tiene coste adicional)|

**Cuándo elegir cada uno**: si trabajas exclusivamente en AWS y quieres integración nativa total (incluyendo rollback automático), CloudFormation tiene sentido. Si hay cualquier posibilidad de trabajar con más de un proveedor, o prefieres un lenguaje más legible, Terraform suele ganar en la práctica de la industria.

### 5. Terraform vs Pulumi

**Pulumi** permite escribir infraestructura usando lenguajes de programación de propósito general: TypeScript, Python, Go, C#, Java.

```python
python# Ejemplo conceptual en Pulumi (Python) — solo ilustrativo
import pulumi_aws as aws

bucket = aws.s3.Bucket("logs", bucket="mi-app-logs-2026")
```

Comparado con Terraform:

- **Ventaja de Pulumi:** puedes usar bucles, condicionales, funciones y estructuras de datos "reales" del lenguaje, sin las limitaciones de un DSL declarativo.
- **Ventaja de Terraform:** HCL, al ser un lenguaje específico de dominio, es más fácil de leer para cualquiera (incluso sin experiencia previa de programación), y evita la tentación de meter lógica de aplicación dentro de la infraestructura.
- Terraform tiene un ecosistema de módulos y una comunidad considerablemente más grande y madura a día de hoy.

### 6. Terraform vs Ansible

Es una comparación algo distinta, porque **no resuelven exactamente el mismo problema**:

- **Terraform:** se centra en **aprovisionar** infraestructura (crear la VPC, la instancia EC2, la base de datos...). Es declarativo y mantiene estado.
- **Ansible:** se centra tradicionalmente en **configurar** lo que ya existe (instalar paquetes, desplegar una aplicación dentro de un servidor ya creado). Es más procedural (ejecuta "playbooks" paso a paso) y no mantiene un fichero de estado persistente.

En muchos proyectos reales, se usan **juntos**: Terraform crea la instancia EC2, y luego Ansible entra por SSH para instalar y configurar el software dentro de ella. Este curso se centra en Terraform, pero es útil entender dónde termina su responsabilidad.

### 7. Casos de uso

Terraform se usa típicamente para:

- **Entornos multi-cuenta:** separar dev, staging y producción en cuentas AWS distintas, replicando la misma infraestructura con distintos parámetros.
- **Multi-región:** desplegar el mismo stack en ```eu-west-1``` y ```us-east-1``` para alta disponibilidad o cercanía al usuario.
- **Infraestructura efímera:** crear entornos temporales para tests de integración (se crean, se prueban, se destruyen en el mismo pipeline de CI).
- **Compliance y auditoría:** al estar todo en Git, cada cambio de infraestructura pasa por revisión de código (pull request) antes de aplicarse.
- **Disaster recovery:** si toda tu infraestructura está en código, recrearla en una región distinta ante un desastre es cuestión de minutos, no días.

### 8. Arquitectura de Terraform

Terraform se compone de dos piezas principales:

```
┌─────────────────────────────┐
│         Terraform Core       │
│  (parsea HCL, calcula el     │
│   grafo de dependencias,     │
│   genera el plan)            │
└───────────────┬──────────────┘
                │ RPC
    ┌───────────┼───────────┐
    │           │           │
┌───▼───┐   ┌───▼───┐   ┌───▼───┐
│Provider│   │Provider│   │Provider│
│  AWS   │   │ GitHub │   │Datadog │
└────────┘   └────────┘   └────────┘
```

- **Terraform Core:** el motor. Lee tus ficheros ```.tf```, construye un grafo de dependencias entre recursos, y decide en qué orden crear/actualizar/eliminar cada uno.
- **Providers:** plugins independientes (binarios separados) que Terraform Core invoca vía RPC. Cada provider sabe "traducir" un bloque ```resource "aws_instance"``` en llamadas reales a la API de AWS.

Esta separación es lo que permite que Terraform sea multi-cloud: el Core no sabe nada específico de AWS — todo ese conocimiento vive en el provider ```hashicorp/aws```.

### 9. Providers

Un **provider** es un plugin que le da a Terraform la capacidad de gestionar recursos de un servicio concreto.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region  = "eu-west-1"
  profile = "mi-perfil" # opcional, usa el perfil de ~/.aws/credentials
}
```

Puntos clave:

- El bloque ```required_providers``` fija **qué** provider usar y **qué** versión (evitando sorpresas si sale una versión nueva con cambios).
- El bloque ```provider "aws" {}``` configura **cómo** conectarse (región, credenciales, tags por defecto...).
- Puedes tener **múltiples configuraciones del mismo provider** usando alias, por ejemplo para desplegar en dos regiones a la vez:

```hcl
provider "aws" {
  region = "eu-west-1"
  alias  = "irlanda"
}

provider "aws" {
  region = "us-east-1"
  alias  = "virginia"
}

resource "aws_s3_bucket" "logs_eu" {
  provider = aws.irlanda
  bucket   = "logs-eu-2026"
}

resource "aws_s3_bucket" "logs_us" {
  provider = aws.virginia
  bucket   = "logs-us-2026"
}
```

### 10. Resources

Un **resource** es la unidad básica de infraestructura: "quiero que exista esto".

```hcl
resource "<TIPO>" "<NOMBRE_LOCAL>" {
  argumento1 = valor1
  argumento2 = valor2
}
```

Ejemplo real:

```hcl
hclresource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "servidor-web"
  }
}
```

- ```aws_instance``` es el **tipo** de recurso (definido por el provider ```aws```).
- ```web``` es **el nombre local**, con el que referencias este recurso dentro de tu código Terraform (por ejemplo, ```aws_instance.web.id```). No es el nombre que verás en AWS.
- Dentro de las llaves van los **argumentos** específicos de ese tipo de recurso (varían según el tipo).

Cada resource, una vez aplicado, tiene:

- **Argumentos de entrada** (los que tú defines).
- **Atributos de salida** (los que AWS devuelve tras crearlo, como el ```id``` o la ```arn```), a los que puedes referenciar desde otros recursos.

```hcl
resource "aws_eip" "ip_fija" {
  instance = aws_instance.web.id  # referencia al atributo "id" del recurso anterior
}
```

### 11. Estado

El **estado** (```terraform.tfstate```) es un fichero JSON que Terraform genera y mantiene automáticamente. Contiene un mapeo entre:

- Lo que declaraste en tu código (```resource "aws_instance" "web"```)
- El recurso real correspondiente en AWS (su ID, y todos sus atributos actuales)

```json
{
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "attributes": {
            "id": "i-0abcd1234efgh5678",
            "instance_type": "t3.micro",
            "public_ip": "34.201.XX.XX"
          }
        }
      ]
    }
  ]
}
```

¿Por qué es imprescindible?


- Sin estado, Terraform no tendría forma de saber que ```aws_instance.web``` en tu código corresponde a la instancia con ID ```i-0abcd...``` en AWS — tendría que adivinarlo o volver a crear todo cada vez.
- En cada ```terraform plan```, Terraform: (1) lee el estado, (2) opcionalmente lo refresca contra la API real, (3) lo compara con tu código, y (4) calcula el diff.

**Cuidado:** el estado puede contener datos sensibles en texto plano (contraseñas generadas, claves...). Por eso más adelante (Parte VI) se trata la gestión de estado remoto y su cifrado.

### 12. Dependencias

Terraform necesita saber en qué **orden** crear, actualizar o destruir recursos. Hay dos formas:

#### Dependencia implícita (la más común)

Cuando un recurso referencia un atributo de otro, Terraform infiere automáticamente que debe crear primero el referenciado:

```hcl
resource "aws_vpc" "principal" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "publica" {
  vpc_id     = aws_vpc.principal.id  # <- dependencia implícita
  cidr_block = "10.0.1.0/24"
}
```

Aquí, Terraform sabe que ```aws_subnet.publica``` depende de ```aws_vpc.principal```, porque usa ```aws_vpc.principal.id```. No hace falta indicarlo manualmente.

#### Dependencia explícita

Cuando dos recursos están relacionados pero **no** comparten ningún atributo en el código (por ejemplo, hay una dependencia funcional o de permisos que Terraform no puede "ver"):

```hcl
resource "aws_iam_role_policy" "permiso" {
  # ...
}

resource "aws_lambda_function" "procesador" {
  # ...
  depends_on = [aws_iam_role_policy.permiso]
}
```

Esto le dice a Terraform: "aunque no lo veas en los atributos, crea primero la policy antes que la Lambda" (útil porque a veces AWS tarda en propagar permisos IAM).

Internamente, todo esto se traduce en un **grafo dirigido acíclico (DAG)**, que puedes visualizar con ```terraform graph``` (lo verás en la Parte IV).

## PARTE II — Instalación

### 1. Instalar Terraform

Terraform se distribuye como un **único binario ejecutable**, sin dependencias externas. Hay varias formas de instalarlo:

#### Opción A: gestor de paquetes del sistema

##### macOS (Homebrew):

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

##### Windows (Chocolatey):

```powershell
choco install terraform
```

##### Linux (Debian/Ubuntu):

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform
```

#### Opción B: binario manual

Descargar el ```.zip``` correspondiente desde ```releases.hashicorp.com/terraform```, descomprimirlo y mover el binario a una carpeta del ```PATH```:

```bash
unzip terraform_1.9.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

#### Opción C (recomendada para el curso): gestor de versiones ```tfenv```

En proyectos reales es habitual necesitar **distintas versiones de Terraform** según el proyecto (uno usa 1.5, otro usa 1.9...). ```tfenv``` permite instalar varias versiones y cambiar entre ellas:

```bash
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

tfenv install 1.9.0
tfenv use 1.9.0
tfenv list          # ver versiones instaladas
```

Esto también permite fijar la versión por proyecto con un fichero ```.terraform-version``` en la raíz del repositorio, que ```tfenv``` detecta automáticamente.

#### Verificar instalación

```bash
terraform -version
```

```
Terraform v1.9.0
on linux_amd64
```

### 2. Instalar AWS CLI

El AWS CLI es la herramienta oficial de línea de comandos de Amazon. Aunque Terraform no la necesita internamente para funcionar, es imprescindible para:

- Configurar credenciales que Terraform usará.
- Inspeccionar recursos manualmente al depurar (aws ec2 describe-instances, etc.).
- Ejecutar comandos puntuales fuera del ciclo de Terraform.

#### Instalación (versión 2, la actual)

##### macOS:

```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

##### Linux:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

##### Windows

Descargar e instalar el .msi desde la documentación oficial de AWS.

#### Verificar instalación

```bash
aws --version
```

```
aws-cli/2.17.0 Python/3.12.0 Linux/6.5.0 exe/x86_64.ubuntu.24
```

### 3. Configurar credenciales

Terraform, a través del provider de AWS, necesita autenticarse. El provider busca credenciales en este **orden de prioridad**:

1. Argumentos explícitos en el bloque ```provider "aws" {}``` (no recomendado para secretos).
2. Variables de entorno.
3. Ficheros de configuración/credenciales de AWS CLI (```~/.aws/credentials```, ```~/.aws/config```).
4. Rol de IAM asociado a la instancia/entorno de ejecución (EC2, ECS, CodeBuild...).

#### Opción A: ```aws configure``` (la más sencilla para empezar)

```bash
aws configure
```

```
AWS Access Key ID [None]: AKIAxxxxxxxxxxxxxxxx
AWS Secret Access Key [None]: ****************************
Default region name [None]: eu-west-1
Default output format [None]: json
```

Esto genera dos ficheros:

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = AKIAxxxxxxxxxxxxxxxx
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

```ini
# ~/.aws/config
[default]
region = eu-west-1
output = json
```

#### Opción B: perfiles múltiples

Si trabajas con varias cuentas AWS (personal, curso, empresa...), conviene usar perfiles nombrados:

```bash
aws configure --profile curso-terraform
```

```ini
# ~/.aws/credentials
[curso-terraform]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

Y en Terraform:

```hcl
provider "aws" {
  region  = "eu-west-1"
  profile = "curso-terraform"
}
```

O bien, sin tocar el código, mediante variable de entorno:

```bash
export AWS_PROFILE=curso-terraform
```

#### Opción C: variables de entorno directas (útil en CI/CD)

```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="eu-west-1"
```

Esta opción se usa mucho en pipelines de CI/CD (Parte IX), donde estas variables se inyectan como secrets del propio sistema de CI, nunca escritas en el repositorio.

#### Opción D: asumir un rol (assume role)

En organizaciones con múltiples cuentas AWS, es habitual autenticarte en una cuenta "central" y luego **asumir un rol** en la cuenta destino:

```hcl
provider "aws" {
  region = "eu-west-1"

  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/TerraformDeployRole"
  }
}
```

Esto evita tener credenciales de larga duración por cada cuenta y sigue el principio de mínimo privilegio a nivel organizativo.

#### Verificar que las credenciales funcionan

```bash
aws sts get-caller-identity
```

```json
{
    "UserId": "AIDAxxxxxxxxxxxxxxxxx",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/tu-usuario"
}
```

Si este comando responde correctamente, Terraform también podrá autenticarse.

### 4. IAM para Terraform

Antes de lanzar tu primer ```terraform apply```, necesitas un usuario o rol de IAM con los **permisos adecuados**. Hay dos enfoques:

#### Enfoque rápido para aprender (no recomendado en producción)

Adjuntar la policy gestionada ```AdministratorAccess``` a tu usuario de curso. Es la vía más simple para no bloquearte con permisos mientras aprendes, pero **nunca se hace así en un entorno real**.

#### Enfoque de mínimo privilegio (el correcto en proyectos reales)

Crear una policy que solo permita las acciones que Terraform necesita para los servicios que vas a gestionar. Ejemplo simplificado, permitiendo solo EC2 y S3:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:RunInstances",
        "ec2:TerminateInstances",
        "ec2:CreateTags",
        "s3:CreateBucket",
        "s3:DeleteBucket",
        "s3:PutBucketVersioning",
        "s3:GetBucketVersioning"
      ],
      "Resource": "*"
    }
  ]
}
```

En un curso conviene empezar con permisos amplios sobre los servicios concretos que se van a usar (IAM, EC2, VPC, S3, RDS...) e ir **restringiendo progresivamente** a medida que entiendes qué usa realmente cada capítulo. Esto se retomará con más profundidad en la Parte X (Buenas prácticas → Seguridad).

#### Usuario de IAM vs Rol de IAM

- **Usuario**: identidad con credenciales fijas (access key + secret key), pensada para personas o para ejecutar Terraform desde tu propio ordenador.
- **Rol**: identidad temporal, sin credenciales fijas, pensada para que la asuman servicios (como una instancia EC2 que ejecuta Terraform) o para el ```assume_role``` visto antes.

Para este curso, lo más práctico es empezar con un **usuario de IAM dedicado exclusivamente a Terraform** (no tu usuario personal de AWS), para poder revocar sus credenciales fácilmente sin afectar a otras cosas.

### 5. VSCode

Visual Studio Code es el editor recomendado para este curso, por su combinación de ligereza y ecosistema de extensiones.

#### Instalación

- **macOS**: ```brew install --cask visual-studio-code```.
- **Windows/Linux**: descargar el instalador desde ```code.visualstudio.com```.

### 6. Extensiones

Para trabajar cómodamente con Terraform en VSCode, instala:

1. **HashiCorp Terraform** (```hashicorp.terraform```): la extensión oficial. Aporta:

    - Resaltado de sintaxis HCL.
    - Autocompletado de argumentos según el provider.
    - Validación en tiempo real.
    - Formato automático (equivalente a ```terraform fmt``` al guardar).

2. **AWS Toolkit** (```amazonwebservices.aws-toolkit-vscode```): permite explorar recursos de AWS directamente desde el editor, útil para verificar visualmente lo que Terraform ha creado.
3. **YAML / Even Better TOML** (opcionales): si en el curso más adelante tocas ficheros de CI/CD (GitHub Actions usa YAML), conviene tenerlas ya instaladas.

#### Configuración recomendada (```settings.json```)

```json
{
  "[terraform]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "hashicorp.terraform"
  },
  "[terraform-vars]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "hashicorp.terraform"
  }
}
```

Esto asegura que cada vez que guardes un fichero ```.tf```, VSCode ejecute automáticamente el equivalente a ```terraform fmt```, manteniendo el código siempre bien formateado sin esfuerzo manual.

### 7. Formato del proyecto

Antes de escribir la primera línea de HCL "de verdad" (Parte III), conviene fijar una estructura de carpetas y ficheros estándar. La convención más extendida es:

```
mi-proyecto-terraform/
├── main.tf          # recursos principales
├── variables.tf     # declaración de variables de entrada
├── outputs.tf       # valores de salida
├── providers.tf     # configuración de terraform{} y provider{}
├── terraform.tfvars # valores concretos de las variables (NO se sube a Git si tiene secretos)
├── .gitignore
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── ec2/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

#### Por qué se separan así los ficheros

Terraform, técnicamente, no obliga a esta separación — **podrías poner todo en un único fichero ```main.tf```** y funcionaría igual, porque Terraform carga y combina todos los ficheros ```.tf``` de un directorio como si fueran uno solo. La separación es una **convención de legibilidad y mantenimiento**:

- ```providers.tf```: para saber de un vistazo con qué proveedores/versiones trabaja el proyecto.
- ```variables.tf```: para ver de un vistazo qué "inputs" espera el proyecto, sin tener que leer toda la lógica.
- ```outputs.tf```: qué expone este proyecto hacia fuera (por ejemplo, para que otro equipo consuma el ID de la VPC creada).
- ```main.tf```: la lógica en sí — los recursos.

#### ```.gitignore``` recomendado

```gitignore
# Estado local y sus backups
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl  # opcional: algunos equipos SÍ lo versionan, ver nota abajo

# Ficheros con variables sensibles
*.tfvars
!example.tfvars

# Logs
crash.log
```

**Nota sobre ```.terraform.lock.hcl```**: este fichero fija las versiones exactas de los providers usados. La recomendación oficial de HashiCorp es **sí versionarlo** en Git (no ignorarlo), para garantizar que todo el equipo use exactamente las mismas versiones de provider. Lo he incluido arriba como ejemplo de decisión a tomar conscientemente, no como regla fija.

## PARTE III — Sintaxis HCL (desde cero)

HCL (HashiCorp Configuration Language) es el lenguaje en el que se escribe Terraform. Es un lenguaje **declarativo**, pensado para ser legible tanto por humanos como parseable por máquinas. En esta parte lo veremos íntegramente, sin asumir conocimientos previos de programación.

### 1. Variables

Una ```variable``` declara un **parámetro de entrada** para tu configuración, para no hardcodear valores.

```hcl
variable "region" {
  description = "Región de AWS donde desplegar"
  type        = string
  default     = "eu-west-1"
}
```

Se usa en el resto del código con ```var.<nombre>```:

```hcl
provider "aws" {
  region = var.region
}
```

Formas de asignar un valor a una variable (por orden de prioridad, de menor a mayor):

1. El ```default``` del propio bloque ```variable```.
2. Fichero ```terraform.tfvars``` (se carga automáticamente).
3. Fichero ```*.auto.tfvars``` (también automático).
4. Flag ```-var="region=us-east-1"``` en la CLI.
5. Flag ```-var-file="produccion.tfvars"```.
6. Variable de entorno ```TF_VAR_region```.

```bash
# terraform.tfvars
region = "us-east-1"
```

```bash
export TF_VAR_region="us-east-1"
terraform apply -var="region=eu-central-1"  # esta gana sobre todo lo anterior
```

### 2. Tipos

HCL tiene un sistema de tipos que puedes declarar explícitamente en cada variable:

```hcl
variable "puerto" {
  type = number
}

variable "activo" {
  type = bool
}

variable "nombres" {
  type = list(string)
}
```

Tipos primitivos: ```string```, ```number```, ```bool```. Tipos de colección: ```list()```, ```set()```, ```map()```, ```object()```, ```tuple()```.

Si no declaras ```type```, Terraform intenta inferirlo, pero **declararlo siempre es una buena práctica**: detecta errores antes de aplicar, y sirve de documentación.

### 3. Strings

Los strings van entre comillas dobles, y soportan **interpolación**:

```hcllocals {
  entor
no = "produccion"
  nombre_bucket = "app-${local.entorno}-logs"
}
```

Resultado: ```"app-produccion-logs"```.

#### Heredocs (texto multilínea)

Útiles, por ejemplo, para scripts de ```user_data```:

```hcl
locals {
  script_inicio = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y nginx
    systemctl start nginx
  EOF
}
```

El ```-``` tras ```<<``` (es decir, ```<<-EOF```) permite indentar el heredoc sin que esa indentación se incluya en el resultado final.

#### Funciones de string comunes

```hcl
upper("hola")            # "HOLA"
lower("HOLA")             # "hola"
trimspace("  hola  ")     # "hola"
substr("terraform", 0, 4) # "terr"
format("app-%s-%03d", "web", 7)  # "app-web-007"
```

### 4. Numbers

Enteros y decimales, sin distinción de tipo (a diferencia de otros lenguajes):

```hcl
variable "cantidad_instancias" {
  type    = number
  default = 3
}

locals {
  total_gb = 100 * 1.5   # 150
  mitad    = 10 / 3      # 3.3333333333
}
```

Operadores aritméticos disponibles: ```+```, ```-```, ```*```, ```/```, ```%``` (módulo).

### 5. Maps

Colecciones **clave-valor**, todas las claves del mismo tipo (normalmente string) y todos los valores del mismo tipo:

```hcl
variable "tipo_instancia_por_entorno" {
  type = map(string)
  default = {
    dev  = "t3.micro"
    test = "t3.small"
    prod = "t3.large"
  }
}
```

Acceso a un valor:

```hcl
resource "aws_instance" "app" {
  instance_type = var.tipo_instancia_por_entorno["prod"]
  # o bien:
  # instance_type = lookup(var.tipo_instancia_por_entorno, "prod", "t3.micro")
}
```

```lookup()``` permite dar un valor por defecto si la clave no existe, evitando errores.

### 6. Lists

Colecciones **ordenadas** de elementos del mismo tipo:

```hcl
variable "zonas_disponibilidad" {
  type    = list(string)
  default = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
}
```

Acceso por índice (empezando en 0):

```hcl
locals {
  primera_zona = var.zonas_disponibilidad[0]  # "eu-west-1a"
}
```

Funciones útiles sobre listas:

```hcl
length(var.zonas_disponibilidad)      # 3
element(var.zonas_disponibilidad, 1)  # "eu-west-1b"
contains(var.zonas_disponibilidad, "eu-west-1a")  # true
```

### 7. Tuples

Como una lista, pero **los tipos de cada posición pueden ser distintos** (y son fijos en número y tipo):

```hcl
locals {
  registro = ["servidor-web", 3, true]  # tuple(string, number, bool)
}
```

En la práctica, las tuples aparecen sobre todo de forma implícita (Terraform las usa internamente), y rara vez las declaras explícitamente en variables de un proyecto normal — es más común usar ```object``` cuando necesitas datos heterogéneos con nombre.

### 8. Objects

Como un map, pero con **atributos con nombre y tipos definidos**, similar a un "registro" o "struct":

```hcl
variable "config_servidor" {
  type = object({
    nombre        = string
    cpu           = number
    tiene_backup  = bool
  })

  default = {
    nombre       = "web-01"
    cpu          = 2
    tiene_backup = true
  }
}
```

Acceso:

```hcl
locals {
  nombre_servidor = var.config_servidor.nombre
}
```

Los objects son especialmente útiles para agrupar configuración relacionada y evitar tener 10 variables sueltas.

### 9. Sets

Como una lista, pero **sin duplicados y sin orden garantizado**. Se usan sobre todo junto a ```for_each```:

```hcl
variable "puertos_permitidos" {
  type    = set(number)
  default = [22, 80, 443]
}
```

Diferencia práctica con ```list```: si intentas añadir un valor duplicado a un ```set```, Terraform lo ignora silenciosamente (no da error, simplemente no habrá duplicado).

### 10. Operadores

#### Aritméticos

```+```, ```-```, ```*```, ```/```, ```%```.

#### Comparación

```==```, ```!=```, ```<```, ```>```, ```<=```, ```>=```.

#### Lógicos

```&&``` (and), ```||``` (or), ```!``` (not).

```hcl
locals {
  es_produccion = var.entorno == "prod"
  necesita_backup = var.es_produccion && var.tiene_datos_criticos
}
```

### 11. Funciones

HCL no permite definir funciones propias (no hay ```function miFuncion()```), pero incluye un catálogo grande de **funciones integradas**. Algunas de las más usadas en el día a día:

```hcl
merge({a = 1}, {b = 2})              # {a = 1, b = 2}
concat(["a", "b"], ["c"])            # ["a", "b", "c"]
join(",", ["a", "b", "c"])           # "a,b,c"
split(",", "a,b,c")                  # ["a", "b", "c"]
coalesce(null, "", "valor")          # "valor" (primer valor no nulo/no vacío)
cidrsubnet("10.0.0.0/16", 8, 1)      # "10.0.1.0/24"
timestamp()                          # fecha/hora actual en formato RFC 3339
```

```cidrsubnet``` es especialmente importante en la Parte V: permite calcular subredes automáticamente a partir de un bloque CIDR mayor, sin tener que calcularlas a mano.

### 12. Condicionales

HCL solo tiene **una** forma de condicional: el operador ternario.

```hcl
resource "aws_instance" "app" {
  instance_type = var.entorno == "prod" ? "t3.large" : "t3.micro"
}
```

Se lee: "si ```var.entorno == 'prod'```, usa ```t3.large```; si no, usa ```t3.micro```".

Se pueden anidar, aunque a partir de 2 niveles conviene extraerlo a un ```local``` para mantener legibilidad:

```hcl
locals {
  tipo_instancia = var.entorno == "prod" ? "t3.large" : (
    var.entorno == "test" ? "t3.small" : "t3.micro"
  )
}
```

### 13. Loops

HCL no tiene bucles imperativos (```for```, ```while``` tradicionales). En su lugar, existen dos mecanismos para **repetir la creación de recursos** (```for_each``` y ```count```), y una construcción de **expresión** para transformar colecciones (```for ... in ...```), que veremos en el punto 18.

### 14. for_each

Crea una instancia del recurso **por cada elemento de un map o set**, usando la clave como identificador:

```hcl
variable "buckets" {
  type    = set(string)
  default = ["logs", "backups", "assets"]
}

resource "aws_s3_bucket" "todos" {
  for_each = var.buckets
  bucket   = "mi-app-${each.value}"
}
```

Dentro del bloque, ```each.key``` y ```each.value``` están disponibles (en un ```set```, ambos son iguales; en un ```map```, ```each.key``` es la clave y ```each.value``` es el valor).

```hcl
variable "instancias" {
  type = map(object({
    tipo = string
  }))
  default = {
    web = { tipo = "t3.micro" }
    api = { tipo = "t3.small" }
  }
}

resource "aws_instance" "app" {
  for_each      = var.instancias
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value.tipo

  tags = {
    Name = "servidor-${each.key}"
  }
}
```

Referenciar un recurso creado con ```for_each``` desde otro sitio requiere la clave:

```hcl
output "ip_web" {
  value = aws_instance.app["web"].public_ip
}
```

**Ventaja clave sobre ```count```**: si eliminas ```"api"``` del map, Terraform sabe exactamente que debe destruir solo esa instancia, sin tocar ```"web"```. Con ```count```, como verás abajo, esto no siempre es así.

### 15. count

Crea **N copias** de un recurso, identificadas por índice numérico (0, 1, 2...):

```hcl
resource "aws_instance" "worker" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "worker-${count.index}"
  }
}
```

Esto crea ```worker-0```, ```worker-1```, ```worker-2```. Se referencian así:

```hcl
output "ip_worker_0" {
  value = aws_instance.worker[0].public_ip
}
```

#### El problema clásico de ```count```

Si tienes 3 instancias (índices 0, 1, 2) y eliminas la del medio de tu lista de origen, Terraform **reindexa todo**, y puede destruir y recrear instancias que en realidad no debían cambiar, simplemente porque su índice cambió. Por eso, la recomendación general es:

- Usa ```count``` cuando los elementos son genuinamente intercambiables (N réplicas idénticas de lo mismo).
- Usa ```for_each``` cuando cada elemento tiene una identidad propia (nombres, distintas configuraciones), para evitar el problema de reindexado.

### 16. Dynamic Blocks

Sirven para generar **bloques anidados repetidos** dentro de un recurso — algo que ni ```count``` ni ```for_each``` (a nivel de recurso) pueden hacer, porque estos operan sobre el recurso completo, no sobre un bloque interno suyo.

Ejemplo clásico: un Security Group con un número variable de reglas de entrada.

```hcl
variable "reglas_entrada" {
  type = list(object({
    puerto      = number
    protocolo   = string
    cidr_origen = string
  }))
  default = [
    { puerto = 22, protocolo = "tcp", cidr_origen = "0.0.0.0/0" },
    { puerto = 80, protocolo = "tcp", cidr_origen = "0.0.0.0/0" },
    { puerto = 443, protocolo = "tcp", cidr_origen = "0.0.0.0/0" },
  ]
}

resource "aws_security_group" "web" {
  name   = "sg-web"
  vpc_id = aws_vpc.principal.id

  dynamic "ingress" {
    for_each = var.reglas_entrada
    content {
      from_port   = ingress.value.puerto
      to_port     = ingress.value.puerto
      protocol    = ingress.value.protocolo
      cidr_blocks = [ingress.value.cidr_origen]
    }
  }
}
```

Esto genera un bloque ```ingress { ... }``` por cada elemento de ```var.reglas_entrada```, sin tener que escribir cada regla a mano ni limitar el número de reglas de antemano.

### 17. Expressions

En HCL, casi todo lo que va a la derecha de un ```=``` es una **expresión**: una referencia, una operación, una llamada a función, un literal... Algunas construcciones destacadas:

#### Splat expressions (```[*]```)

Extraen un atributo de **todos** los elementos generados por un ```for_each``` o ```count``` a la vez:

```hcl
output "ips_publicas" {
  value = aws_instance.app[*].public_ip  # con count
}
```

Con ```for_each```, el equivalente es usar ```values()```:

```hcl
output "ips_publicas" {
  value = values(aws_instance.app)[*].public_ip
}
```

#### Comprensiones ```for ... in ...```

Transforman una colección en otra, de forma similar a un "list comprehension" de Python:

```hcl
locals {
  nombres_mayusculas = [for n in var.nombres : upper(n)]

  # con filtro:
  solo_produccion = [for e in var.entornos : e if e == "prod"]

  # generando un map:
  tipo_por_nombre = {for k, v in var.instancias : k => v.tipo}
}
```

### 18. Null

```null``` representa la **ausencia de valor**. Se usa a menudo para indicar "usa el comportamiento por defecto de AWS" en lugar de forzar un valor concreto:

```hcl
resource "aws_instance" "app" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  subnet_id              = var.usar_subnet_default ? null : aws_subnet.privada.id
}
```

Si una variable opcional no recibe valor y su ```default``` es ```null```, Terraform generalmente delega en el valor por defecto que la propia API de AWS aplicaría.

### 19. Sensitive

Marca variables u outputs para que Terraform **oculte su valor** en la salida de ```plan```/```apply``` (aunque sigue almacenándose en el estado, sin cifrar por defecto — de ahí la importancia de proteger el estado, Parte VI).

```hcl
variable "password_db" {
  type      = string
  sensitive = true
}

output "connection_string" {
  value     = "postgres://admin:${var.password_db}@${aws_db_instance.principal.endpoint}"
  sensitive = true
}
```

Al ejecutar ```terraform apply```, en lugar de mostrar el valor real, la CLI muestra:

```
+ connection_string = (sensitive value)
```

### 20. Validation

Permite añadir **reglas de validación personalizadas** dentro de una variable, para detectar errores de entrada antes de intentar aplicar nada:

```hcl
variable "cidr_vpc" {
  type = string

  validation {
    condition     = can(cidrhost(var.cidr_vpc, 0))
    error_message = "El valor de cidr_vpc debe ser un bloque CIDR válido, por ejemplo 10.0.0.0/16."
  }
}

variable "entorno" {
  type = string

  validation {
    condition     = contains(["dev", "test", "prod"], var.entorno)
    error_message = "El entorno debe ser uno de: dev, test, prod."
  }
}
```

La función ```can()``` es especialmente útil aquí: intenta evaluar una expresión y devuelve ```true```/```false``` según si lanza error o no, en lugar de propagar el error — ideal para validar formatos.

## PARTE IV — Terraform CLI (todos los comandos y parámetros)

La CLI de Terraform es la interfaz principal con la que interactúas día a día. En esta parte se explica cada comando en profundidad, junto con sus parámetros más relevantes.

### 1. terraform init

Inicializa un directorio de trabajo. Es **siempre el primer comando** que se ejecuta en un proyecto nuevo, o tras clonar uno existente.

```bash
terraform init
```

Qué hace exactamente:

1. Lee el bloque ```required_providers``` y descarga los plugins de provider necesarios (los guarda en ```.terraform/providers/```).
2. Configura el **backend** (dónde se guardará el estado — local por defecto, o remoto si está declarado).
3. Descarga los **módulos** referenciados (propios o de un registro).
4. Genera/actualiza el fichero ```.terraform.lock.hcl```, que fija las versiones exactas de los providers.

#### Parámetros más usados

```bash
terraform init -upgrade
```

Fuerza a buscar versiones más nuevas de providers y módulos, en lugar de respetar el lock file existente.

```bash
terraform init -reconfigure
```

Ignora la configuración de backend existente y la reconfigura desde cero (útil si cambias de backend, por ejemplo de local a S3).

```bash
terraform init -migrate-state
```

Como ```-reconfigure```, pero además **intenta migrar** el estado existente al nuevo backend, en lugar de empezar de cero.

```bash
terraform init -backend-config="bucket=mi-bucket-estado"
```

Permite pasar valores de configuración del backend desde la línea de comandos (útil en CI/CD, para no hardcodear el nombre del bucket en el código si cambia por entorno).

```bash
terraform init -input=false
```

Evita que Terraform pregunte de forma interactiva por valores que falten — imprescindible en pipelines automatizados.

### 2. terraform validate

Comprueba que la configuración es **sintácticamente correcta y coherente internamente** (tipos, referencias existentes...), **sin conectarse a AWS**.

```bash
terraform validate
```

Salida en caso de éxito:

```
Success! The configuration is valid.
```

Salida en caso de error:

```
Error: Reference to undeclared resource

  on main.tf line 12, in resource "aws_instance" "app":
  12:   subnet_id = aws_subnet.publica.id

A resource "aws_subnet" "publica" has not been declared.
```

Es el primer paso lógico de cualquier pipeline de CI (Parte VIII/IX): si ```validate``` falla, ni siquiera tiene sentido intentar un ```plan```.

```bash
terraform validate -json
```

Salida en formato JSON, pensada para ser consumida por otras herramientas (linters, pipelines).

### 3. terraform fmt

Reformatea automáticamente el código según el estilo canónico de HCL (indentación, alineación de ```=```, etc.).

```bash
terraform fmt
```

Parámetros:

```bash
terraform fmt -recursive
```

Aplica el formato a todos los subdirectorios (útil si tienes módulos locales en carpetas).

```bash
terraform fmt -check
```

No modifica nada, solo **comprueba** si algo estaría mal formateado y devuelve un código de salida distinto de 0 si es así. Se usa mucho en CI, como "gate" antes de permitir un merge.

```bash
terraform fmt -diff
```

Muestra el diff de qué cambiaría, sin aplicarlo.

### 4. terraform plan

Calcula qué cambios se aplicarían, **sin ejecutarlos**. Es el comando que más se usa en el día a día.

```bash
terraform plan
```

La salida usa un sistema de símbolos:

```
  + crear
  - destruir
  ~ modificar in-place
-/+ destruir y recrear
```

Ejemplo de salida:

```
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t3.micro"
      + id            = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

#### Parámetros más usados

bashterraform plan -out=plan.tfplan

Guarda el plan calculado en un fichero binario, para poder aplicarlo **exactamente** más tarde con ```terraform apply plan.tfplan``` — evita el riesgo de que algo cambie entre el ```plan``` y el ```apply``` (muy usado en CI/CD: un pipeline hace ```plan```, un humano lo revisa, y solo entonces se aplica ese plan exacto).

```bash
terraform plan -var="region=us-east-1"
```

Pasa una variable puntual desde la CLI.

```bash
terraform plan -var-file="produccion.tfvars"
```

Usa un fichero de variables concreto (útil para tener ```dev.tfvars```, ```produccion.tfvars```, etc.).

```bash
terraform plan -target=aws_instance.web
```

Limita el plan a un recurso concreto (y sus dependencias). **Uso ocasional**/**depuración**, no como práctica habitual, porque puede ocultar otros cambios pendientes.

```bash
terraform plan -destroy
```

Muestra qué se destruiría, sin ejecutarlo (equivalente a previsualizar un ```destroy```).

```bash
terraform plan -refresh=false
```

No sincroniza el estado contra la infraestructura real antes de calcular el plan (más rápido, pero puede dar un plan desactualizado si algo cambió fuera de Terraform).

### 5. terraform apply

Ejecuta los cambios, ya sea recalculando el plan en el momento o aplicando uno guardado previamente.

```bash
terraform apply
```

Calcula el plan, lo muestra, y pide confirmación explícita escribiendo ```yes```.

```bash
terraform apply plan.tfplan
```

Aplica un plan ya guardado con ```terraform plan -out=plan.tfplan```, **sin volver a preguntar** (porque ya fue revisado).

#### Parámetros más usados

```bash
terraform apply -auto-approve
```

Omite la confirmación interactiva. Imprescindible en pipelines de CI/CD, pero se usa con cautela en local (fácil de aplicar algo por error).

```bash
terraform apply -var-file="produccion.tfvars"
```

Igual que en ```plan```.

```bash
terraform apply -parallelism=5
```

Limita cuántas operaciones puede ejecutar Terraform en paralelo (por defecto 10). Útil si una API tiene límites de tasa (rate limiting) estrictos.

```bash
terraform apply -replace="aws_instance.web"
```

Fuerza la destrucción y recreación de un recurso concreto en este apply, sin tener que cambiar su configuración (sustituye al antiguo ```terraform taint```).

### 6. terraform destroy

Elimina **todos** los recursos gestionados por la configuración actual.

```bash
terraform destroy
```

```bash
terraform destroy -target=aws_instance.web
```

Destruye solo ese recurso (y lo que dependa exclusivamente de él). Uso puntual, no habitual.

```bash
terraform destroy -auto-approve
```

Sin confirmación **— especialmente peligroso**, se usa casi exclusivamente en entornos efímeros de CI (por ejemplo, tras tests de integración).

### 7. terraform import

Incorpora al **estado** un recurso que ya existe en AWS pero que no fue creado por Terraform (por ejemplo, algo creado manualmente hace tiempo).

```bash
terraform import aws_instance.web i-0abcd1234efgh5678
```

Esto asocia el recurso declarado en tu código (```aws_instance.web```, que debes haber escrito de antemano con una configuración que se aproxime a la real) con la instancia real ```i-0abcd1234efgh5678```.

**Importante**: ```import``` solo actualiza el **estado**; no genera el código HCL por ti (en versiones modernas de Terraform existe el bloque ```import {}``` declarativo, que sí puede generar código junto con ```terraform plan -generate-config-out```, pero el comando clásico ```terraform import``` no lo hace).

```bash
terraform import 'aws_instance.app["web"]' i-0abcd1234efgh5678
```

Sintaxis para importar a un recurso definido con ```for_each``` (nótese las comillas para que la shell no interprete los corchetes).

### 8. terraform taint / untaint

**Nota histórica**: estos comandos están **obsoletos** desde Terraform 0.15+ a favor de ```terraform apply -replace=..```. (visto en el punto 5), pero siguen documentados porque aparecen en proyectos y tutoriales antiguos.

```bash
terraform taint aws_instance.web
```

Marca el recurso para que sea destruido y recreado en el próximo ```apply```, sin cambiar su configuración (útil si sospechas que algo "no está sano" y quieres forzar su recreación).

```bash
terraform untaint aws_instance.web
```

Revierte la marca anterior, si te arrepientes antes de aplicar.

### 9. terraform graph

Genera una representación del **grafo de dependencias** entre recursos, en formato DOT (Graphviz).

```bash
terraform graph
```

```
digraph {
  "aws_instance.web" -> "aws_subnet.publica"
  "aws_subnet.publica" -> "aws_vpc.principal"
}
```

Para visualizarlo como imagen (requiere tener Graphviz instalado):

```bash
terraform graph | dot -Tpng > grafo.png
```

Es especialmente útil para ```entender proyectos grandes``` que no escribiste tú, o para depurar por qué Terraform quiere aplicar cambios en un orden inesperado.

### 10. terraform output

Muestra los valores definidos como ```output``` del módulo raíz, **después de un apply**.

```bash
terraform output
```

```
ip_publica = "34.201.XX.XX"
id_vpc     = "vpc-0abc1234"
```

```bash
terraform output ip_publica
```

Muestra solo ese output concreto.

```bash
terraform output -json
```

Formato JSON, pensado para ser consumido por scripts (por ejemplo, para pasar la IP resultante a otro proceso de un pipeline).

```bash
terraform output -raw ip_publica
```

Devuelve el valor **sin comillas ni formato JSON**, ideal para usarlo directamente en un script bash:

```bash
IP=$(terraform output -raw ip_publica)
ssh ec2-user@$IP
```

### 11. terraform state

Conjunto de subcomandos para **inspeccionar y manipular el estado directamente**. Se usan con cautela — modificar el estado a mano puede desincronizarlo de la realidad si no se hace correctamente (se profundiza en la Parte VI, "State Surgery").

```bash
terraform state list
```

Lista todos los recursos que hay actualmente en el estado.

```
aws_instance.web
aws_vpc.principal
aws_subnet.publica
```

```bash
terraform state show aws_instance.web
```

Muestra todos los atributos actuales de ese recurso concreto, tal como están guardados en el estado.

```bash
terraform state mv aws_instance.web aws_instance.servidor_principal
```

Renombra un recurso **dentro del estado**, sin destruirlo ni recrearlo en AWS — típico al refactorizar código (por ejemplo, mover un recurso a un módulo).

```bash
terraform state rm aws_instance.web
```

Elimina el recurso **del estado**, pero **no lo destruye en AWS**. Terraform simplemente "olvida" que lo gestionaba. Útil si quieres dejar de gestionar algo con Terraform sin borrarlo.

```bash
terraform state pull > estado.json
```

Descarga el estado actual (incluso si es remoto) y lo vuelca a un fichero local, en formato JSON, para inspección manual.

```bash
terraform state push estado.json
```

Sube un fichero de estado local como el nuevo estado remoto. **Muy peligroso** si no sabes exactamente lo que haces — puede sobrescribir el estado real con uno desactualizado.

### 12. terraform workspace

Permite gestionar **múltiples estados aislados** dentro de la misma configuración de código, útil para separar entornos ligeros (dev/test) sin duplicar ficheros ```.tf```.

```bash
terraform workspace list
```

```
* default
  dev
  produccion
```

(El asterisco marca el workspace activo.)

```bash
terraform workspace new dev
```

Crea un nuevo workspace (y cambia a él automáticamente).

```bash
terraform workspace select produccion
```

Cambia al workspace indicado.

```bash
terraform workspace show
```

Muestra el nombre del workspace activo actualmente.

Dentro del código, puedes referenciar el workspace activo con ```terraform.workspace```:

```hcl
resource "aws_instance" "app" {
  instance_type = terraform.workspace == "produccion" ? "t3.large" : "t3.micro"

  tags = {
    Entorno = terraform.workspace
  }
}
```

**Nota importante sobre workspaces**: son útiles para variaciones ligeras (mismo código, distinto tamaño de instancia), pero ```no sustituyen``` una separación real de entornos por carpetas/backends distintos cuando dev y producción tienen configuraciones muy diferentes o cuando quieres aislar completamente el blast radius de un error. Esta discusión se retoma en la Parte X (Buenas prácticas → Patrones).
