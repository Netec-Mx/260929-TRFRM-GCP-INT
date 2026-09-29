# Crear múltiples subredes y validar valores de entrada.

## Metadatos

| Módulo | Duración | Complejidad | Nivel de Bloom |
| :--- | :--- | :--- | :--- |
| Módulo 6: Variables complejas y validación de entrada | 55 minutos | Media | Crear (Nivel 6) |

## Descripción General

En este laboratorio práctico, construirás un módulo y una configuración raíz de Terraform orientada a desplegar una Red de Nube Privada Virtual (VPC) y múltiples subredes de forma dinámica en Google Cloud Platform. Utilizarás mapas de objetos complejos junto con el meta-argumento `for_each` para controlar el aprovisionamiento. 

El foco principal del laboratorio es la resiliencia y el diseño defensivo de infraestructura. Implementarás bloques de validación personalizados (`validation`) utilizando funciones integradas de Terraform (`regex`, `can`, `contains`) y el atributo `nullable` para prevenir el ingreso de rangos CIDR o nombres inválidos antes de realizar cualquier llamada a las APIs de GCP. Los recursos generados en este laboratorio serán la base estructural del pipeline de Integración Continua de la siguiente práctica.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Construir un mapa dinámico de variables estructuradas con objetos complejos para la parametrización de subredes VPC.
- [ ] Diseñar e implementar reglas de validación personalizadas para comprobar de forma anticipada sintaxis de nombres y rangos de red IP en formato CIDR.
- [ ] Aplicar el atributo `nullable = false` para forzar comportamientos de fallback hacia valores predeterminados en variables críticas.
- [ ] Desplegar subredes dinámicamente utilizando el meta-argumento `for_each` acoplado con lógica condicional para el control de entornos.

## Prerrequisitos

Para completar con éxito este laboratorio, requieres:
1. **Conocimientos teóricos previos**:
   - Familiaridad con la sintaxis de HashiCorp Configuration Language (HCL), específicamente el tipo de datos compuesto `map(object(...))`.
   - Comprensión de los conceptos de direccionamiento de red IP (bloques CIDR como `/24`).
   - Comprensión del comando `for_each` y su rol en la instanciación de recursos repetitivos.
2. **Acceso de red y Cloud**:
   - Una cuenta activa de Google Cloud Platform (GCP) con un proyecto creado y permisos de Administrador de Red (`roles/compute.networkAdmin`) y Visor (`roles/viewer`).
   - Credenciales de GCP cargadas en la terminal de trabajo (ejecutando previamente `gcloud auth application-default login`).
3. **Estación de trabajo local**:
   - Una consola/terminal de comandos con acceso a internet sin restricciones corporativas de tipo proxy o firewall para puertos 80, 443 y SSH (22).

## Entorno de Laboratorio

El laboratorio debe realizarse con las siguientes especificaciones técnicas de hardware y software para garantizar la compatibilidad absoluta del código:

### Especificaciones de Hardware (Mínimo Recomendado)
- **CPU**: Arquitectura x86_64 o ARM64 (mínimo 2 núcleos).
- **Memoria RAM**: 8 GB o superior.
- **Espacio en Disco**: 10 GB de espacio libre.

