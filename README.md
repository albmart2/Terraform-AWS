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
