# Crear múltiples entornos (dev, test, prod) con terraform workspace y observar separación del estado.

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 35 minutos |
| **Dificultad** | Media |
| **Nivel de Bloom** | Aplicar |

## Descripción General

Este laboratorio práctico se enfoca en la implementación del aislamiento lógico de entornos utilizando **Terraform Workspaces**. Construyendo sobre la base del backend remoto de Google Cloud Storage (GCS) configurado en prácticas anteriores, aprenderás a evitar la duplicación de código de infraestructura (HCL) mediante el uso de la variable dinámica `terraform.workspace`. 

A lo largo de este laboratorio, configurarás, desplegarás y verificarás la separación física de los archivos de estado (`terraform.tfstate`) bajo el prefijo `env:/` dentro de tu bucket remoto de GCS. Finalmente, realizarás pruebas del ciclo de vida de los espacios de trabajo, analizando cómo Terraform evita colisiones de nombres y recursos entre los entornos de Desarrollo (`dev`) y Producción (`prod`).

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Comprender el funcionamiento de **Terraform Workspaces** y su interacción directa con un backend de Google Cloud Storage (GCS).
- [ ] Implementar la interpolación dinámica con `${terraform.workspace}` para automatizar la nomenclatura y parametrización de recursos (VPCs, Instancias VM) según el entorno activo.
- [ ] Utilizar de manera efectiva los comandos CLI `terraform workspace (list, new, select, show)` para transicionar entre entornos de manera segura.
- [ ] Verificar y validar la estructura física interna de almacenamiento de estados bajo la ruta virtual `env:/` de Google Cloud Storage.

## Prerrequisitos

Antes de iniciar este laboratorio, asegúrate de contar con:
1. **Acceso a Google Cloud Platform**: Una cuenta activa de GCP con permisos de Propietario o Editor sobre un proyecto existente.
2. **Backend Remoto Preconfigurado**: El bucket de almacenamiento de estado remoto de GCS (`tf-state-lock-[PROJECT_ID]`) configurado y funcional de la práctica anterior.
3. **Variables de Entorno**: La variable de entorno `TF_VAR_project_id` debidamente configurada en la terminal.
4. **Conocimientos Previos**: Comprensión sólida de la inicialización de Terraform, el ciclo de vida de los recursos básicos de GCP (VPC y Compute Engine), y el concepto de aislamiento del State File.

## Entorno de Laboratorio

El laboratorio debe realizarse bajo las especificaciones técnicas estables detalladas a continuación.

### Herramientas y Versiones de Software Requeridas