### Versiones de Software Requeridas
| Herramienta / Proveedor | Versión Exacta | Origen de Descarga Oficial |
| :--- | :--- | :--- |
| **Terraform CLI** | 1.8.2 | [HashiCorp Releases (v1.8.2 Linux x86_64)](https://releases.hashicorp.com/terraform/1.8.2/) |
| **Google Provider para Terraform** | 5.25.0 | [Terraform Registry (GCP v5.25.0)](https://registry.terraform.io/providers/hashicorp/google/5.25.0) |
| **Google Cloud CLI (gcloud)** | 472.0.0 | [GCP CLI Archives](https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-472.0.0-linux-x86_64.tar.gz) |

### Constantes del Entorno del Laboratorio
- **Directorio Raíz de Trabajo**: `/home/usuario/terraform-gcp-labs/`
- **Subdirectorio del Laboratorio**: `/home/usuario/terraform-gcp-labs/lab-06-subnets/`
- **Variable de Identificación del Proyecto de GCP**: `TF_VAR_project_id`
- **Región Estándar**: `us-central1`

---

## Instrucciones Paso a Paso

### Paso 1: Preparación del Directorio de Trabajo y Configuración de Proveedores

Antes de escribir el código de validación, debes asegurar la estructura del directorio de trabajo y fijar las versiones de Terraform y del proveedor de Google Cloud de acuerdo con las especificaciones técnicas del curso.

1. Abre tu terminal de Linux e ingresa o crea el directorio raíz definido en las constantes.
```bash
mkdir -p /home/usuario/terraform-gcp-labs/lab-06-subnets
cd /home/usuario/terraform-gcp-labs/lab-06-subnets
```

2. Configura tu variable de entorno de GCP utilizando tu ID de proyecto real. Reemplaza `TU_PROYECTO_GCP_ID` por tu ID de proyecto:
```bash
export TF_VAR_project_id="TU_PROYECTO_GCP_ID"
```

3. Crea el archivo `providers.tf` para declarar de forma estricta las versiones del motor de Terraform y el proveedor de GCP a utilizar.
```bash
cat << 'EOF' > providers.tf
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
  region  = "us-central1"
}
EOF
```

4. Ejecuta el comando de inicialización para descargar el binario del proveedor de Google con la versión exacta indicada.
```bash
terraform init
```

*Salida esperada:*
```text
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/google versions matching "5.25.0"...
- Installing hashicorp/google v5.25.0...
- Installed hashicorp/google v5.25.0 (signed by HashiCorp)
...
Terraform has been successfully initialized!
```

---

### Paso 2: Declarar Variables con Bloques de Validación Avanzados

En este paso, declararás las variables necesarias para modelar la infraestructura de red. Utilizaremos el bloque `validation` y funciones incorporadas para validar la sintaxis de nombres, entornos permitidos y la validez de los rangos de subred IP (CIDR).

1. Crea el archivo `variables.tf` en el mismo directorio.
```bash
cat << 'EOF' > variables.tf
variable "project_id" {
  type        = string
  description = "El ID del proyecto de GCP donde se desplegarán los recursos."
}

variable "environment" {
  type        = string
  description = "Entorno lógico de despliegue de la red. Solo se permiten valores específicos."

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "El entorno especificado no es válido. Debe ser uno de los siguientes: dev, staging, prod."
  }
}

variable "vpc_name" {
  type        = string
  description = "El nombre de la red VPC que será creada."

  validation {
    condition     = can(regex("^[a-z]([-a-z0-9]*[a-z0-9])?$", var.vpc_name))
    error_message = "El nombre de la VPC debe cumplir con las normas de GCP: comenzar con una letra minúscula, terminar con un carácter alfanumérico y contener únicamente letras minúsculas, números o guiones."
  }
}

variable "subnets_config" {
  type = map(object({
    ip_cidr_range = string
    region        = string
    deploy        = bool
  }))
  description = "Mapa de configuraciones de subredes a desplegar en la VPC."

  # Validación 1: Verificar el formato CIDR correcto de la IP
  validation {
    condition = alltrue([
      for subnet_key, subnet in var.subnets_config :
      can(regex("^10\\.[0-9]{1,3}\\.[0-9]{1,3}\\.0/24$", subnet.ip_cidr_range))
    ])
    error_message = "El bloque CIDR de las subredes debe corresponder al espacio privado de red 10.X.Y.0/24."
  }

  # Validación 2: Restringir las regiones permitidas
  validation {
    condition = alltrue([
      for subnet_key, subnet in var.subnets_config :
      contains(["us-central1", "us-east1", "europe-west1"], subnet.region)
    ])
    error_message = "La región especificada para las subredes debe pertenecer a la lista de regiones aprobadas: us-central1, us-east1 o europe-west1."
  }
}

variable "network_routing_mode" {
  type        = string
  description = "Modo de enrutamiento para la VPC de GCP."
  default     = "REGIONAL"
  nullable    = false # Impide el envío intencional de valores nulos

  validation {
    condition     = contains(["REGIONAL", "GLOBAL"], var.network_routing_mode)
    error_message = "El modo de enrutamiento solo puede ser REGIONAL o GLOBAL."
  }
}
EOF
```

---

### Paso 3: Definir Recursos de Red y Lógica Dinámica (`for_each`)

A continuación, crearás la VPC principal de GCP y las subredes utilizando la información de variables. Solo se aprovisionarán las subredes que cumplan la propiedad condicional `deploy = true` definida en su objeto de variables.

1. Crea el archivo `main.tf`.
```bash
cat << 'EOF' > main.tf
## Crear la Red VPC Principal sin subredes automáticas
resource "google_compute_network" "custom_vpc" {
  name                    = "${var.environment}-${var.vpc_name}"
  auto_create_subnetworks = false
  routing_mode            = var.network_routing_mode
  project                 = var.project_id
}

## Filtrar dinámicamente el mapa de subredes que tienen "deploy = true"
locals {
  active_subnets = {
    for name, subnet in var.subnets_config :
    name => subnet if subnet.deploy == true
  }
}

## Desplegar las subredes utilizando for_each con el mapa filtrado
resource "google_compute_subnetwork" "dynamic_subnets" {
  for_each = local.active_subnets

  name          = "${var.environment}-subnet-${each.key}"
  ip_cidr_range = each.value.ip_cidr_range
  region        = each.value.region
  network       = google_compute_network.custom_vpc.id
  project       = var.project_id

  private_ip_google_access = true
}
EOF
```

2. Crea el archivo `outputs.tf` para exponer los datos estructurados que requerirá la práctica posterior de CI/CD.
```bash
cat << 'EOF' > outputs.tf
output "vpc_id" {
  value       = google_compute_network.custom_vpc.id
  description = "El ID único de la Red VPC creada."
}

output "vpc_name" {
  value       = google_compute_network.custom_vpc.name
  description = "El nombre de la Red VPC creada."
}

output "subnets_deployed" {
  value = {
    for name, subnet in google_compute_subnetwork.dynamic_subnets :
    name => {
      id     = subnet.id
      cidr   = subnet.ip_cidr_range
      region = subnet.region
    }
  }
  description = "Detalles de las subredes aprovisionadas activamente."
}
EOF
```

---

### Paso 4: Crear Configuración de Valores por Entorno (`.tfvars`)

Definiremos los datos de configuración reales para simular un entorno de desarrollo (`dev`). Notarás que una de las subredes tiene la propiedad `deploy = false` para verificar que el código filtre las subredes correctamente.

1. Crea el archivo `terraform.tfvars` con valores conformes a las reglas de validación establecidas.
```bash
cat << 'EOF' > terraform.tfvars
environment = "dev"
vpc_name    = "gcp-labs-vpc"

subnets_config = {
  "frontend" = {
    ip_cidr_range = "10.10.1.0/24"
    region        = "us-central1"
    deploy        = true
  },
  "backend" = {
    ip_cidr_range = "10.10.2.0/24"
    region        = "us-central1"
    deploy        = true
  },
  "database" = {
    ip_cidr_range = "10.10.3.0/24"
    region        = "us-east1"
    deploy        = false # No debe ser creada
  }
}

network_routing_mode = "REGIONAL"
EOF
```

---

### Paso 5: Validar y Ejecutar el Plan de Despliegue de Infraestructura

1. Ejecuta el proceso de validación semántica local de Terraform para asegurar que la estructura y tipos coincidan con los de las API.
```bash
terraform validate
```
*Salida esperada:*
```text
Success! The configuration is valid.
```

2. Realiza un plan de ejecución para evaluar qué recursos serán aprovisionados físicamente en la nube de GCP.
```bash
terraform plan
```

3. Revisa la salida detallada en tu terminal. El plan debe indicar la creación de la VPC y **únicamente dos subredes** (`frontend` y `backend`), dejando de lado la subred de `database` dado que `deploy = false`.

*Salida analítica clave:*
```text
Terraform will perform the following actions:

  # google_compute_network.custom_vpc will be created
  + resource "google_compute_network" "custom_vpc" { ... }

  # google_compute_subnetwork.dynamic_subnets["backend"] will be created
  + resource "google_compute_subnetwork" "dynamic_subnets" { ... }

  # google_compute_subnetwork.dynamic_subnets["frontend"] will be created
  + resource "google_compute_subnetwork" "dynamic_subnets" { ... }

Plan: 3 to add, 0 to change, 0 to destroy.
```

4. Aplica el plan a tu proyecto real de Google Cloud.
```bash
terraform apply -auto-approve
```

*Salida del aprovisionamiento exitoso:*
```text
google_compute_network.custom_vpc: Creating...
google_compute_network.custom_vpc: Creation complete after 15s [id=projects/.../global/networks/dev-gcp-labs-vpc]
google_compute_subnetwork.dynamic_subnets["backend"]: Creating...
google_compute_subnetwork.dynamic_subnets["frontend"]: Creating...
...
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

---

## Validación y Pruebas

Para asegurar la robustez de las validaciones de datos agregadas al código, someteremos a Terraform a un análisis bajo escenarios adversos e inválidos. Esto valida que la interceptación local de fallos (fail-fast) esté funcionando según los objetivos.

### Prueba 1: Inyección Adversaria de Entornos No Soportados

1. Intenta ejecutar un plan sobrescribiendo la variable `environment` con un valor no contemplado en el bloque `contains()`.
```bash
terraform plan -var="environment=qa"
```

2. Comprueba que Terraform aborta inmediatamente la fase de inicialización mostrando el mensaje de error estructurado y personalizado en tu código.

*Resultado Esperado:*
```text
╷
│ Error: Invalid value for variable
│ 
│   on variables.tf line 7:
│    7: variable "environment" {
│ 
│ El entorno especificado no es válido. Debe ser uno de los siguientes: dev, staging, prod.
│ 
│ This was checked by the validation rule at variables.tf:11,3-13.
╵
```

---

### Prueba 2: Solapamiento y Sintaxis de Redes Inválidas (Caso Adversario de Redes)

En esta prueba adversaria, intentaremos desplegar una subred que viola el direccionamiento del espacio IP privado interno corporativo (`10.X.Y.0/24`) forzando la inclusión de un rango público y de máscara inadecuada.

1. Ejecuta el plan intentando usar un rango fuera del mapa preestablecido:
```bash
terraform plan -var='subnets_config={"bad_subnet":{ip_cidr_range="192.168.1.100/32",region="us-central1",deploy=true}}'
```

2. Verifica la salida. El analizador de expresiones regulares local debe capturar el patrón erróneo y levantar la alerta diseñada por ti.

*Resultado Esperado:*
```text
╷
│ Error: Invalid value for variable
│ 
│   on variables.tf line 25:
│   25: variable "subnets_config" {
│ 
│ El bloque CIDR de las subredes debe corresponder al espacio privado de red 10.X.Y.0/24.
│ 
│ This was checked by the validation rule at variables.tf:33,3-13.
╵
```

---

### Prueba 3: Validación del Comportamiento `nullable = false`

1. Intenta pasar de forma explícita un valor de `null` al modo de enrutamiento. Al ser `nullable = false` y poseer un `default = "REGIONAL"`, Terraform debe rellenar automáticamente la variable con el valor predeterminado sin arrojar fallas críticas.
```bash
terraform plan -var="network_routing_mode=null"
```
2. Inspecciona el plan generado. Valida que el recurso `google_compute_network.custom_vpc` continúe mostrando el parámetro `routing_mode = "REGIONAL"` en lugar de arrojar error o desplegar con valores nulos que la API de GCP rechazaría de forma tardía.

---

## Solución de Problemas

En caso de encontrar errores comunes durante la ejecución de esta práctica, revisa las siguientes causas y sus respectivas resoluciones:

### Problema 1: Fallo de Inicialización con Versiones Erróneas de Terraform o Proveedores
* **Síntoma**: Mensajes del tipo `Error: Unsupported Terraform Core version` o `Error: Failed to query available provider packages`.
* **Causa**: Estás ejecutando una versión local de Terraform diferente a la especificada (`1.8.2`) en el bloque de configuración del archivo `providers.tf` o se ha cambiado manualmente la ruta del backend.
* **Solución**:
  1. Ejecuta `terraform -version` y asegura que coincida con la `1.8.2` requerida. Si no es así, descárgala del repositorio oficial indicado en el apartado de software.
  2. Si modificaste el archivo de proveedores y se generaron conflictos de caché de plugins, elimina la carpeta oculta de trabajo e inicializa de nuevo:
     ```bash
     rm -rf .terraform/ .terraform.lock.hcl
     terraform init
     ```

### Problema 2: El plan falla por el Mensaje de Validación de Mayúsculas y Puntos
* **Síntoma**: Error de sintaxis en Terraform al declarar bloques `validation`: `Error: Invalid validation error message`.
* **Causa**: Se definió un texto para `error_message` que no se ajusta a las convenciones estilísticas obligatorias de Terraform (comenzar con letra mayúscula y finalizar con un punto `.`).
* **Solución**: Corrige tus bloques de validación en `variables.tf` para que todos los mensajes terminen con un punto (`.`) y comiencen con mayúscula. Ejemplo: `error_message = "El enrutamiento solo puede ser REGIONAL o GLOBAL."`.

---

## Limpieza

Para evitar el consumo de créditos y costos indeseados en la facturación de tu cuenta de Google Cloud Platform, es mandatorio eliminar toda la infraestructura de pruebas creada.

1. Asegúrate de estar en el directorio correcto de trabajo:
```bash
cd /home/usuario/terraform-gcp-labs/lab-06-subnets
```

2. Ejecuta la destrucción completa de la red VPC y subredes creadas dinámicamente en el plan.
```bash
terraform destroy -auto-approve
```

*Salida esperada:*
```text
google_compute_subnetwork.dynamic_subnets["frontend"]: Destroying... [id=projects/.../subnetworks/dev-subnet-frontend]
google_compute_subnetwork.dynamic_subnets["backend"]: Destroying... [id=projects/.../subnetworks/dev-subnet-backend]
...
google_compute_network.custom_vpc: Destroying... [id=projects/.../global/networks/dev-gcp-labs-vpc]
...
Destroy complete! Resources: 3 destroyed.
```

---

## Resumen

En esta práctica implementaste con éxito las mejores prácticas para validación avanzada de entradas en Terraform. A través de la experimentación y pruebas adversarias completadas, lograste:

1. **Diseñar Código Defensivo**: Utilizaste el bloque `validation` para mitigar fallos tardíos, asegurando que solo configuraciones correctas en entornos y rangos de red privados fuesen enviadas a Google Cloud.
2. **Optimizar Despliegues Complejos**: Empleaste `for_each` y filtros lógicos locales para permitir que múltiples subredes de red fuesen aprovisionadas dinámicamente o excluidas condicionalmente con una sola bandera booleana (`deploy`).
3. **Manejar Ausencia de Datos de Entrada**: Comprendiste las implicaciones de `nullable = false` para forzar la adopción de valores por defecto cuando las llamadas de automatización omiten argumentos críticos.

### Recursos Adicionales de Estudio
- [Documentación oficial de Terraform: Custom Validation Rules](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules)
- [Funciones de Cadena y Expresiones Regulares en HCL](https://developer.hashicorp.com/terraform/language/functions/regex)
- [Políticas de Creación de Subredes en GCP VPC](https://cloud.google.com/vpc/docs/subnets)
