# Crear y usar módulos de red y máquinas virtuales.

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 55 minutos |
| **Complejidad** | Media |
| **Nivel de Taxonomía de Bloom** | Crear / Aplicar |
| **Objetivos de Aprendizaje** | 1. Refactorizar código monolítico en submódulos encapsulados.<br>2. Diseñar interfaces parametrizadas (inputs/outputs).<br>3. Consumir outputs de módulos de red como inputs en módulos de cómputo. |

---

## Descripción General

Este laboratorio práctico guía al estudiante en el proceso de refactorización de una infraestructura monolítica de Google Cloud Platform (GCP) en Terraform hacia un diseño modular limpio, escalable y alineado con las mejores prácticas operativas de la industria. 

El estudiante organizará el código del proyecto estructurando el directorio de trabajo en módulos locales reutilizables. Diseñará un módulo de red (`modules/vpc`) que exponga el ID y el nombre de la subred creada, y un módulo de cómputo (`modules/vm`) que reciba estos parámetros para desplegar una máquina virtual en la subred correspondiente. Finalmente, integrará ambos módulos en el directorio raíz mediante un flujo unidireccional de datos controlado por variables de entrada (`inputs`) y valores de salida (`outputs`).

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Construir y organizar un sistema de archivos para desarrollo modular en Terraform localizando recursos bajo el principio DRY (*Don't Repeat Yourself*).
- [ ] Definir e implementar variables de entrada (`variables.tf`) estructuradas con tipos de datos explícitos para parametrizar redes y recursos de cómputo.
- [ ] Desarrollar bloques de salida (`outputs.tf`) que expongan atributos críticos de recursos aislados dentro de un módulo hijo.
- [ ] Enlazar múltiples módulos locales dentro de un módulo raíz utilizando la asignación de salidas como parámetros de entrada.
- [ ] Validar y aplicar el flujo completo utilizando el CLI de Terraform (`init`, `validate`, `plan`, `apply`) asegurando el aprovisionamiento correcto en GCP.

---

## Prerrequisitos

Para completar con éxito este laboratorio, debes cumplir con los siguientes requisitos previos:
1. **Conocimientos teóricos y prácticos:**
   - Comprensión del ciclo de vida básico de Terraform (`init`, `plan`, `apply`, `destroy`).
   - Familiaridad con los recursos de GCP: Redes de Computación Virtual (VPC), Subredes, Reglas de Firewall y Máquinas Virtuales (Compute Engine).
   - Concepto de flujo de transferencia de datos en HCL (interfaz de variables y outputs).

2. **Acceso y Configuración:**
   - Acceso a una cuenta de Google Cloud Platform (GCP) con permisos de Administrador/Propietario (*Owner*) o Editor en un proyecto específico.
   - Una sesión de terminal Linux local con acceso a internet sin restricciones corporativas para puertos `80`, `443` y `22`.

---

## Entorno de Laboratorio

Este laboratorio está diseñado para ejecutarse en la estación de trabajo local configurada bajo los siguientes parámetros de sistema y herramientas validadas.

### Requisitos de Hardware del Sistema

| Componente | Requisito Mínimo | Requisito Recomendado |
| :--- | :--- | :--- |
| **CPU** | Arquitectura x86_64 o ARM64 (2 núcleos) | Arquitectura x86_64 (4 núcleos o superior) |
| **Memoria RAM** | 8 GB | 16 GB |
| **Espacio en Disco** | 10 GB Libres | 20 GB Libres (SSD) |
| **Conexión de Red** | 10 Mbps simétricos sin bloqueos de firewall | 50 Mbps o superior |

### Herramientas de Software Requeridas

| Software / Herramienta | Versión Exacta | Arquitectura | Enlace Oficial de Descarga | Licencia |
| :--- | :--- | :--- | :--- | :--- |
| **HashiCorp Terraform** | `1.8.2` | Linux x86_64 | [Descargar de HashiCorp](https://releases.hashicorp.com/terraform/1.8.2/) | Mozilla Public License v2.0 |
| **Google Cloud Provider** | `5.25.0` | Plugin (gcp) | [Registro de Terraform](https://registry.terraform.io/providers/hashicorp/google/5.25.0) | Mozilla Public License v2.0 |
| **Google Cloud SDK (gcloud CLI)**| `472.0.0` | Linux x86_64 | [Descargar de Google Cloud](https://cloud.google.com/sdk/docs/release-notes) | Apache License 2.0 |

> **Nota sobre licencias de IA:** Si utilizas herramientas de asistencia de codificación durante la práctica (por ejemplo, *GitHub Copilot* o *Copilot Chat*), ten en cuenta que estas utilidades requieren una suscripción comercial o educativa activa vinculada a tu cuenta de usuario e IDE.

### Comandos de Inicialización del Entorno

Antes de comenzar el laboratorio, ejecuta las siguientes instrucciones en tu terminal local para asegurar la ruta de trabajo y las variables de entorno predeterminadas requeridas:

```bash
## 1. Asegurar la existencia del directorio raíz del laboratorio
mkdir -p /home/usuario/terraform-gcp-labs/

## 2. Navegar a la ruta de trabajo establecida
cd /home/usuario/terraform-gcp-labs/

## 3. Exportar el ID de tu proyecto de GCP como variable de entorno de Terraform.
## REEMPLAZA "tu-proyecto-id-gcp" con el ID real provisto por tu administrador de la consola.
export TF_VAR_project_id="tu-proyecto-id-gcp"

## 4. Validar las versiones de las herramientas críticas instaladas
terraform -v
gcloud --version
```

---

## Instrucciones Paso a Paso

Sigue detenidamente los pasos secuenciales para estructurar, codificar y desplegar tu infraestructura modular.

### Paso 1: Crear la Estructura de Directorios para los Módulos

**Objetivo:** Organizar el sistema de archivos del proyecto local para separar las configuraciones del módulo raíz y los módulos hijo de red y cómputo.

**Instrucciones:**

1. Asegúrate de estar posicionado en el directorio raíz de la práctica:
   ```bash
   cd /home/usuario/terraform-gcp-labs/
   ```

2. Ejecuta el comando de creación de directorios para definir la jerarquía modular:
   ```bash
   mkdir -p modules/vpc modules/vm
   ```

3. Genera los archivos en blanco necesarios para cada módulo. Ejecuta los siguientes comandos:
   ```bash
   # Archivos para el módulo VPC
   touch modules/vpc/main.tf modules/vpc/variables.tf modules/vpc/outputs.tf

   # Archivos para el módulo VM
   touch modules/vm/main.tf modules/vm/variables.tf modules/vm/outputs.tf

   # Archivos para el módulo raíz
   touch main.tf variables.tf providers.tf terraform.tfvars
   ```

4. Valida que la estructura del sistema de archivos sea idéntica a la siguiente estructura lógica:
   ```text
   /home/usuario/terraform-gcp-labs/
   ├── main.tf
   ├── providers.tf
   ├── terraform.tfvars
   ├── variables.tf
   └── modules/
       ├── vm/
       │   ├── main.tf
       │   ├── outputs.tf
       │   └── variables.tf
       └── vpc/
           ├── main.tf
           ├── outputs.tf
           └── variables.tf
   ```

**Output esperado:**
Al ejecutar `tree .` o inspeccionar el panel lateral del editor de código, se debe apreciar la separación completa de las carpetas `vpc` y `vm` dentro del directorio contenedor `modules`.

**Verificación:**
```bash
find . -maxdepth 3 -not -path '*/.*'
```
Debe listar exactamente los archivos `.tf` creados sin subcarpetas adicionales ni archivos huérfanos.

---

### Paso 2: Desarrollar el Módulo de Red (`modules/vpc`)

**Objetivo:** Implementar un módulo autónomo que cree una red VPC personalizada, una subred con el rango CIDR parametrizado y una regla de firewall básica para el acceso por puerto 22 (SSH) y 80 (HTTP).

**Instrucciones:**

1. Abre el archivo `modules/vpc/variables.tf` con tu editor de texto y define los parámetros requeridos para la red:
   ```hcl
   variable "vpc_name" {
     type        = string
     description = "Nombre que se le asignará a la red VPC personalizada."
   }

   variable "subnet_name" {
     type        = string
     description = "Nombre que se le asignará a la subred de GCP."
   }

   variable "ip_cidr_range" {
     type        = string
     description = "Rango de IPs de la subred en formato CIDR (ej. 10.0.1.0/24)."
   }

   variable "region" {
     type        = string
     description = "Región de Google Cloud donde se desplegará la subred."
     default     = "us-central1"
   }
   ```

2. Abre el archivo `modules/vpc/main.tf` y define la red VPC, la subred asociada y las reglas de firewall para el tráfico seguro:
   ```hcl
   resource "google_compute_network" "custom_vpc" {
     name                    = var.vpc_name
     auto_create_subnetworks = false
   }

   resource "google_compute_subnetwork" "custom_subnet" {
     name          = var.subnet_name
     ip_cidr_range = var.ip_cidr_range
     region        = var.region
     network       = google_compute_network.custom_vpc.id
     
     private_ip_google_access = true
   }

   resource "google_compute_firewall" "allow_ssh_http" {
     name    = "${var.vpc_name}-allow-ssh-http"
     network = google_compute_network.custom_vpc.name

     allow {
       protocol = "tcp"
       ports    = ["22", "80"]
     }

     # Rango CIDR global para permitir accesos demostrativos de administración
     source_ranges = ["0.0.0.0/0"]
   }
   ```

3. Configura el archivo `modules/vpc/outputs.tf` para exponer los datos que requerirá el módulo de cómputo:
   ```hcl
   output "vpc_id" {
     value       = google_compute_network.custom_vpc.id
     description = "El ID único de la red VPC generada."
   }

   output "vpc_name" {
     value       = google_compute_network.custom_vpc.name
     description = "El nombre de la red VPC generada."
   }

   output "subnet_id" {
     value       = google_compute_subnetwork.custom_subnet.id
     description = "La URI o ID de la subred generada."
   }

   output "subnet_name" {
     value       = google_compute_subnetwork.custom_subnet.name
     description = "El nombre de la subred generada."
   }
   ```

**Output esperado:**
Los archivos del directorio `modules/vpc/` están guardados con las variables definidas, los recursos tipificados de forma genérica y las salidas del identificador de subred listas para ser consumidas.

**Verificación:**
Ejecuta una validación sintáctica básica en el directorio del módulo:
```bash
terraform fmt -check modules/vpc/
```

---

### Paso 3: Desarrollar el Módulo de Cómputo (`modules/vm`)

**Objetivo:** Crear un módulo parametrizado para instanciar máquinas virtuales Compute Engine (`e2-micro`), mapeándolas dinámicamente a la red y subred creadas por el módulo anterior.

**Instrucciones:**

1. Abre el archivo `modules/vm/variables.tf` y declara los parámetros necesarios para la instancia de procesamiento:
   ```hcl
   variable "instance_name" {
     type        = string
     description = "Nombre único de la instancia de máquina virtual."
   }

   variable "machine_type" {
     type        = string
     description = "Tipo de máquina virtual (ej. e2-micro para desarrollo, e2-small para pruebas)."
     default     = "e2-micro"
   }

   variable "zone" {
     type        = string
     description = "Zona específica de GCP para el aprovisionamiento de la instancia."
     default     = "us-central1-a"
   }

   variable "subnet_name" {
     type        = string
     description = "El nombre de la subred donde se conectará la tarjeta de red (NIC) de la VM."
   }

   variable "network_name" {
     type        = string
     description = "El nombre de la VPC de red asociada."
   }
   ```

2. Configura la lógica de creación del recurso en `modules/vm/main.tf` especificando la imagen del sistema operativo de arranque y la asociación dinámica de red:
   ```hcl
   resource "google_compute_instance" "vm_instance" {
     name         = var.instance_name
     machine_type = var.machine_type
     zone         = var.zone

     # Etiqueta requerida para identificar recursos de prueba
     labels = {
       environment = "development"
       managed_by  = "terraform"
     }

     boot_disk {
       initialize_params {
         image = "debian-cloud/debian-12"
         size  = 10
         type  = "pd-standard"
       }
     }

     network_interface {
       network    = var.network_name
       subnetwork = var.subnet_name

       # La presencia de access_config asigna una IP externa efímera
       access_config {
         # Dejar vacío para IP pública dinámica
       }
     }

     metadata = {
       startup-script = "echo 'Lab 2 Completado: Servidor Web Inicializado' > /var/www/html/index.html"
     }
   }
   ```

3. Agrega la siguiente salida en `modules/vm/outputs.tf` para obtener la dirección IP pública de la instancia de desarrollo tras el despliegue:
   ```hcl
   output "instance_public_ip" {
     value       = google_compute_instance.vm_instance.network_interface[0].access_config[0].assigned_network_ip
     description = "La dirección IP externa asignada a la máquina virtual."
   }

   output "instance_self_link" {
     value       = google_compute_instance.vm_instance.self_link
     description = "URI del enlace interno de la instancia de cómputo."
   }
   ```

**Output esperado:**
El submódulo de cómputo está configurado para no tener ninguna dependencia estática con IDs de red cableados directamente en el código de recursos (*hardcoded*).

**Verificación:**
Verifica la estructura sintáctica del módulo:
```bash
terraform fmt -check modules/vm/
```

---

### Paso 4: Configurar el Módulo Raíz

**Objetivo:** Integrar y conectar los dos submódulos creados en el directorio principal, pasando las salidas del módulo de red como variables de entrada del módulo de cómputo.

**Instrucciones:**

1. Configura el archivo de definición de proveedores en el directorio raíz (`/home/usuario/terraform-gcp-labs/providers.tf`):
   ```hcl
   terraform {
     required_version = "1.8.2"
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
     zone    = var.zone
   }
   ```

2. Abre el archivo de variables del módulo raíz (`/home/usuario/terraform-gcp-labs/variables.tf`) para exponer las opciones personalizables globales:
   ```hcl
   variable "project_id" {
     type        = string
     description = "El ID único del proyecto de GCP asignado para el laboratorio."
   }

   variable "region" {
     type        = string
     default     = "us-central1"
     description = "Región estándar por defecto para redes de GCP."
   }

   variable "zone" {
     type        = string
     default     = "us-central1-a"
     description = "Zona estándar por defecto para recursos de cómputo."
   }
   ```

3. Crea el archivo de asignación de variables locales (`/home/usuario/terraform-gcp-labs/terraform.tfvars`) para establecer los rangos y nombres que utilizará tu arquitectura modular:
   ```hcl
   # El valor de project_id se leerá automáticamente de la variable de entorno TF_VAR_project_id.
   # No definas project_id aquí para evitar la filtración de credenciales.
   region = "us-central1"
   zone   = "us-central1-a"
   ```

4. Define las llamadas a los módulos en el archivo principal (`/home/usuario/terraform-gcp-labs/main.tf`):
   ```hcl
   # 1. Llamada al Módulo de Red (Hijo)
   module "network" {
     source = "./modules/vpc"

     vpc_name      = "lab2-modular-vpc"
     subnet_name   = "lab2-modular-subnet"
     ip_cidr_range = "10.10.10.0/24"
     region        = var.region
   }

   # 2. Llamada al Módulo de Cómputo (Hijo) - Notar el encadenamiento de variables
   module "compute" {
     source = "./modules/vm"

     instance_name = "lab2-modular-vm"
     machine_type  = "e2-micro" # VM económica para desarrollo local
     zone          = var.zone

     # Flujo de datos: Los inputs de este módulo se alimentan de los outputs del módulo network
     network_name  = module.network.vpc_name
     subnet_name   = module.network.subnet_name
   }

   # 3. Outputs de nivel de módulo raíz
   output "vpc_created_id" {
     value       = module.network.vpc_id
     description = "Identificador de la red creada mediante el módulo hijo."
   }

   output "vm_assigned_ip" {
     value       = module.compute.instance_public_ip
     description = "IP externa para acceso por consola a la máquina virtual."
   }
   ```

**Output esperado:**
El módulo raíz se convierte en un orquestador declarativo muy limpio que consume los submódulos locales y conecta sus entradas y salidas sin almacenar rutas estáticas de GCP.

**Verificación:**
Prueba el formato y la estructura del proyecto completo:
```bash
terraform fmt -recursive
```

---

### Paso 5: Inicialización y Aplicación de los Módulos

**Objetivo:** Ejecutar el ciclo de despliegue de Terraform para registrar los submódulos en el directorio local `.terraform` y aprovisionar la arquitectura integrada en Google Cloud.

**Instrucciones:**

1. Asegúrate de estar en el directorio de trabajo donde se encuentran los archivos `providers.tf` y `main.tf` del módulo raíz:
   ```bash
   cd /home/usuario/terraform-gcp-labs/
   ```

2. Ejecuta el proceso de inicialización del proyecto. El comando `init` detectará las carpetas en `./modules/` y creará enlaces lógicos en la caché local:
   ```bash
   terraform init
   ```

3. Valida la corrección semántica y lógica de tu código refactorizado con el comando de validación:
   ```bash
   terraform validate
   ```

4. Genera un plan de ejecución de infraestructura guardándolo de forma segura en un archivo temporal de plan para asegurar la consistencia del despliegue:
   ```bash
   terraform plan -out=tfplan.binary
   ```
   *Examina detenidamente la salida del plan. El CLI de Terraform informará de la creación de 4 recursos (Red, Subred, Regla de Firewall, Instancia VM).*

5. Aplica los cambios aprobados en el plan:
   ```bash
   terraform apply tfplan.binary
   ```

**Output esperado:**
Un mensaje final exitoso de Terraform indicando que los cambios se aplicaron correctamente:
```text
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

vm_assigned_ip = "34.121.x.x"
vpc_created_id = "projects/tu-proyecto-id-gcp/global/networks/lab2-modular-vpc"
```

**Verificación:**
```bash
## Validar desde el CLI de Google Cloud (gcloud) que los recursos existan con los parámetros de nuestro módulo
gcloud compute instances list --filter="name=lab2-modular-vm"
gcloud compute networks list --filter="name=lab2-modular-vpc"
```

---

## Validación y Pruebas

Para garantizar que los módulos locales aíslan correctamente la lógica y que la infraestructura se ha desplegado conforme a las especificaciones estipuladas, realiza las siguientes pruebas y validaciones.

### Prueba 1: Verificación de la VPC y Subred Modularizada

Ejecuta el siguiente comando de inspección a través de `gcloud` para verificar que la red herede los parámetros exactos proporcionados a las variables de entrada de los módulos locales:

```bash
gcloud compute networks subnets describe lab2-modular-subnet \
    --region=us-central1 \
    --format="json(ipCidrRange, privateIpGoogleAccess, network)"
```

**Resultado Esperado:**
El formato de retorno en JSON debe reflejar la dirección IP en el rango específico configurado (`10.10.10.0/24`) y mostrar `privateIpGoogleAccess` configurado en `true` según la definición de nuestro recurso modular.

---

### Prueba 2: Verificación de Conectividad y Consumo de Outputs

Utilizando la salida `vm_assigned_ip` provista por el comando `terraform output`, realiza una comprobación de ping o conexión SSH simulada para verificar que las reglas de firewall definidas dentro del módulo de red estén operativas:

```bash
## Obtener la IP pública directamente desde el estado modularizado de Terraform
VM_IP=$(terraform output -raw vm_assigned_ip)

## Comprobar estado de puertos usando un escáner rápido de red local o comando curl
curl -I -m 5 http://$VM_IP/
```

*Nota: La VM podría tomar hasta 2 minutos en inicializar su configuración interna y el script de arranque básico de servidor web.*

---

### Prueba de Limitaciones de IA (Caso Adversarial)

Para asegurar la robustez de tus módulos contra errores de configuración, fallos imprevistos de LLMs (en caso de utilizar prompts automáticos para generar código adicional) e inconsistencias, realiza la siguiente prueba adversarial:

Intenta inyectar de forma forzada un valor de CIDR no válido (por ejemplo, una cadena de texto vacía, o un rango CIDR mal estructurado como `"999.999.999.999/99"`) directamente en el archivo de variables del módulo raíz `terraform.tfvars`, o intenta desplegar la máquina virtual especificando una zona que no coincida con la región de la red (por ejemplo, `region = "us-central1"` pero `zone = "europe-west1-b"`).

1. Abre tu archivo `/home/usuario/terraform-gcp-labs/terraform.tfvars` y temporalmente introduce el conflicto de zona:
   ```hcl
   region = "us-central1"
   zone   = "europe-west1-b" # Conflicto geográfico forzado
   ```

2. Ejecuta un comando de validación y un plan:
   ```bash
   terraform plan
   ```

3. **Verificación de Resistencia:** Examina cómo Terraform detecta la inconsistencia de aprovisionamiento regional/zonal. El comando de plan o apply fallará con un error provisto por el proveedor de Google Cloud indicando que la zona `europe-west1-b` no es válida dentro de la subred regional asignada en `us-central1`. 
   
   *Este ejercicio demuestra que, aunque encapsulemos recursos en módulos, las validaciones lógicas del proveedor y la API de GCP continúan actuando como una red de seguridad infalible contra malas configuraciones automatizadas.*

4. **Corrección:** Regresa tu archivo `/home/usuario/terraform-gcp-labs/terraform.tfvars` a su estado correcto:
   ```hcl
   region = "us-central1"
   zone   = "us-central1-a"
   ```

---

## Solución de Problemas

A continuación, se describen los dos problemas más comunes que pueden surgir durante el desarrollo y consumo de módulos locales en Terraform, junto con sus respectivos síntomas, causas raíz y soluciones.

### Problema 1: Fallo de Inicialización de Módulos Modificados (`Error: Module not installed`)

- **Síntomas:** Al ejecutar `terraform plan` o `terraform apply` después de haber añadido un nuevo bloque de módulo o haber modificado significativamente las rutas relativas en el código de llamadas del módulo raíz, el CLI de Terraform muestra el siguiente error:
  ```text
  Error: Module not installed

  on main.tf line 12:
  12: module "compute" {

  The source address "./modules/vm" was not found in the local cache. Run "terraform init" to install it.
  ```
- **Causa:** Terraform mantiene una base de datos local y un mapa de punteros lógicos a los submódulos locales dentro de la carpeta oculta `.terraform/modules/`. Si agregas un nuevo bloque `module` en tu configuración de HCL, Terraform se niega a operar sobre él hasta que se realice el enlace formal de la ruta mediante el comando de inicialización.
- **Solución:** Ejecuta nuevamente el comando de inicialización del backend para reconstruir el mapeo interno de los submódulos:
  ```bash
  terraform init
  ```
  Esto actualizará las referencias del directorio de caché local sin alterar el estado de los recursos que ya estén aprovisionados en Google Cloud.

---

### Problema 2: Error de Ciclo de Dependencia Circular (`Cycle error: module.network depends on module.compute...`)

- **Síntomas:** La ejecución de `terraform plan` falla abruptamente, mostrando un diagrama de dependencias en bucle en la consola de comandos con el siguiente error sintáctico:
  ```text
  Error: Cycle: module.compute.var.subnet_name, module.network.output.subnet_name
  ```
- **Causa:** Este problema ocurre cuando el flujo de datos no es estrictamente unidireccional. Si diseñas tus submódulos de manera que el módulo de red requiera una variable que solo se expone como salida del módulo de cómputo, y a su vez, el de cómputo requiere una salida del de red, se genera un bucle lógico. En Terraform, un módulo hijo no puede hacer referencias cruzadas circulares directamente con otro.
- **Solución:** Sigue la técnica de "Pasar por el Módulo Raíz" (*Root Module Orchestration*). Asegura que el flujo de datos siempre sea:
  1. El módulo de red (`modules/vpc`) crea los recursos base y expone únicamente outputs genéricos (ej. `subnet_name`).
  2. El módulo raíz recibe esos outputs y los pasa como variables directas (`inputs`) al módulo de cómputo (`modules/vm`).
  
  Evita declarar dependencias implícitas dentro de las configuraciones internas de los submódulos hijo. Revisa tus archivos `main.tf` y remueve cualquier variable de red que referencie elementos de cómputo directamente en su núcleo.

---

## Limpieza

Para evitar cargos inesperados en tu cuenta de Google Cloud Platform por recursos de red y procesamiento que no utilices, asegúrate de destruir todos los recursos creados al finalizar este laboratorio.

Sigue detenidamente estos pasos ordenados de limpieza:

1. Asegúrate de estar posicionado en el directorio raíz del laboratorio:
   ```bash
   cd /home/usuario/terraform-gcp-labs/
   ```

2. Ejecuta el plan de destrucción indicando la variable de proyecto específica:
   ```bash
   terraform destroy -auto-approve
   ```

3. Valida la salida final de Terraform confirmando la eliminación de todos los recursos aprovisionados:
   ```text
   Destroy complete! Resources: 4 destroyed.
   ```

4. Opcional: Elimina de forma local la caché de plugins y proveedores descargados de Terraform si deseas liberar espacio de disco en tu estación de trabajo:
   ```bash
   rm -rf .terraform/ .terraform.lock.hcl tfplan.binary
   ```

---

## Resumen

En este laboratorio práctico has completado con éxito la transición de un esquema de infraestructura monolítico a un modelo de desarrollo modular siguiendo las mejores prácticas operativas para Terraform y Google Cloud Platform.

### Conceptos Clave Aprendidos

- **Estructura Modular Local:** Has aprendido a construir la estructura canónica de archivos (`main.tf`, `variables.tf`, `outputs.tf`) dentro de directorios independientes para organizar mejor el código del proyecto.
- **Flujo de Datos Unidireccional:** Comprendiste cómo las variables de entrada actúan como parámetros de configuración del módulo y las salidas actúan como el retorno necesario para vincular recursos lógicos.
- **Evitar Dependencias Estáticas:** Ahora tus configuraciones de máquinas virtuales no tienen cadenas estáticas (*hardcoded*) ni de red, lo que permite reutilizar el módulo de cómputo en cualquier otra topología.
- **Mantenibilidad:** El código del módulo raíz se ha simplificado a menos de 30 líneas, permitiendo implementar políticas corporativas unificadas de manera limpia y legible.

### Recursos Adicionales

Para profundizar en la creación avanzada de módulos y estándares empresariales, consulta la siguiente documentación oficial de referencia:
- [HashiCorp: Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
- [Terraform Registry: Google Cloud Platform Modules](https://registry.terraform.io/namespaces/terraform-google-modules)
- [Google Cloud: Cloud Foundation Fabric Framework](https://github.com/GoogleCloudPlatform/cloud-foundation-fabric)
