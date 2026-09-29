# Crear archivos .tfvars por entorno y outputs condicionales.

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 35 minutos |
| **Dificultad** | Alta (Hard) |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio práctico de nivel avanzado, refinarás la estrategia multi-entorno implementada anteriormente utilizando Workspaces de Terraform. Aprenderás a desacoplar de forma estricta la configuración del entorno de ejecución mediante el uso de archivos de asignación de variables externas (`dev.tfvars` y `prod.tfvars`). 

En lugar de incrustar lógica condicional compleja basada puramente en el nombre del workspace actual dentro del código de infraestructura (`main.tf`), parametrizarás los recursos a través de variables de tipo estructural avanzado (`object` y `map(string)`). Implementarás la creación condicional de una dirección IP externa estática mediante expresiones lógicas y bloques de configuración dinámicos (`dynamic blocks`), finalizando con el diseño de un output selectivo que controlará la exposición de información sensible basada en la lógica de negocio de cada entorno.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Construir esquemas de variables complejas de tipo `object` que contengan tipos mixtos, incluidos mapas y booleanos.
- [ ] Separar de forma limpia las configuraciones de entorno mediante archivos específicos de variables de Terraform (`.tfvars`).
- [ ] Implementar asignaciones condicionales y bloques dinámicos (`dynamic "access_config"`) basados en variables estructurales.
- [ ] Diseñar y evaluar outputs condicionales con lógica ternaria para proteger información exclusiva de entornos productivos.
- [ ] Gestionar de forma segura el despliegue concurrente de múltiples entornos lógicos sobre un backend remoto de Google Cloud Storage (GCS) con bloqueo de estado.

## Prerrequisitos

Para completar con éxito este laboratorio, debes cumplir con los siguientes requisitos:

1. **Conocimientos teóricos:**
   - Comprensión del ciclo de vida de Terraform (`init`, `plan`, `apply`, `destroy`).
   - Experiencia básica operando con Workspaces de Terraform.
   - Entendimiento sobre los tipos de datos en HCL, especialmente el tipo estructurado `object`.

2. **Accesos y credenciales:**
   - Acceso a una cuenta activa de Google Cloud Platform (GCP) con un proyecto aprovisionado.
   - Permisos de Administrador de IAM para gestionar recursos de red y cómputo (`Compute Admin`) y buckets de almacenamiento (`Storage Admin`).
   - Cuenta configurada localmente mediante la gcloud CLI.

## Entorno de Laboratorio

Este laboratorio está diseñado para ejecutarse en una estación de trabajo con sistema operativo Linux, macOS o Windows (utilizando WSL2).

### Componentes de Software Utilizados