| Software / Herramienta | Versión Exacta (Probada) | Arquitectura / Distribución | Licencia | Enlace de Referencia |
| :--- | :--- | :--- | :--- | :--- |
| **HashiCorp Terraform** | v1.8.2 | x86_64 / Linux (Debian/Ubuntu) | BUSL-1.1 | [Terraform v1.8.2](https://releases.hashicorp.com/terraform/1.8.2/) |
| **Google Cloud CLI** | v472.0.0 (gcloud) | x86_64 / Linux (Debian/Ubuntu) | Apache-2.0 | [Google Cloud SDK 472.0](https://cloud.google.com/sdk/docs/release-notes) |
| **Google Provider para Terraform**| v5.25.0 | Provider Plugin | MPL-2.0 | [Google Provider Registry](https://registry.terraform.io/providers/hashicorp/google/5.25.0) |

### Constantes del Laboratorio
* **Directorio de Trabajo Local**: `/home/usuario/terraform-gcp-labs/`
* **Región Predeterminada**: `us-central1`
* **Zona Predeterminada**: `us-central1-a`
* **Nomenclatura del Bucket**: `tf-state-lock-[PROJECT_ID]`

### Inicialización del Entorno de Consola
Antes de iniciar, prepara las variables de entorno de tu terminal interactiva ejecutando:

```bash
## Definir ID de Proyecto (Reemplaza con tu ID de Proyecto real de GCP)
export TF_VAR_project_id="tu-proyecto-gcp-id"

## Crear directorio de trabajo si no existe y posicionarse en él
mkdir -p /home/usuario/terraform-gcp-labs/
cd /home/usuario/terraform-gcp-labs/
```

---

## Instrucciones Paso a Paso

### Paso 1: Preparación del Directorio de Trabajo y Configuración del Backend

En este paso, estructuraremos los archivos iniciales de Terraform y definiremos el archivo de configuración del backend remoto (`backend.tf`) de modo que use dinámicamente tu bucket de GCS previamente configurado.

1. Navega al directorio raíz del laboratorio y limpia configuraciones locales residuales si existieran:
   ```bash
   cd /home/usuario/terraform-gcp-labs/
   rm -rf .terraform* terraform.tfstate*
   ```

2. Crea el archivo `backend.tf` reemplazando `REPLACE_WITH_PROJECT_ID` con tu ID del proyecto GCP real. Puedes automatizar esto con el siguiente comando en bash:
   ```bash
   cat <<EOF > backend.tf
   terraform {
     required_version = ">= 1.8.2"
     required_providers {
       google = {
         source  = "hashicorp/google"
         version = "5.25.0"
       }
     }
     backend "gcs" {
       bucket = "tf-state-lock-${TF_VAR_project_id}"
       prefix = "terraform/state"
     }
   }
   EOF
   ```

3. Crea el archivo `variables.tf` para parametrizar de manera segura el ID del proyecto, la región y la zona.
   ```bash
   cat <<EOF > variables.tf
   variable "project_id" {
     type        = string
     description = "ID del proyecto de Google Cloud (GCP)"
   }

   variable "region" {
     type        = string
     default     = "us-central1"
     description = "Región estándar para el despliegue"
   }

   variable "zone" {
     type        = string
     default     = "us-central1-a"
     description = "Zona de cómputo predeterminada"
   }
   EOF
   ```

* **Resultado Esperado**: Creación exitosa de los archivos de definición estructural sin variables duras ("hardcodeadas") de proyecto.
* **Verificación**: Confirma visualmente que los archivos existan usando `ls -lh`:
  ```bash
  ls -lh backend.tf variables.tf
  ```

---

### Paso 2: Creación de Código Adaptable con Workspace Interpolation

Implementaremos una lógica dinámica donde el nombre de la VPC, el de la VM y la capacidad de cómputo varíen automáticamente en base al espacio de trabajo activo (`terraform.workspace`). Esto nos permite desplegar recursos pequeños en desarrollo y optimizados en producción usando el mismo código exacto.

1. Crea el archivo principal `main.tf`:
   ```bash
   cat <<EOF > main.tf
   provider "google" {
     project = var.project_id
     region  = var.region
     zone    = var.zone
   }

   locals {
     # Definición del tipo de máquina según el entorno actual
     machine_types = {
       default = "e2-micro"
       dev     = "e2-micro"
       prod    = "e2-small"
     }
     
     # Selección basada en Lookup con fallback seguro a default
     current_machine_type = lookup(local.machine_types, terraform.workspace, "e2-micro")
   }

   # Red de VPC con nombre dinámico
   resource "google_compute_network" "vpc_network" {
     name                    = "vpc-\${terraform.workspace}"
     auto_create_subnetworks = true
   }

   # Instancia de Compute Engine con parámetros dinámicos
   resource "google_compute_instance" "vm_instance" {
     name         = "vm-\${terraform.workspace}"
     machine_type = local.current_machine_type
     zone         = var.zone

     boot_disk {
       initialize_params {
         image = "debian-cloud/debian-11"
       }
     }

     network_interface {
       network = google_compute_network.vpc_network.id
       access_config {
         # Asigna una dirección IP pública efímera para pruebas de conectividad
       }
     }

     labels = {
       environment = terraform.workspace
       provisioner = "terraform"
     }
   }
   EOF
   ```

2. Crea el archivo de salidas `outputs.tf` para inspeccionar el estado dinámico resultante:
   ```bash
   cat <<EOF > outputs.tf
   output "workspace_actual" {
     value       = terraform.workspace
     description = "El Workspace de Terraform actualmente activo"
   }

   output "instancia_nombre" {
     value       = google_compute_instance.vm_instance.name
     description = "Nombre asignado a la Instancia de Cómputo"
   }

   output "instancia_tipo_maquina" {
     value       = google_compute_instance.vm_instance.machine_type
     description = "Tipo de máquina desplegado"
   }

   output "vpc_nombre_creado" {
     value       = google_compute_network.vpc_network.name
     description = "Nombre asignado a la VPC"
   }
   EOF
   ```

* **Resultado Esperado**: Arquitectura dinámicamente mapeada escrita en archivos HCL. Note el uso de `\${terraform.workspace}` con barra de escape en el script Bash para asegurar que no se evalúe antes de escribirse en el archivo.
* **Verificación**: Realiza un cat a `main.tf` para asegurar que las variables queden escritas como `${terraform.workspace}` y no vacías.
  ```bash
  grep -F "terraform.workspace" main.tf
  ```

---

### Paso 3: Inicialización del Backend y Despliegue en Entorno de Desarrollo (dev)

Inicializaremos la configuración, crearemos el espacio de trabajo para desarrollo (`dev`) y desplegaremos la primera variación de la arquitectura.

1. Inicializa el directorio de trabajo de Terraform contra el backend de GCS:
   ```bash
   terraform init
   ```

2. Lista los workspaces existentes (por defecto verás únicamente `default` con un asterisco indicador):
   ```bash
   terraform workspace list
   ```

3. Crea el workspace de desarrollo (`dev`). Este comando creará el espacio y cambiará inmediatamente tu contexto activo a él:
   ```bash
   terraform workspace new dev
   ```

4. Verifica el workspace actual en pantalla:
   ```bash
   terraform workspace show
   ```

5. Ejecuta un plan de ejecución para corroborar la correcta interpolación. La máquina resultante debe ser `e2-micro` y el nombre de los recursos debe contener `-dev`:
   ```bash
   terraform plan
   ```

6. Despliega la infraestructura asociada al workspace `dev`:
   ```bash
   terraform apply -auto-approve
   ```

* **Resultado Esperado**: Aprovisionamiento exitoso de `vpc-dev` y `vm-dev` con un tamaño `e2-micro`.
* **Verificación**: Analiza los outputs del final del despliegue:
  ```text
  Outputs:
  instancia_nombre = "vm-dev"
  instancia_tipo_maquina = "e2-micro"
  vpc_nombre_creado = "vpc-dev"
  workspace_actual = "dev"
  ```

---

### Paso 4: Creación, Alternancia y Despliegue en Entorno de Producción (prod)

Ahora repetiremos el proceso de aprovisionamiento en un workspace aislado llamado `prod`. El código fuente detectará automáticamente el cambio de contexto y escalará el tipo de máquina a `e2-small` sin realizar cambios manuales en el archivo de configuración.

1. Crea el nuevo workspace `prod`:
   ```bash
   terraform workspace new prod
   ```

2. Confirma que te encuentras en el espacio `prod` y que `dev` aún existe:
   ```bash
   terraform workspace list
   ```
   *Deberías ver un asterisco junto a `prod` indicando que es el workspace activo.*

3. Realiza la planificación. Comprueba que el tipo de máquina ha mutado a `e2-small` y los recursos correspondientes tendrán el sufijo `-prod`:
   ```bash
   terraform plan
   ```

4. Despliega la infraestructura de producción:
   ```bash
   terraform apply -auto-approve
   ```

* **Resultado Esperado**: Aprovisionamiento sin colisiones del entorno de producción. Se crean `vpc-prod` y `vm-prod` en paralelo en la nube de GCP.
* **Verificación**: Confirma los resultados desde los Outputs finales:
  ```text
  Outputs:
  instancia_nombre = "vm-prod"
  instancia_tipo_maquina = "e2-small"
  vpc_nombre_creado = "vpc-prod"
  workspace_actual = "prod"
  ```

---

### Paso 5: Inspección Física de la Estructura de Estados en Google Cloud Storage

Con ambos entornos desplegados, examinaremos cómo organiza Terraform físicamente estos archivos de estado concurrentes dentro de un único bucket de GCS remoto.

1. Utiliza la CLI de Google Cloud (`gcloud`) para listar todos los objetos persistidos dentro de tu bucket:
   ```bash
   gcloud storage objects list gs://tf-state-lock-${TF_VAR_project_id}/ --recursive
   ```

* **Resultado Esperado**: Observarás una estructura jerárquica limpia. El estado predeterminado de un workspace común va a la ruta del prefijo principal, pero los estados creados mediante la funcionalidad de workspaces personalizados se aíslan en la ruta virtual `env:/[nombre_workspace]/`.
* **Verificación**: Debes observar exactamente la siguiente estructura de salida física en la terminal:
  ```text
  gs://tf-state-lock-[PROJECT_ID]/env:/dev/terraform/state/default.tfstate
  gs://tf-state-lock-[PROJECT_ID]/env:/prod/terraform/state/default.tfstate
  ```
  *(Nota: El archivo de estado por defecto no se verá aquí a menos que hayas desplegado recursos bajo el workspace `default`).*

---

## Validación y Pruebas

En esta sección, se aplicarán métodos de control rigurosos para validar el aislamiento y el comportamiento adaptativo de los workspaces, simulando además fallos humanos o de lógica comunes en entornos reales.

### Prueba 1: Verificación cruzada mediante la consola / CLI de Google Cloud
Verifica que ambas máquinas virtuales de Compute Engine se encuentren activas simultáneamente en tu proyecto con sus respectivas nomenclaturas y configuraciones específicas:

```bash
gcloud compute instances list --filter="labels.provisioner=terraform" --format="table(name, machineType, status, labels.environment)"
```

**Salida Esperada:**
```text
NAME     MACHINE_TYPE  STATUS   ENVIRONMENT
vm-dev   e2-micro      RUNNING  dev
vm-prod  e2-small      RUNNING  prod
```

---

### Prueba Adversa (Limitación / Control de Errores)
Para validar la resiliencia operativa y entender las limitaciones del motor de Terraform, ejecuta la siguiente prueba de inyección de error manual:

#### Intento de eliminación de Workspace Activo con Recursos Asociados
Intentaremos forzar la eliminación del espacio de trabajo `dev` mientras nos encontramos trabajando en él o mientras mantiene recursos activos aprovisionados.

1. Regresa al espacio `dev` temporalmente:
   ```bash
   terraform workspace select dev
   ```

2. Intenta eliminar el espacio de trabajo actual:
   ```bash
   terraform workspace delete dev
   ```
   **Resultado Esperado de la Terminal:**
   ```text
   Error: Instance cannot be deleted

   The workspace "dev" is your active workspace. You cannot delete the active
   workspace.
   ```

3. Cambia al workspace `prod` e intenta nuevamente la eliminación de `dev` (sin destruirlo antes):
   ```bash
   terraform workspace select prod
   ```
   ```bash
   terraform workspace delete dev
   ```
   **Resultado Esperado de la Terminal:**
   ```text
   Error: Workspace "dev" is not empty.

   The workspace "dev" contains resources. Deleting it would orphan these resources
   in your cloud provider. To delete anyway, use the -force flag.
   ```

**Análisis de Seguridad de IA / Terraform**: Este comportamiento nativo de Terraform bloquea la pérdida catastrófica o desvinculación de recursos ("orphaning"), previniendo que los estados se queden desasociados de los objetos reales de GCP. Requiere la supervisión explícita del operador para limpiar primero la infraestructura de dicho espacio de trabajo.

---

## Solución de Problemas

A continuación, se listan dos problemas operativos realistas documentados durante el uso de Terraform Workspaces en Google Cloud:

### Problema 1: El tipo de máquina no cambia al cambiar de Workspace
* **Síntomas**: Cambias de workspace utilizando `terraform workspace select prod` pero al ejecutar `terraform plan` el plan de ejecución indica que se va a desplegar o mantener una máquina tipo `e2-micro` en lugar de `e2-small`.
* **Causa Raíz**: Error de sintaxis o asignación en el bloque `locals` del archivo `main.tf`. Específicamente, la función `lookup` no está resolviendo el nombre dinámico del workspace contra las llaves del mapa debido a un error tipográfico en el nombre del workspace (ej. crear el workspace como `production` en lugar de `prod` que es la llave definida en el mapa local).
* **Solución**:
  1. Ejecuta `terraform workspace show` para verificar el nombre exacto de tu entorno activo.
  2. Confirma que la llave existe exactamente igual en tu bloque `locals` de `main.tf`:
     ```hcl
     machine_types = {
       dev  = "e2-micro"
       prod = "e2-small"
     }
     ```
  3. Si es necesario, recrea el workspace con el nombre correcto o agrega la nueva llave correspondiente en el archivo de configuración de variables locales de Terraform.

### Problema 2: Error "Bucket not found" o "Access Denied" durante `terraform init`
* **Síntomas**: Al arrancar el laboratorio o inicializar un nuevo workspace, Terraform arroja el siguiente error:
  ```text
  Error: Failed to get existing workspaces: storage: bucket "tf-state-lock-..." does not exist
  ```
* **Causa Raíz**: La variable de entorno `TF_VAR_project_id` no se exportó correctamente en la consola interactiva actual, o se ingresó un nombre de proyecto con errores de escritura, lo que genera que el bloque `backend "gcs"` intente acceder a un bucket inexistente.
* **Solución**:
  1. Confirma el ID de tu proyecto actual en gcloud:
     ```bash
     gcloud config get-value project
     ```
  2. Exporta nuevamente la variable con el valor verificado:
     ```bash
     export TF_VAR_project_id="ID_DE_PROYECTO_CONFIRMADO"
     ```
  3. Ejecuta el script de recreación del archivo `backend.tf` proporcionado en el Paso 1 para actualizar el nombre real del bucket y vuelve a correr `terraform init`.

---

## Limpieza

Para mitigar cargos de facturación en tu cuenta de Google Cloud y mantener las prácticas de higiene operativas de infraestructura descritas en los lineamientos del curso, realiza la destrucción ordenada de cada uno de los entornos desplegados en este laboratorio.

1. **Destrucción del Entorno de Producción (`prod`)**:
   Asegúrate de estar posicionado en el espacio de trabajo `prod`:
   ```bash
   terraform workspace select prod
   ```
   Destruye la infraestructura de producción:
   ```bash
   terraform destroy -auto-approve
   ```

2. **Destrucción del Entorno de Desarrollo (`dev`)**:
   Cambia de entorno hacia `dev`:
   ```bash
   terraform workspace select dev
   ```
   Destruye la infraestructura de desarrollo:
   ```bash
   terraform destroy -auto-approve
   ```

3. **Remoción de los Workspaces Vacíos**:
   Cambia al workspace predeterminado para poder eliminar los personalizados:
   ```bash
   terraform workspace select default
   ```
   Elimina los workspaces lógicos ahora vacíos:
   ```bash
   terraform workspace delete dev
   ```
   ```bash
   terraform workspace delete prod
   ```

4. **Confirmación Final de Limpieza**:
   Asegúrate de que solo exista el workspace default y de que no queden instancias activas en GCP:
   ```bash
   terraform workspace list
   gcloud compute instances list --filter="labels.provisioner=terraform"
   ```

---

## Resumen

### Puntos Clave Aprendidos

- **Aislamiento de Entornos**: Los workspaces de Terraform proporcionan una segmentación lógica de entornos muy simple a nivel del archivo de estado utilizando un único juego de archivos HCL de configuración.
- **Estructura en GCS**: Al interactuar con el backend nativo de `gcs`, los entornos personalizados se almacenan físicamente bajo rutas prefijadas con la sintaxis `env:/[workspace_name]/` para evitar colisiones catastróficas de sobreescritura del estado.
- **Inyección Dinámica**: La variable dinámica integrada `${terraform.workspace}` permite inyectar el nombre de entorno directamente a los recursos de Google Cloud, asegurando nombres únicos globales (ej. buckets, VPCs, nombres de VM) de forma automática.
- **Límites de los Workspaces**: Aprendiste que, aunque los workspaces son excelentes para gestionar copias similares de una misma infraestructura con variaciones pequeñas, para entornos con estructuras de seguridad radicalmente distintas o proyectos diferentes de GCP, se recomiendan estrategias alternativas como la separación física por carpetas/directorios.

### Recursos para Ampliar Conocimientos
- [Documentación Oficial de Workspaces en Terraform CLI](https://developer.hashicorp.com/terraform/cli/workspaces)
- [Mejores Prácticas de GCP para Infraestructura como Código con Terraform](https://cloud.google.com/docs/terraform/best-practices-for-terraform)
