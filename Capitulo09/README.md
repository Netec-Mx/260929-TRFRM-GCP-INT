# Refactorizar un proyecto real monolítico en modular y validado.

## Metadatos

| Característica | Detalle |
| :--- | :--- |
| **Duración** | 40 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Crear (Create) |

## Descripción General

En este laboratorio, abordarás un escenario de ingeniería muy común: la refactorización de una infraestructura monolítica y "plana" que ha crecido de forma desorganizada hacia una arquitectura altamente limpia, modular y validada. Tomarás un código inicial que define una Red de VPC, una subred y un secreto en Secret Manager de Google Cloud, y lo reestructurarás en dos módulos locales reutilizables (`modules/networking` y `modules/security`).

Para asegurar que la transición ocurra sin interrupciones ni destrucción accidental de recursos reales en Google Cloud, aprenderás a implementar bloques declarativos `moved` de Terraform. Finalmente, aplicarás validaciones sintácticas y formateo automático de código para cumplir con las mejores prácticas del ciclo de vida operativo.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Extraer una arquitectura monolítica plana (VPC, subredes y secretos) hacia módulos reutilizables locales con interfaces claras de variables y salidas (Inputs/Outputs).
- [ ] Implementar bloques declarativos `moved` para mapear el estado de Terraform existente hacia la nueva estructura modular, garantizando un plan de cero destrucciones (`0 to destroy`).
- [ ] Construir documentación técnica estructurada mediante descripciones explícitas en variables y archivos README.md auto-explicativos.
- [ ] Validar y formatear el código de infraestructura utilizando de forma nativa las utilidades `terraform fmt` y `terraform validate`.

## Prerrequisitos

Para completar este laboratorio de forma exitosa, debes contar con:
1. **Conocimientos Teóricos**:
   - Comprensión del flujo de trabajo de Terraform (`init`, `plan`, `apply`, `destroy`).
   - Familiaridad con el concepto de paso de parámetros entre módulos a través de `variables` y `outputs`.
   - Comprensión básica de los recursos de Google Cloud (`google_compute_network`, `google_compute_subnetwork`, `google_secret_manager_secret`).
2. **Accesos y Credenciales**:
   - Acceso a una cuenta activa de Google Cloud Platform (GCP) con un proyecto configurado y permisos de Administrador/Propietario para crear recursos de red y seguridad.
   - Autenticación activa en tu terminal local mediante la herramienta de línea de comandos de Google Cloud (`gcloud auth application-default login`).
3. **Variables de Entorno**:
   - Tener configurada la variable de entorno `TF_VAR_project_id` apuntando al ID único de tu proyecto GCP.

## Entorno de Laboratorio

El laboratorio se ejecutará en el directorio de trabajo estándar establecido para las prácticas. A continuación, se detallan las versiones específicas de software utilizadas para garantizar la reproducibilidad absoluta:

### Requisitos de Software