| Herramienta | Versión Exacta | Origen de Descarga Oficial |
| :--- | :--- | :--- |
| **Terraform CLI** | v1.8.2 | [HashiCorp Releases](https://releases.hashicorp.com/terraform/1.8.2/) |
| **Google Cloud Provider** | v5.25.0 | [Terraform Registry](https://registry.terraform.io/providers/hashicorp/google/5.25.0) |
| **Google Cloud SDK (gcloud CLI)** | v478.0.0 | [Google Cloud SDK Downloads](https://cloud.google.com/sdk/docs/release-notes-478-0-0) |

### Constantes del Entorno

Asegura que tu entorno cumpla con las siguientes rutas de trabajo estándar y esquemas de nombres:
* **Directorio raíz de la práctica:** `/home/usuario/terraform-gcp-labs/`
* **ID del Proyecto GCP:** Debe configurarse en la variable de entorno `TF_VAR_project_id`.
* **Región estándar:** `us-central1`
* **Zona estándar:** `us-central1-a`
* **Nomenclatura del Bucket de Backend:** `tf-state-lock-[PROJECT_ID]`

### Comandos de Preparación del Entorno

Antes de comenzar la edición de archivos, ejecuta los siguientes comandos en tu terminal local para asegurar la creación del directorio de trabajo y la correcta autenticación en Google Cloud.

```bash
## 1. Crear y acceder al directorio del laboratorio
mkdir -p /home/usuario/terraform-gcp-labs/
cd /home/usuario/terraform-gcp-labs/

## 2. Configurar el ID del proyecto de GCP actual (Reemplaza con tu ID de proyecto real)
export TF_VAR_project_id="tu-proyecto-gcp-id"
gcloud config set project "$TF_VAR_project_id"

## 3. Autenticación frente a las APIs de Google Cloud
gcloud auth application-default login --no-launch-browser
```

## Instrucciones Paso a Paso

### Paso 1: Inicialización del Backend Remoto Compartido

En este paso prepararás el archivo de configuración del backend remoto utilizando Google Cloud Storage como backend compartido con soporte nativo de bloqueo de estado.

1. Crea el archivo `backend.tf` en el directorio raíz utilizando el siguiente bloque de configuración. El bucket de almacenamiento se inicializará dinámicamente utilizando parámetros de inicialización de Terraform para evitar valores estáticos ("hardcoded").

```hcl
## /home/usuario/terraform-gcp-labs/backend.tf

terraform {
  required_version = "1.8.2"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "5.25.0"
    }
  }
  backend "gcs" {
    # El bucket se pasará dinámicamente como argumento de inicialización para garantizar unicidad global.
    prefix = "terraform/state/lab05"
  }
}
```

2. Ejecuta la inicialización de Terraform. Si el bucket no ha sido creado previamente, créalo rápidamente a través de la consola de GCP o con `gcloud`:

```bash
## Crear el bucket si no existe (debe ser globalmente único)
gsutil mb -p "$TF_VAR_project_id" -l us-central1 "gs://tf-state-lock-$TF_VAR_project_id"

## Inicializar Terraform inyectando el nombre del bucket de almacenamiento remoto
terraform init -backend-config="bucket=tf-state-lock-$TF_VAR_project_id"
```

**Salida esperada de la terminal:**
```text
Initializing the backend...
Successfully configured the backend "gcs"! Terraform will now use this backend.

Initializing provider plugins...
- Finding hashicorp/google versions matching "5.25.0"...
- Installing hashicorp/google v5.25.0...
- Installed hashicorp/google v5.25.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

---

### Paso 2: Definición de Variables Estructurales Complejas

Definirás las variables de entrada de Terraform en el archivo `variables.tf`. Implementarás estructuras complejas tipo `object` que agrupen configuraciones lógicas de red y características de la instancia virtual.

1. Crea el archivo `variables.tf` con el siguiente código:

```hcl
## /home/usuario/terraform-gcp-labs/variables.tf

variable "project_id" {
  type        = string
  description = "ID del Proyecto en Google Cloud Platform."
}

variable "region" {
  type        = string
  default     = "us-central1"
  description = "Región de Google Cloud utilizada para alojar los recursos."
}

variable "zone" {
  type        = string
  default     = "us-central1-a"
  description = "Zona específica de GCP para el aprovisionamiento de la instancia."
}

variable "environment" {
  type        = string
  description = "Identificador del entorno lógico actual (por ejemplo: dev, prod)."
}

## Variable estructural para encapsular toda la parametrización de la máquina virtual
variable "instance_config" {
  type = object({
    machine_type = string
    boot_image   = string
    labels       = map(string)
  })
  description = "Configuración jerárquica de la máquina virtual de Compute Engine."
}

## Variable estructural para encapsular la parametrización de la infraestructura de red
variable "network_config" {
  type = object({
    vpc_name          = string
    subnet_cidr       = string
    reserve_static_ip = bool
  })
  description = "Estructura de configuración para la VPC y el direccionamiento externo."
}
```

**Verificación:** Ejecuta el comando de validación para asegurar la ausencia de errores de sintaxis en tus declaraciones de variables.
```bash
terraform validate
```
*Salida esperada:* `Success! The configuration is valid.`

---

### Paso 3: Diseño de los Archivos de Variables de Entorno (.tfvars)

Crearás los archivos de variables separados `dev.tfvars` y `prod.tfvars`. Cada archivo definirá valores adaptados para simular políticas operativas y financieras realistas (desarrollo ágil de bajo costo vs. producción robusta y de alta disponibilidad).

1. Crea el archivo `dev.tfvars`:

```hcl
## /home/usuario/terraform-gcp-labs/dev.tfvars

environment = "dev"

instance_config = {
  machine_type = "e2-micro" # Tipo de máquina optimizado para bajo costo en desarrollo
  boot_image   = "debian-cloud/debian-11"
  labels = {
    entorno     = "desarrollo"
    centro_costo = "investigacion"
    origen      = "terraform"
  }
}

network_config = {
  vpc_name          = "vpc-dev-custom"
  subnet_cidr       = "10.10.10.0/24"
  reserve_static_ip = false # No se requiere IP pública externa estática en desarrollo
}
```

2. Crea el archivo `prod.tfvars`:

```hcl
## /home/usuario/terraform-gcp-labs/prod.tfvars

environment = "prod"

instance_config = {
  machine_type = "e2-medium" # Mayor CPU y memoria asignados para tráfico productivo
  boot_image   = "debian-cloud/debian-11"
  labels = {
    entorno     = "produccion"
    centro_costo = "core-it"
    origen      = "terraform"
  }
}

network_config = {
  vpc_name          = "vpc-prod-custom"
  subnet_cidr       = "10.20.10.0/24"
  reserve_static_ip = true # Requerido: IP pública reservada estática para DNS corporativo
}
```

---

### Paso 4: Implementación de la Infraestructura de GCP con Lógica Condicional Dinámica

En este paso, escribirás el archivo de aprovisionamiento principal `main.tf`. Emplearás lógica ternaria, recursos basados en conteo condicional (`count`) y bloques dinámicos (`dynamic`) para construir un flujo de infraestructura flexible.

1. Crea el archivo `main.tf` con el siguiente código:

```hcl
## /home/usuario/terraform-gcp-labs/main.tf

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

## Red VPC Personalizada
resource "google_compute_network" "vpc" {
  name                    = var.network_config.vpc_name
  auto_create_subnetworks = false
}

## Subred Única del Entorno
resource "google_compute_subnetwork" "subnet" {
  name          = "${var.network_config.vpc_name}-subnetwork"
  ip_cidr_range = var.network_config.subnet_cidr
  network       = google_compute_network.vpc.id
  region        = var.region
}

## Recurso Condicional: Reserva de IP Externa Estática
## Se creará únicamente si reserve_static_ip está configurado como true (Producción)
resource "google_compute_address" "static_ip" {
  count  = var.network_config.reserve_static_ip ? 1 : 0
  name   = "ip-estatica-${var.environment}"
  region = var.region
}

## Instancia de Cómputo Principal
resource "google_compute_instance" "vm" {
  name         = "instancia-servicio-${var.environment}"
  machine_type = var.instance_config.machine_type
  zone         = var.zone

  labels = var.instance_config.labels

  boot_disk {
    initialize_params {
      image = var.instance_config.boot_image
    }
  }

  network_interface {
    network    = google_compute_network.vpc.id
    subnetwork = google_compute_subnetwork.subnet.id

    # Bloque dinámico condicional para gestionar la conectividad WAN externa:
    # Si reserve_static_ip es true, aprovisiona un bloque access_config asociando la IP estática reservada.
    # Si es false, no se declara ningún bloque access_config (la VM de desarrollo no poseerá direccionamiento público).
    dynamic "access_config" {
      for_each = var.network_config.reserve_static_ip ? [1] : []
      content {
        nat_ip = google_compute_address.static_ip[0].address
      }
    }
  }
}
```

---

### Paso 5: Implementación de Outputs Condicionales de Seguridad

Diseñarás las salidas de Terraform (`outputs.tf`) de tal forma que expongan información sensible basada en decisiones lógicas de negocio, enmascarando los datos o respondiendo con alertas semánticas controladas.

1. Crea el archivo `outputs.tf` con el siguiente bloque de código:

```hcl
## /home/usuario/terraform-gcp-labs/outputs.tf

output "vm_name" {
  value       = google_compute_instance.vm.name
  description = "Nombre oficial de la máquina virtual instanciada."
}

## Output condicional utilizando lógica ternaria para proteger los endpoints
output "static_public_ip" {
  value       = var.network_config.reserve_static_ip ? google_compute_address.static_ip[0].address : "ACCESO_WAN_RESTRINGIDO_ENTORNO_DESARROLLO"
  description = "IP externa pública asignada. Sólo visible para entornos con reserva IP habilitada."
}
```

---

### Paso 6: Ejecución, Despliegue Paralelo en Workspaces y Pruebas de Flujo

Llevarás a cabo el aprovisionamiento aislado en dos entornos lógicos mediante el uso de workspaces de Terraform y la inyección controlada de tus archivos `.tfvars`.

1. Crea y cambia al workspace `development`:
```bash
terraform workspace new development
terraform workspace select development
```

2. Planifica y ejecuta el despliegue del entorno de desarrollo pasando el archivo de variables adecuado:
```bash
terraform plan -var-file="dev.tfvars" -out=plan_dev.tfplan
```

*Analiza la salida del plan:* Observa que **no** se aprovisionará el recurso de dirección IP externa estática (`google_compute_address.static_ip`) y la máquina virtual no dispondrá de bloque `access_config`.

3. Aplica los cambios en desarrollo:
```bash
terraform apply plan_dev.tfplan
```

**Salida parcial esperada de los Outputs de Desarrollo:**
```text
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

static_public_ip = "ACCESO_WAN_RESTRINGIDO_ENTORNO_DESARROLLO"
vm_name = "instancia-servicio-dev"
```

4. Crea y cambia al workspace `production`:
```bash
terraform workspace new production
terraform workspace select production
```

5. Planifica y despliega el entorno de producción aplicando el archivo de variables correspondiente:
```bash
terraform plan -var-file="prod.tfvars" -out=plan_prod.tfplan
```

*Analiza la salida del plan:* Notarás que se aprovisionará el recurso `google_compute_address.static_ip` y el bloque dinámico `access_config` de la VM mapeará su propiedad `nat_ip` al primer índice de dicha IP.

6. Aplica los cambios en producción:
```bash
terraform apply plan_prod.tfplan
```

**Salida parcial esperada de los Outputs de Producción:**
```text
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

static_public_ip = "34.135.210.45"   # (Esta IP variará en función de la asignación dinámica de GCP)
vm_name = "instancia-servicio-prod"
```

---

## Validación y Pruebas

Para garantizar el cumplimiento de los objetivos de diseño de este laboratorio, ejecuta las siguientes actividades de comprobación empírica.

### Prueba 1: Verificación de Aislamiento de Estados con Workspace y Backend Remoto

Verifica que el estado de desarrollo y producción permanezcan completamente independientes en Google Cloud Storage mediante el backend remoto.

```bash
## Listar los archivos generados dentro del bucket de backend remoto usando gsutil
gsutil ls -r "gs://tf-state-lock-$TF_VAR_project_id/terraform/state/lab05/**"
```

*Resultado esperado:* La salida debe reflejar dos rutas diferentes en la estructura jerárquica del bucket para cada uno de los entornos creados:
- `gs://tf-state-lock-[PROJECT_ID]/terraform/state/lab05/` (Estado por defecto o root)
- `gs://tf-state-lock-[PROJECT_ID]/terraform/state/lab05/workspace_key_dir/development/default.tfstate`
- `gs://tf-state-lock-[PROJECT_ID]/terraform/state/lab05/workspace_key_dir/production/default.tfstate`

---

### Prueba 2: Caso Adversario — Detección de Colisiones de Tipos de Datos (Validación de Esquema Estricto)

Para experimentar cómo actúan los tipos de datos estricto `object` ante datos incorrectos, edita intencionalmente tu archivo `dev.tfvars` agregando un campo inválido o un tipo de datos inconsistente (por ejemplo, define un valor numérico para la propiedad `boot_image` en `instance_config` o añade un atributo inexistente al objeto).

1. Abre `dev.tfvars` y modifica de forma errónea la configuración temporalmente:

```hcl
## Cambio adverso intencional para forzar error de tipado estricto
instance_config = {
  machine_type = "e2-micro"
  boot_image   = 123456789  # Un entero en lugar de un String
  labels = {
    entorno = "desarrollo"
  }
}
```

2. Ejecuta una planificación:
```bash
terraform plan -var-file="dev.tfvars"
```

*Resultado esperado:* Terraform arrojará un error inmediato bloqueando el proceso de ejecución mucho antes de intentar comunicarse con las APIs de Google Cloud Platform. Esto demuestra la ventaja crítica del modelado estructural con tipos estrictos:

```text
╷
│ Error: Invalid value for input variable
│ 
│   on dev.tfvars line 5:
│   5: instance_config = {
│   6:   machine_type = "e2-micro"
│   7:   boot_image   = 123456789
│   8:   labels = {
│   9:     entorno = "desarrollo"
│  10:   }
│  11: }
│ 
│ boot_image: string required.
╵
```

3. **Restaura** el archivo `dev.tfvars` con su contenido original antes de continuar con la sección de limpieza.

---

## Solución de Problemas

A continuación, se describen dos escenarios de falla típicos durante la ejecución de este laboratorio, junto con las estrategias específicas para remediarlos:

### Problema 1: Error "Bucket Not Found" o Bloqueos al Inicializar el Backend Remoto
* **Síntoma:** Al ejecutar `terraform init -backend-config="..."`, se visualiza un mensaje de error indicando: `Error: Failed to get existing workspaces: storage: bucket doesn't exist` o errores relacionados con permisos HTTP 403.
* **Causa:** El ID de proyecto definido en `$TF_VAR_project_id` contiene caracteres no válidos o el bucket de almacenamiento de estado remoto de Google Cloud Storage no fue creado en el proyecto actual debido a problemas de nomenclatura única global.
* **Solución:** Ejecuta los siguientes comandos para reconstruir el bucket local de control asegurándote de que coincida de forma unívoca con tu ID de proyecto de Google Cloud:
  ```bash
  # Confirmar ID de proyecto actual
  gcloud config get-value project
  # Forzar recreación del bucket con permisos adecuados
  gsutil mb -p "$(gcloud config get-value project)" -l us-central1 "gs://tf-state-lock-$(gcloud config get-value project)"
  # Re-inicializar backend
  terraform init -reconfigure -backend-config="bucket=tf-state-lock-$(gcloud config get-value project)"
  ```

### Problema 2: Error "Dynamic block 'access_config' refers to invalid resource address"
* **Síntoma:** Al ejecutar un plan o apply sobre el workspace de desarrollo, se genera el error: `Error: Invalid index on google_compute_address.static_ip[0].address... count is 0`.
* **Causa:** En entornos de desarrollo (`dev.tfvars`), `reserve_static_ip` está configurado en `false`, lo que significa que el recurso de Terraform `google_compute_address.static_ip` tiene una longitud de lista cero debido al parámetro condicional `count = 0`. Al intentar acceder al índice `[0]` dentro del bloque dinámico o el output condicional de forma incorrecta, Terraform lanza un error de índice fuera de rango.
* **Solución:** Revisa las expresiones condicionales de tu código. Asegura que el acceso de configuración dynamic `"access_config"` en tu `main.tf` tenga la siguiente forma, asegurando que el bucle `for_each` evalúe a una lista vacía `[]` si `reserve_static_ip` es falso, previniendo que se evalúe la expresión inaccesible `google_compute_address.static_ip[0]`:
  ```hcl
  dynamic "access_config" {
    for_each = var.network_config.reserve_static_ip ? [1] : []
    content {
      nat_ip = google_compute_address.static_ip[0].address
    }
  }
  ```

---

## Limpieza

Es fundamental liberar los recursos aprovisionados para evitar cargos no deseados en la cuenta de facturación de Google Cloud.

```bash
## 1. Asegurar la limpieza del entorno productivo
terraform workspace select production
terraform destroy -var-file="prod.tfvars" -auto-approve

## 2. Asegurar la limpieza del entorno de desarrollo
terraform workspace select development
terraform destroy -var-file="dev.tfvars" -auto-approve

## 3. Retornar al workspace default y eliminar los entornos adicionales
terraform workspace select default
terraform workspace delete production
terraform workspace delete development

## 4. Eliminar el bucket de almacenamiento de estado si no se requiere persistencia
gsutil rm -r "gs://tf-state-lock-$TF_VAR_project_id"
```

## Resumen

En este laboratorio has consolidado tu capacidad para construir infraestructuras reutilizables, modulares y seguras mediante el desacoplamiento de configuraciones de entorno de la lógica de código central de Terraform.

### Puntos Clave Aprendidos
- **Uso de `.tfvars` separados:** Los archivos `.tfvars` permiten gestionar variables específicas de entorno de manera limpia, evitando sobrecargar el archivo `main.tf` con múltiples condicionales complejas basadas en workspaces.
- **Tipado estructural con `object`:** Permite agrupar atributos relacionados, lo que resulta en un código más robusto, validado a nivel sintáctico en fase de planificación antes de interactuar con las APIs del proveedor de la nube.
- **Mapeo y bloques dinámicos (`dynamic`)**: El uso estratégico de `dynamic` junto con expresiones booleanas permite controlar la creación de sub-componentes (como configuraciones de interfaces de red con o sin IPs públicas) de forma óptima sin duplicar código de infraestructura.
- **Seguridad en Outputs:** Utilizando operadores condicionales binarios o ternarios, puedes limitar la visualización de direcciones IP, DNS u otros secretos de infraestructura de acuerdo al alcance y las políticas de cada entorno.
