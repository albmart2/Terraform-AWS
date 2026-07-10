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

## PARTE V — AWS (completa)

### IAM (Identity and Access Management)

IAM es el servicio que controla **quién puede hacer qué** dentro de una cuenta AWS. Es la base de todo lo demás: sin entender IAM, es fácil crear infraestructura insegura sin darse cuenta.

#### Usuarios

Un **usuario IAM** representa una identidad con credenciales propias y permanentes (access key + secret key, o contraseña para la consola).

```hcl
resource "aws_iam_user" "desarrollador" {
  name = "ana-desarrolladora"

  tags = {
    Departamento = "Backend"
  }
}

resource "aws_iam_access_key" "desarrollador_key" {
  user = aws_iam_user.desarrollador.name
}
```

**Importante**: ```aws_iam_access_key``` genera un secret que queda en el estado de Terraform. En proyectos reales, se recomienda **no gestionar usuarios humanos vía Terraform** cuando sea posible (usar SSO/Identity Center en su lugar), reservando IAM de Terraform sobre todo para roles y permisos de servicios.

#### Roles

Un **rol IAM** es una identidad **sin credenciales fijas*, que puede ser "asumida" temporalmente por un servicio de AWS (como EC2, Lambda) o por otro usuario/cuenta.

```hcl
resource "aws_iam_role" "rol_ec2" {
  name = "rol-ec2-app"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}
```

El bloque ```assume_role_policy``` (llamado trust policy) define **quién puede asumir este rol** — en este caso, el propio servicio EC2. Esto permite, por ejemplo, que una instancia EC2 tenga permisos para leer de S3 sin necesidad de guardar credenciales dentro de la instancia.

#### Policies

Una **policy** es un documento JSON que define permisos: qué acciones están permitidas (o denegadas) sobre qué recursos.

```hcl
resource "aws_iam_policy" "acceso_s3_logs" {
  name = "acceso-s3-logs"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::mi-app-logs-2026/*"
      }
    ]
  })
}
```

Estructura de un statement:

- **Effect**: ```Allow``` o ```Deny```.
- **Action**: qué operaciones de la API (con comodines posibles, ```s3:*```).
- **Resource**: sobre qué recursos concretos (ARNs), con comodines posibles.
- **Condition** (opcional): restricciones adicionales (por IP origen, por etiqueta, por hora...).

#### Permissions (vincular policy con identidad)

Una policy, por sí sola, no hace nada — debe **adjuntarse** a un usuario, grupo o rol:

```hcl
resource "aws_iam_role_policy_attachment" "adjuntar" {
  role       = aws_iam_role.rol_ec2.name
  policy_arn = aws_iam_policy.acceso_s3_logs.arn
}
```

Para que una instancia EC2 use realmente ese rol, se necesita además un **Instance Profile**:

```hcl
resource "aws_iam_instance_profile" "perfil_ec2" {
  name = "perfil-ec2-app"
  role = aws_iam_role.rol_ec2.name
}

resource "aws_instance" "app" {
  ami                  = "ami-0c55b159cbfafe1f0"
  instance_type        = "t3.micro"
  iam_instance_profile = aws_iam_instance_profile.perfil_ec2.name
}
```

Con esto, cualquier código que corra dentro de esa instancia puede leer/escribir en el bucket ```mi-app-logs-2026``` **sin necesitar credenciales explícitas** — las obtiene automáticamente del servicio de metadata (visto más abajo).

#### Least Privilege (mínimo privilegio)

Es el principio de dar solo los permisos estrictamente necesarios, ni uno más. En la práctica:

- Evita ```Action = "*"``` y ```Resource = "*"``` salvo que sea genuinamente necesario (y casi nunca lo es).
- Prefiere policies específicas por servicio y recurso concreto, en lugar de policies gestionadas amplias como ```AmazonS3FullAccess```.
- Usa ```Condition``` para acotar aún más (por ejemplo, solo permitir acceso desde una VPC concreta).

```hcl
# Mal: demasiado permisivo
resource "aws_iam_policy" "malo" {
  policy = jsonencode({
    Statement = [{ Effect = "Allow", Action = "*", Resource = "*" }]
  })
}

# Bien: acotado a lo necesario
resource "aws_iam_policy" "bueno" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "arn:aws:s3:::mi-app-logs-2026/lecturas/*"
    }]
  })
}
```

### EC2 (Elastic Compute Cloud)

EC2 son las máquinas virtuales de AWS. Es probablemente el servicio más usado en cualquier curso de introducción a la nube.

#### AMI (Amazon Machine Image)