| Herramienta | Versión Exacta | Origen / Enlace Oficial |
| :--- | :--- | :--- |
| **Terraform CLI** | v1.8.2 (Linux x86_64) | [HashiCorp Releases v1.8.2](https://releases.hashicorp.com/terraform/1.8.2/) |
| **Google Cloud CLI** | 472.0.0 (o superior) | [Google Cloud SDK Archives](https://cloud.google.com/sdk/docs/downloads-versioned-archives) |
| **GCP Provider for Terraform** | v5.25.0 | [Registry HashiCorp GCP Provider v5.25.0](https://registry.terraform.io/providers/hashicorp/google/5.25.0) |

### Estructura de Directorios del Laboratorio

Trabajarás bajo el directorio raíz `/home/usuario/terraform-gcp-labs/`. El resultado de la estructura de archivos al terminar el laboratorio debe coincidir exactamente con el siguiente esquema:

```text
/home/usuario/terraform-gcp-labs/lab09/
├── main.tf
├── providers.tf
├── variables.tf
├── outputs.tf
├── migrations.tf
├── terraform.tfvars
└── modules/
    ├── networking/
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── README.md
    └── security/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── README.md
```

## Instrucciones Paso a Paso

### Paso 1: Configurar la Infraestructura Monolítica Inicial

**Objetivo**: Simular el estado de partida de una infraestructura "plana" mediante la creación y despliegue inicial de los recursos en GCP dentro de un archivo raíz acoplado.

1. Abre una terminal de comandos en tu estación de trabajo.
2. Posiciónate en el directorio raíz de laboratorios y crea la carpeta para esta práctica:
   ```bash
   mkdir -p /home/usuario/terraform-gcp-labs/lab09
   cd /home/usuario/terraform-gcp-labs/lab09
   ```

3. Crea el archivo `providers.tf` para definir la versión exacta del proveedor de Google Cloud:
   ```hcl
   # /home/usuario/terraform-gcp-labs/lab09/providers.tf
   terraform {
     required_version = ">= 1.8.2"
     required_providers {
       google = {
         source  = "hashicorp/google"
         version = "5.25.0"
       }
     }
   }

   provider "google" {
     project = var.project_id
     region  = var.region
   }
   ```

4. Crea el archivo `variables.tf` para inicializar las variables requeridas de manera plana:
   ```hcl
   # /home/usuario/terraform-gcp-labs/lab09/variables.tf
   variable "project_id" {
     type        = string
     description = "El ID del proyecto de Google Cloud."
   }

   variable "region" {
     type        = string
     description = "Región por defecto para los recursos."
     default     = "us-central1"
   }

   variable "environment" {
     type        = string
     description = "Etiqueta para identificar el entorno de desarrollo/producción."
     default     = "dev"
   }
   ```

5. Crea el archivo de valores `terraform.tfvars`. Asegúrate de reemplazar `TU_PROJECT_ID_AQUÍ` por el ID real de tu proyecto GCP, o asegúrate de que esté configurado como variable de entorno:
   ```hcl
   # /home/usuario/terraform-gcp-labs/lab09/terraform.tfvars
   project_id  = "TU_PROJECT_ID_AQUÍ" # Reemplaza con tu ID real de GCP
   region      = "us-central1"
   environment = "dev"
   ```

6. Crea el archivo monolítico `main.tf` original (plano) que contiene la VPC, una subred y el secreto en Secret Manager:
   ```hcl
   # /home/usuario/terraform-gcp-labs/lab09/main.tf
   resource "google_compute_network" "monolithic_vpc" {
     name                    = "${var.environment}-mono-vpc"
     auto_create_subnetworks = false
   }

   resource "google_compute_subnetwork" "monolithic_subnet" {
     name          = "${var.environment}-mono-subnet"
     ip_cidr_range = "10.10.1.0/24"
     region        = var.region
     network       = google_compute_network.monolithic_vpc.id
   }

   resource "google_secret_manager_secret" "monolithic_secret" {
     secret_id = "${var.environment}-mono-api-key"
     replication {
       auto {}
     }
   }
   ```

7. Ejecuta la inicialización y el aprovisionamiento para registrar la infraestructura en el archivo de estado local de Terraform:
   ```bash
   terraform init
   terraform apply -auto-approve
   ```

**Resultado esperado**:
La terminal confirmará la creación exitosa de tres recursos físicos en tu proyecto de GCP. El comando finalizará imprimiendo un mensaje similar al siguiente:
```text
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

**Verificación**:
Ejecuta el siguiente comando para verificar que los recursos se encuentran registrados en tu archivo local `terraform.tfstate`:
```bash
terraform state list
```
Deberías ver listados exactamente los siguientes elementos:
```text
google_compute_network.monolithic_vpc
google_compute_subnetwork.monolithic_subnet
google_secret_manager_secret.monolithic_secret
```

---

### Paso 2: Diseñar y Crear los Módulos Locales

**Objetivo**: Aislar y encapsular la infraestructura plana en módulos locales independientes (`networking` y `security`), parametrizando sus entradas y salidas para que no dependan directamente del contexto raíz global.

#### Módulo 1: Networking (`modules/networking`)

1. Crea la estructura de directorios necesaria para el módulo de red:
   ```bash
   mkdir -p /home/usuario/terraform-gcp-labs/lab09/modules/networking
   ```

2. Crea el archivo de variables de entrada `/home/usuario/terraform-gcp-labs/lab09/modules/networking/variables.tf`:
   ```hcl
   variable "vpc_name" {
     type        = string
     description = "El nombre de la Red de VPC en GCP."
   }

   variable "subnet_name" {
     type        = string
     description = "El nombre de la subred primaria."
   }

   variable "subnet_cidr" {
     type        = string
     description = "El bloque CIDR para la subred."
     default     = "10.10.1.0/24"
   }

   variable "region" {
     type        = string
     description = "Región de GCP donde se desplegará la subred."
   }
   ```

3. Crea el archivo de lógica principal `/home/usuario/terraform-gcp-labs/lab09/modules/networking/main.tf`:
   ```hcl
   resource "google_compute_network" "vpc" {
     name                    = var.vpc_name
     auto_create_subnetworks = false
   }

   resource "google_compute_subnetwork" "subnet" {
     name          = var.subnet_name
     ip_cidr_range = var.subnet_cidr
     region        = var.region
     network       = google_compute_network.vpc.id
   }
   ```

4. Crea el archivo de salidas `/home/usuario/terraform-gcp-labs/lab09/modules/networking/outputs.tf`:
   ```hcl
   output "vpc_id" {
     value       = google_compute_network.vpc.id
     description = "El ID o URI de la VPC de GCP creada."
   }

   output "subnet_id" {
     value       = google_compute_subnetwork.subnet.id
     description = "El ID o URI de la subred de GCP creada."
   }
   ```

5. Agrega documentación básica en formato Markdown `/home/usuario/terraform-gcp-labs/lab09/modules/networking/README.md`:
   ```markdown
   # Módulo de Redes (Networking Module)

   Este módulo local permite aprovisionar de forma segura una Red de VPC personalizada y una subred asociada en Google Cloud Platform.

   ## Entradas (Inputs)
   - `vpc_name`: Nombre de la Red de VPC.
   - `subnet_name`: Nombre de la subred principal.
   - `subnet_cidr`: Segmento CIDR asignado a la subred.
   - `region`: Región de despliegue en GCP.

   ## Salidas (Outputs)
   - `vpc_id`: Identificador de la VPC.
   - `subnet_id`: Identificador de la subred.
   ```

---

#### Módulo 2: Security (`modules/security`)

1. Crea la estructura de directorios necesaria para el módulo de seguridad:
   ```bash
   mkdir -p /home/usuario/terraform-gcp-labs/lab09/modules/security
   ```

2. Crea el archivo de variables de entrada `/home/usuario/terraform-gcp-labs/lab09/modules/security/variables.tf`:
   ```hcl
   variable "secret_id" {
     type        = string
     description = "El identificador del secreto en Secret Manager de GCP."
   }
   ```

3. Crea el archivo de lógica principal `/home/usuario/terraform-gcp-labs/lab09/modules/security/main.tf`:
   ```hcl
   resource "google_secret_manager_secret" "secret" {
     secret_id = var.secret_id
     replication {
       auto {}
     }
   }
   ```

4. Crea el archivo de salidas `/home/usuario/terraform-gcp-labs/lab09/modules/security/outputs.tf`:
   ```hcl
   output "secret_id" {
     value       = google_secret_manager_secret.secret.id
     description = "La ruta completa (URI) del recurso secreto en GCP."
   }
   ```

5. Crea el archivo de documentación básica `/home/usuario/terraform-gcp-labs/lab09/modules/security/README.md`:
   ```markdown
   # Módulo de Seguridad (Security Module)

   Este módulo encapsula la creación de secretos corporativos en Google Cloud Secret Manager con replicación automática activada.

   ## Entradas (Inputs)
   - `secret_id`: ID único para registrar el secreto.

   ## Salidas (Outputs)
   - `secret_id`: ID completo del recurso creado.
   ```

**Resultado esperado**:
La estructura de los módulos locales ya está completamente configurada y con la sintaxis parametrizada, lista para ser consumida de forma independiente por el directorio raíz.

---

### Paso 3: Refactorizar el Directorio Raíz e Integrar Bloques Moved

**Objetivo**: Modificar los archivos del directorio raíz para que dejen de usar recursos individuales y, en su lugar, consuman los módulos locales. Adicionalmente, configurar un archivo de migración declarativa mediante bloques `moved`.

1. Sobrescribe por completo el archivo `main.tf` de la raíz `/home/usuario/terraform-gcp-labs/lab09/main.tf` con las llamadas a los módulos:
   ```hcl
   # /home/usuario/terraform-gcp-labs/lab09/main.tf

   module "networking" {
     source      = "./modules/networking"
     vpc_name    = "${var.environment}-mono-vpc"
     subnet_name = "${var.environment}-mono-subnet"
     subnet_cidr = "10.10.1.0/24"
     region      = var.region
   }

   module "security" {
     source    = "./modules/security"
     secret_id = "${var.environment}-mono-api-key"
   }
   ```

2. Crea un nuevo archivo llamado `migrations.tf` en el directorio raíz `/home/usuario/terraform-gcp-labs/lab09/migrations.tf`. Este archivo le indicará a Terraform que los antiguos recursos planos ahora pertenecen a los nuevos módulos, evitando su destrucción física en GCP:
   ```hcl
   # /home/usuario/terraform-gcp-labs/lab09/migrations.tf

   moved {
     from = google_compute_network.monolithic_vpc
     to   = module.networking.google_compute_network.vpc
   }

   moved {
     from = google_compute_subnetwork.monolithic_subnet
     to   = module.networking.google_compute_subnetwork.subnet
   }

   moved {
     from = google_secret_manager_secret.monolithic_secret
     to   = module.security.google_secret_manager_secret.secret
   }
   ```

3. Crea el archivo `outputs.tf` en la raíz para exponer el estado final estructurado y legible:
   ```hcl
   # /home/usuario/terraform-gcp-labs/lab09/outputs.tf

   output "vpc_id" {
     value       = module.networking.vpc_id
     description = "ID de la VPC migrada mediante el módulo de red."
   }

   output "subnet_id" {
     value       = module.networking.subnet_id
     description = "ID de la subred migrada mediante el módulo de red."
   }

   output "secret_id" {
     value       = module.security.secret_id
     description = "ID del secreto migrado mediante el módulo de seguridad."
   }
   ```

---

### Paso 4: Validar, Formatear y Verificar la Migración sin Destrucción

**Objetivo**: Garantizar el cumplimiento de buenas prácticas de desarrollo en Terraform (formato y semántica) y comprobar que el plan de migración no genere impacto destructivo en GCP.

1. **Formateo**: Ejecuta el formateador recursivo de Terraform para unificar el estilo de sangrías, tabulaciones y espaciados tanto en el directorio raíz como en los módulos:
   ```bash
   terraform fmt -recursive
   ```

2. **Inicialización**: Inicializa de nuevo el directorio de trabajo para registrar y descargar los metadatos de los módulos locales recientemente configurados:
   ```bash
   terraform init
   ```

3. **Validación Sintáctica**: Ejecuta la validación nativa del compilador de Terraform para asegurar que no existan discrepancias lógicas, de sintaxis o variables huérfanas:
   ```bash
   terraform validate
   ```
   *Deberás observar en consola el mensaje:* `Success! The configuration is valid.`

4. **Planificación y Migración**: Ejecuta un plan de ejecución de Terraform y examina con extrema atención la salida:
   ```bash
   terraform plan
   ```

**Resultado esperado en la consola**:
En lugar de mostrar que se añadirán 3 recursos y se destruirán 3 recursos, la terminal imprimirá un bloque con el texto `has moved to`. La sección final del plan indicará exactamente que no se realizará ningún cambio físico destructivo en Google Cloud:

```text
Terraform will perform the following actions:

  # google_compute_network.monolithic_vpc has moved to module.networking.google_compute_network.vpc
    ~ resource "google_compute_network" "vpc" {
        id   = "projects/TU_PROJECT_ID_AQUÍ/global/networks/dev-mono-vpc"
        name = "dev-mono-vpc"
        # (y el resto de atributos no cambian)
      }

  # google_compute_subnetwork.monolithic_subnet has moved to module.networking.google_compute_subnetwork.subnet
    ~ resource "google_compute_subnetwork" "subnet" {
        id   = "projects/TU_PROJECT_ID_AQUÍ/regions/us-central1/subnetworks/dev-mono-subnet"
        name = "dev-mono-subnet"
        # (y el resto de atributos no cambian)
      }

  # google_secret_manager_secret.monolithic_secret has moved to module.security.google_secret_manager_secret.secret
    ~ resource "google_secret_manager_secret" "secret" {
        id   = "projects/TU_PROJECT_ID_AQUÍ/secrets/dev-mono-api-key"
        # (y el resto de atributos no cambian)
      }

Plan: 0 to add, 0 to change, 0 to destroy.
```

**Verificación de Seguridad**:
Confirma visualmente que el plan diga exactamente **`0 to add, 0 to change, 0 to destroy`**. Si indica destrucciones, aborta el proceso de inmediato.

5. **Aplicar Migración de Estado**: Si el plan es de cero cambios físicos, aplica la actualización del estado local de Terraform de manera segura:
   ```bash
   terraform apply -auto-approve
   ```

El estado se actualizará de forma instantánea puesto que no hay APIs de Google Cloud que invocar para modificar infraestructura física. Tu base de datos de Terraform estará ahora correctamente modularizada.

---

## Validación y Pruebas

Para garantizar que el laboratorio se ha completado siguiendo los más altos estándares de ingeniería y robustez operativa, realiza las siguientes actividades de evaluación:

### 1. Comprobación Estructural del Estado
Ejecuta el listado de recursos para certificar que el archivo de estado de Terraform (`terraform.tfstate`) refleja con éxito las rutas modularizadas:
```bash
terraform state list
```
**Resultado esperado en consola**:
```text
module.networking.google_compute_network.vpc
module.networking.google_compute_subnetwork.subnet
module.security.google_secret_manager_secret.secret
```

### 2. Prueba Adversaria (Caso Negativo / Inyección de Configuración Errónea)
Para validar la solidez del diseño y comprobar el comportamiento de protección ante fallos, introduce temporalmente una variable conflictiva o elimina un bloque `moved` para observar el impacto:
1. Renombra de forma temporal el archivo de migraciones para simular un escenario donde un desarrollador olvidó declararlo:
   ```bash
   mv migrations.tf migrations.tf.bak
   ```
2. Ejecuta una simulación de plan para validar la respuesta del compilador:
   ```bash
   terraform plan
   ```
3. **Comportamiento Crítico Observado**:
   Notarás que la herramienta, al perder el mapeo histórico, interpreta la configuración de la siguiente manera:
   ```text
   Plan: 3 to add, 0 to change, 3 to destroy.
   ```
   *¡ALERTA!: Esto causaría la destrucción total de la VPC, de todas sus subredes y la pérdida permanente de los secretos del Secret Manager corporativo.*
4. **Restaurar**: Devuelve el archivo a su lugar seguro para validar que la infraestructura se mantenga intacta:
   ```bash
   mv migrations.tf.bak migrations.tf
   terraform plan
   ```
   Confirmarás que vuelve a mostrar el esperado mensaje de seguridad: `Plan: 0 to add, 0 to change, 0 to destroy`.

---

## Solución de Problemas

A continuación se describen dos escenarios típicos de fallo que puedes experimentar durante la ejecución de este laboratorio, junto con su causa raíz y plan de mitigación:

### Caso 1: Error "Missing required argument" al inicializar o planificar el módulo
* **Síntoma**: Al ejecutar `terraform plan`, se produce un error en el compilador similar al siguiente:
  ```text
  Error: Missing required argument
  on main.tf line X:
  The argument "vpc_name" is required, but no definition was found.
  ```
* **Causa**: Al refactorizar y llamar al módulo dentro del `main.tf` del directorio raíz, no estás declarando u omitiste una variable que fue configurada como obligatoria (es decir, sin un atributo `default` asignado) en el archivo `variables.tf` interno de la subcarpeta del módulo.
* **Solución**: Compara minuciosamente las variables requeridas en `modules/networking/variables.tf` con el bloque de invocación en `main.tf` de la raíz. Asegúrate de declarar de manera explícita cada variable requerida por el módulo, pasándole su respectivo valor o referencia de variable raíz.

### Caso 2: El plan muestra destrucciones ("to destroy") tras aplicar los bloques moved
* **Síntoma**: A pesar de que creaste el archivo `migrations.tf` y configuraste bloques `moved`, la salida de la terminal sigue listando que algunos recursos serán destruidos y reemplazados.
* **Causa**: Este comportamiento errático ocurre si hay diferencias de nomenclatura (typos) en las rutas declaradas en los argumentos `from` o `to` dentro de los bloques `moved`. Por ejemplo, si especificaste `module.networking.google_compute_network.monolithic_vpc` en lugar de la referencia interna real `module.networking.google_compute_network.vpc`.
* **Solución**:
  1. Ejecuta `terraform state list` para copiar la dirección exacta actual en el estado (este será tu parámetro `from`).
  2. Abre `modules/networking/main.tf` (o el módulo de seguridad según corresponda) y verifica la definición exacta del recurso y su tipo (este será tu parámetro `to`, prefijado con el nombre de llamada del módulo). Corrije los caracteres erróneos en el archivo `migrations.tf` y vuelve a correr `terraform plan`.

---

## Limpieza

Para evitar el consumo de recursos económicos innecesarios en tu cuenta de GCP, ejecuta las tareas de desmantelamiento de la infraestructura una vez que hayas verificado la correcta finalización del laboratorio:

1. Asegúrate de encontrarte en el directorio raíz de trabajo para esta práctica:
   ```bash
   cd /home/usuario/terraform-gcp-labs/lab09
   ```

2. Ejecuta el comando de destrucción de recursos de Terraform, utilizando el respectivo archivo de variables que contiene el ID de tu proyecto GCP:
   ```bash
   terraform destroy -auto-approve
   ```

3. **Verificación de Limpieza**:
   Confirma en la salida de terminal que los tres recursos físicos creados originalmente hayan sido destruidos por completo:
   ```text
   Destroy complete! Resources: 3 destroyed.
   ```

4. *(Opcional)* Si ya no realizarás más prácticas en esta sesión, puedes remover de manera local las carpetas y archivos temporales generados por la inicialización de Terraform:
   ```bash
   rm -rf .terraform/ .terraform.lock.hcl terraform.tfstate terraform.tfstate.backup
   ```

---

## Resumen

En este laboratorio has completado con éxito una tarea crítica de arquitectura de sistemas con Terraform:

- **Estructuración Modular**: Refactorizaste una configuración monolítica de un único archivo plano hacia una arquitectura organizada basada en módulos locales reutilizables con un claro flujo de datos (inputs/outputs).
- **Mantenibilidad y Robustez**: Documentaste apropiadamente las variables y outputs agregando un contexto técnico explícito a cada recurso creado, y aplicaste las herramientas nativas de consistencia `terraform fmt` y `terraform validate`.
- **Migración de Estado Declarativa**: Implementaste bloques `moved {}`, dominando el enfoque moderno recomendado de HashiCorp para manipular la base de datos de estado sin interferir directamente con los recursos en producción ni incurrir en destrucción o indisponibilidad de servicios.

### Recursos Adicionales y Lecturas Recomendadas
* [Documentación Oficial de Terraform: Refactoring de Configuraciones](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring)
* [Uso Avanzado de Bloques Moved en Proyectos Multientorno](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring#moved-blocks)
* [Buenas Prácticas para el Diseño de Módulos (HashiCorp Style Guide)](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