Una AMI es la **imagen base** (sistema operativo + software preinstalado + configuración) desde la que se lanza una instancia. En vez de hardcodear el ID de una AMI (que cambia según la región y se actualiza con el tiempo), es buena práctica **buscarla dinámicamente** con un ```data source```:

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}
```

Esto garantiza que siempre se use la AMI de Amazon Linux más reciente disponible, sin tener que actualizar el ID manualmente cada pocos meses.

#### Tipos de instancia

Los tipos de instancia definen la combinación de CPU, memoria, red y almacenamiento. Se agrupan en familias:

|Familia|Enfoque|Ejemplo de uso|
|-------|-------|--------------|
|```t (t3, t4g)```|Uso general, con "burst" de CPU|Webs pequeñas, entornos dev|
|```m (m5, m6i)```|Uso general balanceado|Aplicaciones estándar de producción|
|```c (c5, c6i)```|Optimizado a CPU|Procesamiento intensivo, colas|
|```r (r5, r6i)```|Optimizado a memoria|Bases de datos en memoria, caché|
|```i (i3)```|Almacenamiento local NVMe|Bases de datos con alto I/O|

```hcl
variable "tipo_instancia" {
  type    = string
  default = "t3.micro"

  validation {
    condition     = can(regex("^[a-z][0-9][a-z]?\\.", var.tipo_instancia))
    error_message = "Debe ser un tipo de instancia EC2 válido, por ejemplo t3.micro."
  }
}
```

####EBS (Elastic Block Store)

Discos de bloque persistentes que se adjuntan a una instancia. El volumen "raíz" se puede configurar directamente en el recurso ```aws_instance```:

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }
}
```

Para discos **adicionales** (separados del root):

```hcl
resource "aws_ebs_volume" "datos" {
  availability_zone = aws_instance.app.availability_zone
  size              = 100
  type              = "gp3"
  encrypted         = true
}

resource "aws_volume_attachment" "adjuntar_datos" {
  device_name = "/dev/sdh"
  volume_id   = aws_ebs_volume.datos.id
  instance_id = aws_instance.app.id
}
```

#### Key Pairs

Par de claves SSH para acceso remoto seguro. La práctica recomendada es **generar la clave privada localmente** y subir solo la pública a AWS:

```bash
ssh-keygen -t ed25519 -f clave-curso -C "curso-terraform"
```

```hcl
resource "aws_key_pair" "curso" {
  key_name   = "clave-curso"
  public_key = file("${path.module}/clave-curso.pub")
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  key_name      = aws_key_pair.curso.key_name
}
```

```clave-curso``` (la privada) **nunca** debe subirse a Git ni gestionarse vía Terraform — solo la pública.

#### Elastic IP

Una IP pública **fija**, que se puede reasignar entre instancias (a diferencia de la IP pública "normal" de una instancia, que cambia si esta se detiene y arranca de nuevo).

```hcl
resource "aws_eip" "ip_fija" {
  instance = aws_instance.app.id
  domain   = "vpc"
}

output "ip_publica_fija" {
  value = aws_eip.ip_fija.public_ip
}
```

#### User Data

Script que se ejecuta **automáticamente** la primera vez que arranca la instancia (bootstrapping) — muy usado para instalar software sin tener que conectarse manualmente por SSH.

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    dnf update -y
    dnf install -y nginx
    systemctl enable nginx
    systemctl start nginx
    echo "<h1>Desplegado con Terraform</h1>" > /usr/share/nginx/html/index.html
  EOF
}
```

**Nota**: si cambias el contenido de ```user_data``` en una instancia ya creada, Terraform lo actualiza en el estado, pero **AWS no vuelve a ejecutar el script automáticamente** en una instancia ya arrancada — solo se ejecuta en el primer arranque. Para forzar la recreación cuando cambia el script, se suele combinar con ```user_data_replace_on_change = true```:

```hcl
resource "aws_instance" "app" {
  # ...
  user_data_replace_on_change = true
}
```

#### Metadata

Cada instancia EC2 expone un servicio interno de metadata, accesible **solo desde dentro** de la propia instancia, en ```http://169.254.169.254```. Permite que un script (o la propia aplicación) obtenga información sobre sí misma sin necesidad de credenciales:

```bash
# Desde dentro de la instancia (IMDSv2, el método seguro actual):
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

Terraform permite forzar el uso de **IMDSv2** (más seguro que la v1, que es vulnerable a ciertos ataques de SSRF) directamente en la definición de la instancia:

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  metadata_options {
    http_tokens                = "required"  # fuerza IMDSv2
    http_endpoint               = "enabled"
    http_put_response_hop_limit = 1
  }
}
```

Este es exactamente el tipo de configuración que herramientas como ```tfsec```/```checkov``` comprueban automáticamente, marcando como vulnerable cualquier instancia sin ```http_tokens = "required"```.

##### VPC y CIDR

Una **VPC** (Virtual Private Cloud) es tu red privada y aislada dentro de AWS. Todo lo demás (instancias, bases de datos, balanceadores) vive dentro de una VPC.

El **CIDR** define el rango de direcciones IP disponible para la red, en notación ```IP/máscara```:

```hcl
resource "aws_vpc" "principal" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "vpc-principal"
  }
}
```

```10.0.0.0/16``` significa que los primeros 16 bits son fijos (10.0), dejando 65.536 direcciones IP disponibles (10.0.0.0 – 10.0.255.255) para repartir entre subnets.

#### Subnets

Subdivisiones de la VPC, cada una asociada a una **zona de disponibilidad** concreta. Se distingue entre subnets **públicas** (con salida directa a internet) y **privadas*+ (sin ella).

```hcl
resource "aws_subnet" "publica_a" {
  vpc_id                  = aws_vpc.principal.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "eu-west-1a"
  map_public_ip_on_launch = true

  tags = { Name = "subnet-publica-a" }
}

resource "aws_subnet" "privada_a" {
  vpc_id            = aws_vpc.principal.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "eu-west-1a"

  tags = { Name = "subnet-privada-a" }
}
```

Usando ```cidrsubnet()``` se puede calcular esto dinámicamente en vez de escribirlo a mano:

```hcl
resource "aws_subnet" "publica" {
  for_each          = toset(["eu-west-1a", "eu-west-1b"])
  vpc_id            = aws_vpc.principal.id
  cidr_block        = cidrsubnet(aws_vpc.principal.cidr_block, 8, index(["eu-west-1a", "eu-west-1b"], each.value))
  availability_zone = each.value
}
```

#### Routing

Una **tabla de rutas** determina hacia dónde sale el tráfico de una subnet.

```hcl
resource "aws_route_table" "publica" {
  vpc_id = aws_vpc.principal.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.principal.id
  }

  tags = { Name = "rt-publica" }
}

resource "aws_route_table_association" "publica_a" {
  subnet_id      = aws_subnet.publica_a.id
  route_table_id = aws_route_table.publica.id
}
```

#### Internet Gateway

Componente que da salida (y entrada) a internet a una VPC:

```hcl
resource "aws_internet_gateway" "principal" {
  vpc_id = aws_vpc.principal.id
  tags   = { Name = "igw-principal" }
}
```

#### NAT Gateway

Permite que recursos en subnets **privadas** salgan a internet (por ejemplo, para descargar actualizaciones) **sin ser accesibles desde fuera**:

```hcl
resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "principal" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.publica_a.id  # el NAT vive en una subnet pública

  tags = { Name = "nat-principal" }
}

resource "aws_route_table" "privada" {
  vpc_id = aws_vpc.principal.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.principal.id
  }
}

resource "aws_route_table_association" "privada_a" {
  subnet_id      = aws_subnet.privada_a.id
  route_table_id = aws_route_table.privada.id
}
```

**Coste importante a tener en cuenta**: un NAT Gateway tiene coste por hora y por GB transferido — es habitual que sea una de las partidas de coste más altas de una cuenta AWS pequeña. Se retoma en la Parte X (Costes).

#### Security Groups

Firewall a nivel de **instancia**, con estado (*stateful*: si permites tráfico de entrada, la respuesta de salida se permite automáticamente).

```hcl
resource "aws_security_group" "web" {
  name   = "sg-web"
  vpc_id = aws_vpc.principal.id

  ingress {
    description = "HTTP desde cualquier origen"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH solo desde mi IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.10/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

#### Network ACL

Firewall a nivel de **subnet**, sin estado (*stateless*: hay que permitir explícitamente entrada Y salida por separado). Es una capa adicional, más rígida, que complementa (no sustituye) a los Security Groups:

```hcl
resource "aws_network_acl" "publica" {
  vpc_id     = aws_vpc.principal.id
  subnet_ids = [aws_subnet.publica_a.id]

  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  egress {
    rule_no    = 100
    protocol   = "-1"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
}
```

#### VPC Endpoints

Permiten acceder a servicios de AWS (como S3 o DynamoDB) **sin salir a internet**, mejorando seguridad y, a menudo, reduciendo coste de NAT:

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.principal.id
  service_name = "com.amazonaws.eu-west-1.s3"
  route_table_ids = [aws_route_table.privada.id]
}
```

#### Peering

Conexión directa entre dos VPCs (misma cuenta u otra), como si fueran una sola red a efectos de enrutamiento:

```hcl
resource "aws_vpc_peering_connection" "con_otra_vpc" {
  vpc_id      = aws_vpc.principal.id
  peer_vpc_id = "vpc-0123456789abcdef0"
  auto_accept = true
}
```

#### Transit Gateway

Un "hub" central para conectar **muchas** VPCs (y redes on-premise vía VPN/Direct Connect) sin tener que crear peering entre cada par de VPCs (que crece de forma combinatoria):

```hcl
resource "aws_ec2_transit_gateway" "principal" {
  description = "TGW central de la organización"
}

resource "aws_ec2_transit_gateway_vpc_attachment" "principal" {
  transit_gateway_id = aws_ec2_transit_gateway.principal.id
  vpc_id             = aws_vpc.principal.id
  subnet_ids         = [aws_subnet.privada_a.id]
}
```

### Balanceadores

#### ALB (Application Load Balancer)

Balanceador de **capa 7** (HTTP/HTTPS), con capacidad de enrutar según ruta, cabecera o dominio:

```hcl
resource "aws_lb" "principal" {
  name               = "alb-principal"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web.id]
  subnets            = [aws_subnet.publica_a.id, aws_subnet.publica_b.id]
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.principal.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

#### NLB (Network Load Balancer)

Balanceador de **capa 4** (TCP/UDP), pensado para altísimo rendimiento y baja latencia, o protocolos no HTTP:

```hcl
resource "aws_lb" "nlb" {
  name               = "nlb-principal"
  internal           = false
  load_balancer_type = "network"
  subnets            = [aws_subnet.publica_a.id]
}
```

#### Target Groups

Conjunto de destinos (instancias, IPs o Lambdas) a los que el balanceador dirige tráfico:

```hcl
resource "aws_lb_target_group" "app" {
  name     = "tg-app"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.principal.id

  health_check {
    path                = "/salud"
    interval            = 30
    healthy_threshold   = 2
    unhealthy_threshold = 3
    matcher             = "200"
  }
}

resource "aws_lb_target_group_attachment" "app" {
  target_group_arn = aws_lb_target_group.app.arn
  target_id        = aws_instance.app.id
  port             = 80
}
```

#### Health Checks

Ya vistos arriba dentro del ```target_group```: comprobaciones periódicas para determinar si un destino puede seguir recibiendo tráfico. Si falla el número de veces indicado en ```unhealthy_threshold```, el balanceador deja de enviarle tráfico hasta que vuelva a responder correctamente ```healthy_threshold``` veces seguidas.

### Auto Scaling

#### Launch Templates

Plantilla que define cómo lanzar cada instancia del grupo de autoescalado (equivalente reutilizable a ```aws_instance```, pero pensado para ser referenciado por un Auto Scaling Group):

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "lt-app-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = base64encode(<<-EOF
    #!/bin/bash
    dnf install -y nginx
    systemctl enable --now nginx
  EOF
  )
}
```

#### Auto Scaling Group + Scaling Policies

```hcl
resource "aws_autoscaling_group" "app" {
  desired_capacity   = 2
  min_size           = 1
  max_size           = 5
  vpc_zone_identifier = [aws_subnet.privada_a.id]
  target_group_arns  = [aws_lb_target_group.app.arn]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "asg-app"
    propagate_at_launch = true
  }
}

resource "aws_autoscaling_policy" "escalar_por_cpu" {
  name                   = "escalar-cpu"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 60.0
  }
}
```

Esta política de tipo ```TargetTrackingScaling``` mantiene automáticamente el uso medio de CPU del grupo cerca del 60%, añadiendo o quitando instancias según haga falta — sin tener que definir manualmente umbrales de CloudWatch por separado.

### S3 (Simple Storage Service)

#### Buckets

```hcl
resource "aws_s3_bucket" "app" {
  bucket = "mi-app-datos-2026"
}
```

#### Versioning

```hcl
resource "aws_s3_bucket_versioning" "app" {
  bucket = aws_s3_bucket.app.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

Con versionado activo, sobrescribir o borrar un objeto no lo elimina realmente — crea una nueva versión o un "delete marker", permitiendo recuperar versiones anteriores.

#### Lifecycle

Reglas para mover objetos a almacenamiento más barato, o eliminarlos, según su antigüedad:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "app" {
  bucket = aws_s3_bucket.app.id

  rule {
    id     = "archivar-antiguos"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}
```

#### Encryption

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "app" {
  bucket = aws_s3_bucket.app.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
    bucket_key_enabled = true
  }
}
```

#### Replication

Copia automática de objetos hacia otro bucket, típicamente en otra región (requiere versionado activo en ambos buckets y un rol IAM con permisos):

```hcl
resource "aws_s3_bucket_replication_configuration" "app" {
  role   = aws_iam_role.replicacion.arn
  bucket = aws_s3_bucket.app.id

  rule {
    id     = "replicar-todo"
    status = "Enabled"

    destination {
      bucket        = "arn:aws:s3:::mi-app-datos-2026-replica"
      storage_class = "STANDARD"
    }
  }

  depends_on = [aws_s3_bucket_versioning.app]
}
```

#### Hosting Web

```hcl
resource "aws_s3_bucket_website_configuration" "app" {
  bucket = aws_s3_bucket.app.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}
```

Nota: para servir contenido públicamente hoy en día se recomienda combinar S3 con **CloudFront delante** (visto más abajo), en vez de exponer el hosting web de S3 directamente, por seguridad y rendimiento.

### RDS (Relational Database Service)

```hcl
resource "aws_db_subnet_group" "principal" {
  name       = "db-subnet-group"
  subnet_ids = [aws_subnet.privada_a.id, aws_subnet.privada_b.id]
}

resource "aws_db_instance" "principal" {
  identifier             = "app-db"
  engine                 = "postgres"
  engine_version         = "16.3"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  db_name                = "appdb"
  username               = "admin"
  password               = var.password_db  # marcado sensitive en variables.tf
  db_subnet_group_name   = aws_db_subnet_group.principal.name
  vpc_security_group_ids = [aws_security_group.db.id]

  backup_retention_period = 7
  skip_final_snapshot     = false
  final_snapshot_identifier = "app-db-final-snapshot"
}
```

#### Aurora

Motor propio de AWS, compatible con MySQL/PostgreSQL, con arquitectura de almacenamiento distribuido separada del cómputo:

```hcl
resource "aws_rds_cluster" "aurora" {
  cluster_identifier = "app-aurora"
  engine             = "aurora-postgresql"
  engine_version     = "16.2"
  master_username    = "admin"
  master_password    = var.password_db
  database_name      = "appdb"

  db_subnet_group_name = aws_db_subnet_group.principal.name
}

resource "aws_rds_cluster_instance" "aurora_instancias" {
  count              = 2
  cluster_identifier = aws_rds_cluster.aurora.id
  instance_class     = "db.r6g.large"
  engine             = aws_rds_cluster.aurora.engine
}
```

#### Snapshots y Backups

```hcl
resource "aws_db_snapshot" "manual" {
  db_instance_identifier = aws_db_instance.principal.identifier
  db_snapshot_identifier = "snapshot-manual-2026"
}
```

Los backups automáticos se configuran con ```backup_retention_period``` (visto arriba) y ```backup_window```, sin necesidad de un recurso separado.

#### Subnet Groups

Ya visto arriba (```aws_db_subnet_group```): define en qué subnets puede desplegarse la base de datos — normalmente subnets privadas, sin acceso directo desde internet.

### Otros servicios

#### DynamoDB

Base de datos NoSQL clave-valor/documentos, totalmente gestionada:

```hcl
resource "aws_dynamodb_table" "sesiones" {
  name         = "sesiones"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id_sesion"

  attribute {
    name = "id_sesion"
    type = "S"
  }

  ttl {
    attribute_name = "expira_en"
    enabled        = true
  }
}
```

#### Lambda

Cómputo serverless: código que se ejecuta en respuesta a eventos, sin gestionar servidores.

```hcl
resource "aws_lambda_function" "procesador" {
  function_name = "procesador-eventos"
  role          = aws_iam_role.rol_lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  filename      = "lambda.zip"
  source_code_hash = filebase64sha256("lambda.zip")
}
```

#### API Gateway

Expone APIs HTTP que pueden invocar Lambda u otros backends:

```hcl
resource "aws_apigatewayv2_api" "principal" {
  name          = "api-app"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  api_id             = aws_apigatewayv2_api.principal.id
  integration_type   = "AWS_PROXY"
  integration_uri    = aws_lambda_function.procesador.invoke_arn
}

resource "aws_apigatewayv2_route" "default" {
  api_id    = aws_apigatewayv2_api.principal.id
  route_key = "POST /procesar"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}
```

#### Route53

DNS gestionado:

```hcl
resource "aws_route53_zone" "principal" {
  name = "miapp.com"
}

resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.principal.zone_id
  name    = "www.miapp.com"
  type    = "A"

  alias {
    name                   = aws_lb.principal.dns_name
    zone_id                = aws_lb.principal.zone_id
    evaluate_target_health = true
  }
}
```

#### CloudFront

CDN de AWS:

```hcl
resource "aws_cloudfront_distribution" "app" {
  enabled = true

  origin {
    domain_name = aws_s3_bucket.app.bucket_regional_domain_name
    origin_id   = "s3-app"
  }

  default_cache_behavior {
    target_origin_id       = "s3-app"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods          = ["GET", "HEAD"]

    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

#### CloudWatch

Monitorización, métricas, logs y alarmas:

```hcl
resource "aws_cloudwatch_metric_alarm" "cpu_alta" {
  alarm_name          = "cpu-alta-app"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80

  dimensions = {
    InstanceId = aws_instance.app.id
  }
}
```

#### Secrets Manager

```hcl
resource "aws_secretsmanager_secret" "password_db" {
  name = "app/password-db"
}

resource "aws_secretsmanager_secret_version" "password_db" {
  secret_id     = aws_secretsmanager_secret.password_db.id
  secret_string = var.password_db
}
```

Incluye rotación automática configurable, algo que Parameter Store no ofrece de forma nativa.

#### Parameter Store

Alternativa más ligera (y gratuita hasta cierto límite) dentro de Systems Manager, para configuración y secretos sencillos:

```hcl
resource "aws_ssm_parameter" "nivel_log" {
  name  = "/app/nivel-log"
  type  = "String"
  value = "INFO"
}

resource "aws_ssm_parameter" "password_db" {
  name  = "/app/password-db"
  type  = "SecureString"
  value = var.password_db
}
```

#### ECS (Elastic Container Service)

Orquestador de contenedores nativo de AWS:

```hcl
resource "aws_ecs_cluster" "principal" {
  name = "cluster-app"
}

resource "aws_ecs_task_definition" "app" {
  family                   = "app"
  requires_compatibilities  = ["FARGATE"]
  network_mode              = "awsvpc"
  cpu                       = "256"
  memory                    = "512"
  execution_role_arn        = aws_iam_role.rol_ec2.arn

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${aws_ecr_repository.app.repository_url}:latest"
      portMappings = [{ containerPort = 80 }]
    }
  ])
}

resource "aws_ecs_service" "app" {
  name            = "servicio-app"
  cluster         = aws_ecs_cluster.principal.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = [aws_subnet.privada_a.id]
    security_groups = [aws_security_group.web.id]
  }
}
```

#### ECR (Elastic Container Registry)

Registro privado de imágenes de contenedor:

```hcl
resource "aws_ecr_repository" "app" {
  name                 = "app"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}
```

#### EKS (Elastic Kubernetes Service)

Kubernetes gestionado por AWS:

```hcl
resource "aws_eks_cluster" "principal" {
  name     = "cluster-k8s"
  role_arn = aws_iam_role.rol_eks.arn
  version  = "1.30"

  vpc_config {
    subnet_ids = [aws_subnet.privada_a.id, aws_subnet.privada_b.id]
  }
}

resource "aws_eks_node_group" "principal" {
  cluster_name    = aws_eks_cluster.principal.name
  node_group_name = "nodos-principales"
  node_role_arn   = aws_iam_role.rol_nodos_eks.arn
  subnet_ids      = [aws_subnet.privada_a.id]

  scaling_config {
    desired_size = 2
    min_size     = 1
    max_size     = 4
  }
}
```

## PARTE VI — Terraform Avanzado

Con AWS ya cubierto en profundidad, esta parte vuelve a Terraform en sí mismo: cómo organizar, reutilizar y gestionar en equipo configuraciones que crecen más allá de un único fichero.

### 1. Modules

Un **módulo** es un conjunto de ficheros ```.tf``` que se agrupan y se reutilizan como si fueran una "función" de infraestructura: recibe variables de entrada y expone outputs.

**Todo proyecto de Terraform es, técnicamente, un módulo** — el directorio raíz es el "módulo raíz". Lo que llamamos "módulos" en la práctica son módulos **hijos**, invocados desde otro sitio.

#### Estructura de un módulo

```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

```hcl
# modules/vpc/variables.tf
variable "cidr_block" {
  type = string
}

variable "nombre" {
  type = string
}
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "principal" {
  cidr_block = var.cidr_block
  tags       = { Name = var.nombre }
}

resource "aws_subnet" "publica" {
  vpc_id     = aws_vpc.principal.id
  cidr_block = cidrsubnet(var.cidr_block, 8, 1)
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.principal.id
}

output "subnet_publica_id" {
  value = aws_subnet.publica.id
}
```

#### Uso del módulo desde el módulo raíz

```hcl
# main.tf (raíz)
module "red_produccion" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  nombre     = "vpc-produccion"
}

module "red_desarrollo" {
  source     = "./modules/vpc"
  cidr_block = "10.1.0.0/16"
  nombre     = "vpc-desarrollo"
}

resource "aws_instance" "app" {
  subnet_id = module.red_produccion.subnet_publica_id
  # ...
}
```

Con un único módulo, se crean dos **VPCs distintas** (producción y desarrollo) sin duplicar código.

#### Módulos de un registro (público o privado)

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"

  name = "vpc-app"
  cidr = "10.0.0.0/16"

  azs             = ["eu-west-1a", "eu-west-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
}
```

El registro público de Terraform (```registry.terraform.io```) tiene módulos verificados y mantenidos por la comunidad para los patrones más comunes (VPC, EKS, RDS...), que ahorran tener que reescribir configuraciones estándar desde cero.

#### 2. Remote State y Backend

Por defecto, el estado se guarda en un fichero local (```terraform.tfstate```) en tu propio ordenador. Esto es un problema en equipo: si dos personas aplican cambios desde su propia máquina, cada una tiene una copia distinta del estado, y se pisan entre sí.

Un **backend remoto** resuelve esto guardando el estado en un lugar centralizado y compartido:

```hcl
terraform {
  backend "s3" {
    bucket         = "mi-empresa-terraform-state"
    key            = "app/produccion/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

Con esto:

- El estado vive en S3, no en tu disco.
- Cualquiera del equipo con permisos puede ejecutar ```terraform plan```/```apply``` y ver el estado real y actualizado.
- ```dynamodb_table``` añade **bloqueo**, para que dos personas no apliquen a la vez.

#### 3. Outputs

Ya vistos en módulos, pero como recordatorio de su uso en el módulo raíz: exponen valores tras un ```apply```, consultables con ```terraform output``` o consumibles por otro proyecto vía ```terraform_remote_state```.

```hcl
output "ip_publica_app" {
  description = "IP pública de la instancia principal"
  value       = aws_instance.app.public_ip
}
```

### 4. Locals

Valores calculados o alias internos, para no repetir la misma expresión en varios sitios ni ensuciar el código con lógica repetida:

```hcl
locals {
  nombre_proyecto = "mi-app"
  entorno         = terraform.workspace

  nombre_completo = "${local.nombre_proyecto}-${local.entorno}"

  tags_comunes = {
    Proyecto = local.nombre_proyecto
    Entorno  = local.entorno
    GestionadoPor = "terraform"
  }
}

resource "aws_instance" "app" {
  # ...
  tags = merge(local.tags_comunes, { Name = local.nombre_completo })
}

resource "aws_s3_bucket" "app" {
  bucket = "${local.nombre_completo}-datos"
  tags   = local.tags_comunes
}
```

A diferencia de las ```variable```, los ```locals``` **no se pueden sobreescribir desde fuera** (ni por CLI, ni por ```.tfvars```) — son puramente internos al módulo.

### 5. Data Sources

Consultas de **solo lectura** a recursos que ya existen — propios (creados por otro proyecto Terraform) o de terceros (creados manualmente, o gestionados por otro equipo):

```hcl
data "aws_vpc" "existente" {
  filter {
    name   = "tag:Name"
    values = ["vpc-produccion"]
  }
}

data "aws_subnets" "privadas" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existente.id]
  }

  tags = {
    Tipo = "privada"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.aws_subnets.privadas.ids[0]
  # ...
}
```

Ya usamos data sources antes sin nombrarlos como tal: ```data "aws_ami"``` es exactamente esto — una consulta de solo lectura a AWS, sin gestionar el ciclo de vida de lo consultado.

### 6. terraform_remote_state (consumir el estado de otro proyecto)

Un caso especial de data source que permite leer los **outputs** de otro proyecto Terraform, típicamente cuando la red la gestiona un equipo/repositorio y la aplicación la gestiona otro:

```hcl
data "terraform_remote_state" "red" {
  backend = "s3"
  config = {
    bucket = "mi-empresa-terraform-state"
    key    = "red/produccion/terraform.tfstate"
    region = "eu-west-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.red.outputs.subnet_privada_id
}
```

Esto conecta dos proyectos Terraform completamente independientes (con sus propios ciclos de ```plan```/```apply```) a través del estado, sin necesidad de módulos compartidos.

### 7. Provisioners

Mecanismo para ejecutar acciones (típicamente scripts) **tras crear** un recurso. HashiCorp los describe explícitamente como "**último recurso**", porque rompen el modelo declarativo de Terraform: introducen pasos imperativos y no se pueden planificar de antemano de forma fiable.

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  key_name      = aws_key_pair.curso.key_name

  provisioner "remote-exec" {
    inline = [
      "sudo dnf install -y nginx",
      "sudo systemctl start nginx"
    ]

    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("clave-curso")
      host        = self.public_ip
    }
  }
}
```

**Por qué evitarlos cuando sea posible**: si el script falla a mitad, el recurso queda en un estado ambiguo (¿se creó pero no se configuró?), y Terraform no puede "recalcular" bien un plan futuro. Alternativas preferibles casi siempre: ```user_data```, o herramientas de configuración dedicadas como Ansible.

### 8. Lifecycle

Meta-argumento disponible en **cualquier resource**, para controlar comportamientos especiales de su ciclo de vida:

```hcl
resource "aws_instance" "app" {
  # ...

  lifecycle {
    create_before_destroy = true
    prevent_destroy        = false
    ignore_changes          = [tags["UltimaModificacion"]]
  }
}
```

- ```create_before_destroy```: cuando un cambio requiere destruir y recrear un recurso, crea primero el nuevo y luego destruye el viejo (en vez del orden por defecto, destruir-luego-crear). Imprescindible para recursos que no pueden tener downtime, como un Launch Template en uso.
- ```prevent_destroy```: bloquea cualquier intento de ```destroy``` sobre ese recurso concreto (Terraform da error si lo intentas), como salvaguarda para recursos críticos (una base de datos de producción, por ejemplo).
- ```ignore_changes```: le dice a Terraform que ignore cambios en atributos concretos, útil cuando algo externo (un proceso automático, o AWS mismo) modifica ese atributo y no quieres que Terraform intente revertirlo constantemente.

### 9. Depends_on

Ya visto en la Parte I como concepto; aquí, su forma explícita para forzar una dependencia que Terraform no puede inferir de los atributos:

```hcl
resource "aws_iam_role_policy" "permiso_lambda" {
  # ...
}

resource "aws_lambda_function" "procesador" {
  # ...
  depends_on = [aws_iam_role_policy.permiso_lambda]
}
```

También aplicable a **módulos completos**:

```hcl
module "app" {
  source     = "./modules/app"
  depends_on = [module.red]
}
```

### 10. Import (ampliación)

Aquí, el enfoque **declarativo** moderno (Terraform 1.5+), que además puede generar el código HCL automáticamente:

```hcl
import {
  to = aws_instance.app
  id = "i-0abcd1234efgh5678"
}
```

```bash
terraform plan -generate-config-out=generado.tf
```

Esto genera un fichero ```generado.tf``` con el bloque ```resource "aws_instance" "app" { ... }``` ya relleno con la configuración real de la instancia importada — mucho más rápido que escribirlo a mano y luego ajustar hasta que el ```plan``` no muestre diferencias.

### 11. State Move

Mover un recurso **dentro del estado**, sin tocar la infraestructura real — típico al refactorizar código (por ejemplo, mover un recurso suelto a dentro de un módulo):

```bash
terraform state mv aws_instance.app module.servidor_app.aws_instance.app
```

También existe el bloque declarativo ```moved {}``` (Terraform 1.1+), preferible porque queda documentado en el propio código en lugar de ser un comando puntual que alguien podría olvidar ejecutar:

```hcl
moved {
  from = aws_instance.app
  to   = module.servidor_app.aws_instance.app
}
```

Cuando alguien ejecuta ```terraform plan``` con este bloque presente, Terraform detecta automáticamente el movimiento y actualiza el estado sin planificar destruir/recrear nada.

### 12. Refresh

Sincroniza el estado con la infraestructura **real**, detectando cambios hechos fuera de Terraform (drift):

```bash
terraform apply -refresh-only
```

Este comando muestra las diferencias entre el estado guardado y la realidad, y pregunta si quieres **actualizar el estado** para reflejarlas (sin tocar la infraestructura real — solo corrige lo que Terraform "cree" que existe).

Es la forma correcta de detectar y decidir qué hacer ante un **configuration drift** (por ejemplo, alguien cambió el tamaño de una instancia manualmente desde la consola).

### 13. State Surgery

Conjunto de técnicas **avanzadas y delicadas** para editar el estado directamente cuando los comandos estándar no bastan — por ejemplo, si el estado se corrompió, o si necesitas hacer un cambio muy específico que ningún comando cubre.

```bash
# Descargar el estado a un fichero local
terraform state pull > estado.json

# Editarlo manualmente (con muchísimo cuidado) con un editor de texto
# o con herramientas como jq:
jq '.resources[] | select(.name=="app")' estado.json

# Subir el estado modificado de vuelta
terraform state push estado.json
```

#### Advertencias importantes:

- Siempre haz una copia de seguridad del estado antes de tocarlo (```terraform state pull > backup.json```).
- ```state push``` sobrescribe el estado remoto entero — si alguien más aplicó cambios mientras tanto, los perderías.
- Es una herramienta de último recurso, no de uso rutinario. Casi todo lo que "parece" requerir cirugía de estado se resuelve mejor con ```state mv```, ```moved {}```, o ```import```.


### 14. Remote Backend (garantías)

Un backend remoto bien configurado ofrece dos garantías clave:

- **Consistencia**: todo el equipo trabaja contra el mismo estado, sin copias divergentes.
- **Bloqueo (locking)**: mientras alguien está aplicando cambios, nadie más puede iniciar otro apply sobre el mismo estado a la vez, evitando corrupción por escrituras simultáneas.


### 15. S3 Backend (patrón estándar en AWS)

El patrón más común para proyectos en AWS es usar S3 como backend, con versionado activado (para poder recuperar estados anteriores si algo sale mal):

```hcl
resource "aws_s3_bucket" "estado" {
  bucket = "mi-empresa-terraform-state"
}

resource "aws_s3_bucket_versioning" "estado" {
  bucket = aws_s3_bucket.estado.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "estado" {
  bucket = aws_s3_bucket.estado.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

**Nota curiosa**: este bucket que guarda el estado, en sí mismo, normalmente se crea una **única vez** con un ```terraform apply``` local (con backend local), antes de que exista ningún backend remoto al que apuntar — es el clásico problema del "huevo y la gallina" al arrancar un proyecto nuevo.

### 16. DynamoDB Locking

Antes de Terraform 1.10, el bloqueo de estado en el backend S3 requería una tabla DynamoDB separada:

```hcl
resource "aws_dynamodb_table" "locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

```hcl
terraform {
  backend "s3" {
    bucket         = "mi-empresa-terraform-state"
    key            = "app/produccion/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-locks"  # bloqueo
    encrypt        = true
  }
}
```

Cómo funciona: al ejecutar ```apply```, Terraform intenta crear un ítem en esta tabla con el ID del estado como clave. Si ya existe (otro ```apply``` en curso), Terraform espera o falla con un error de "state locked", en vez de arriesgarse a una escritura concurrente corrupta.

**Nota de versión**: desde Terraform 1.10+, el backend S3 soporta bloqueo **nativo** usando condiciones de S3, sin necesitar ya una tabla DynamoDB separada — pero encontrarás ```dynamodb_table``` en la inmensa mayoría de proyectos existentes y documentación, por lo que sigue siendo importante entenderlo.
