# Escanear código con tfsec

## Metadatos

| Dimensión | Detalle |
| :--- | :--- |
| **Duración** | 20 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio práctico, el estudiante aplicará un cinturón de herramientas de aseguramiento y análisis estático sobre código de Terraform diseñado para Google Cloud Platform (GCP). Utilizando el entorno local `/home/usuario/terraform-gcp-labs/`, se sembrará intencionalmente un escenario con vulnerabilidades de seguridad comunes (como firewalls excesivamente permisivos y buckets de almacenamiento expuestos). Posteriormente, el estudiante instalará, ejecutará y configurará herramientas clave del ecosistema de aseguramiento como `tfsec`, `tflint` y `checkov` para detectar estos riesgos antes de realizar un despliegue, y autogenerará la documentación técnica del módulo utilizando `terraform-docs`.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [x] **Integrar** herramientas de análisis estático (`tfsec`, `tflint` y `checkov`) para evaluar la seguridad y calidad del código de Terraform.
- [x] **Identificar y corregir** brechas de seguridad críticas como reglas de firewall con origen de red inseguro (`0.0.0.0/0`) o buckets de GCS sin controles de acceso unificados.
- [x] **Generar** de forma automática la documentación técnica de variables, salidas y recursos de un módulo de Terraform en formato Markdown utilizando `terraform-docs`.
- [x] **Evaluar** críticamente el uso de comentarios de evasión de reglas de seguridad en entornos de revisión humana de código.

## Prerrequisitos

Para completar este laboratorio de forma exitosa, se requiere:
- **Conocimientos teóricos**: Comprensión del paradigma de seguridad "Shift-Left" y del funcionamiento básico de sintaxis HCL en Terraform.
- **Acceso a Sistemas**: Una estación de trabajo con sistema operativo Linux (Ubuntu 22.04 LTS o superior es recomendado para compatibilidad de comandos nativos en el directorio `/home/usuario/terraform-gcp-labs/`).
- **Conexión de Red**: Conexión a Internet sin restricciones de firewall saliente para descargar los binarios oficiales de las herramientas de auditoría.

## Entorno de Laboratorio

El laboratorio se ejecutará de forma local dentro del directorio raíz estándar. A continuación se listan las especificaciones de hardware y software requeridas:

### Requisitos de Hardware mínimos

| Componente | Requisito Mínimo | Requisito Recomendado |
| :--- | :--- | :--- |
| **Procesador (CPU)** | x86_64 o ARM64 (2 núcleos) | x86_64 o ARM64 (4 núcleos) |
| **Memoria RAM** | 8 GB | 16 GB |
| **Espacio de Disco** | 10 GB libres | 20 GB libres |

### Requisitos de Software y Herramientas

| Herramienta | Versión Exacta | Arquitectura de Destino | Enlace de Descarga Oficial |
| :--- | :--- | :--- | :--- |
| **Terraform CLI** | v1.8.2 | linux_amd64 / linux_arm64 | [HashiCorp Releases](https://releases.hashicorp.com/terraform/1.8.2/) |
| **tfsec** | v1.28.5 | linux_amd64 / linux_arm64 | [AquaSecurity tfsec GitHub](https://github.com/aquasecurity/tfsec/releases/tag/v1.28.5) |
| **tflint** | v0.50.3 | linux_amd64 / linux_arm64 | [TFLint GitHub Releases](https://github.com/terraform-linters/tflint/releases/tag/v0.50.3) |
| **checkov** | v3.2.100 | Python Package (PyPI) | [Checkov PyPI Page](https://pypi.org/project/checkov/3.2.100/) |
| **terraform-docs** | v0.17.0 | linux_amd64 / linux_arm64 | [Terraform-docs GitHub](https://github.com/terraform-docs/terraform-docs/releases/tag/v0.17.0) |

> **Nota sobre Herramientas de IA y Licencias**: En el caso de que utilices asistentes basados en Inteligencia Artificial (como Microsoft 365 Copilot, GitHub Copilot Chat en VS Code, o ChatGPT) para interpretar o corregir el código HCL vulnerable reportado por los escáneres de este laboratorio, ten en cuenta que estas herramientas requieren licenciamiento específico (como la suscripción activa de GitHub Copilot o planes corporativos de OpenAI). Este laboratorio está diseñado para ejecutarse de forma autosuficiente sin necesidad imperativa de un asistente IA, aunque estos pueden complementar la velocidad de remediación si se usan de forma supervisada.

### Comandos de Inicialización del Entorno

Ejecuta los siguientes comandos en tu terminal local para asegurar la creación del directorio de trabajo estructurado y descargar las utilidades necesarias:

```bash
## 1. Crear el árbol de directorios de trabajo
mkdir -p /home/usuario/terraform-gcp-labs/modules/compute
cd /home/usuario/terraform-gcp-labs/

## 2. Descargar e instalar de forma local los binarios requeridos en un directorio accesible del PATH o de forma directa
mkdir -p /home/usuario/terraform-gcp-labs/bin
export PATH="/home/usuario/terraform-gcp-labs/bin:$PATH"

## Descarga de tfsec v1.28.5 (Ejemplo para Linux AMD64)
curl -sSLo /home/usuario/terraform-gcp-labs/bin/tfsec https://github.com/aquasecurity/tfsec/releases/download/v1.28.5/tfsec-linux-amd64
chmod +x /home/usuario/terraform-gcp-labs/bin/tfsec

## Descarga de tflint v0.50.3
curl -sSLo tflint.zip https://github.com/terraform-linters/tflint/releases/download/v0.50.3/tflint_linux_amd64.zip
unzip -o tflint.zip -d /home/usuario/terraform-gcp-labs/bin/
rm tflint.zip

## Descarga de terraform-docs v0.17.0
curl -sSLo terraform-docs.tar.gz https://github.com/terraform-docs/terraform-docs/releases/download/v0.17.0/terraform-docs-v0.17.0-linux-amd64.tar.gz
tar -xzf terraform-docs.tar.gz -C /home/usuario/terraform-gcp-labs/bin/ terraform-docs
rm terraform-docs.tar.gz

## Verificar versiones instaladas
tfsec --version
tflint --version
terraform-docs --version
```

---

## Instrucciones Paso a Paso

### Paso 1: Preparación del código con brechas de seguridad

**Objetivo**: Crear un escenario de configuración realista con fallas críticas de seguridad y calidad de código de Terraform en GCP para evaluar la efectividad de las herramientas.

**Instrucciones**:

1. Posiciónate en la carpeta del módulo de cómputo:
   ```bash
   cd /home/usuario/terraform-gcp-labs/modules/compute
   ```

2. Crea un archivo `main.tf` con fallas intencionales severas de seguridad (regla de firewall SSH abierta al público global, bucket de almacenamiento sin control de acceso uniforme, y almacenamiento de una API key harcodeada en texto plano para simular filtración de secretos). Copia el siguiente contenido textualmente:

   ```hcl
   # /home/usuario/terraform-gcp-labs/modules/compute/main.tf

   terraform {
     required_version = ">= 1.8.2"
     required_providers {
       google = {
         source  = "hashicorp/google"
         version = "5.25.0"
       }
     }
   }

   # Recurso 1: Red VPC y Subred para Pruebas
   resource "google_compute_network" "vpc_red" {
     name                    = "vpc-seguridad-test"
     auto_create_subnetworks = false
   }

   resource "google_compute_subnetwork" "subnet_test" {
     name          = "subnet-seguridad-test"
     ip_cidr_range = "10.0.1.0/24"
     region        = var.gcp_region
     network       = google_compute_network.vpc_red.id
     # FALLA DE SEGURIDAD 1: No tiene habilitados los logs de flujo (Flow Logs)
   }

   # Recurso 2: Regla de Firewall Insegura
   resource "google_compute_firewall" "firewall_inseguro" {
     name    = "fw-allow-ssh-public"
     network = google_compute_network.vpc_red.name

     allow {
       protocol = "tcp"
       ports    = ["22"]
     }

     # FALLA DE SEGURIDAD 2: Exposición de puerto administrativo SSH a internet global (0.0.0.0/0)
     source_ranges = ["0.0.0.0/0"]
   }

   # Recurso 3: Bucket de Google Cloud Storage vulnerable
   resource "google_storage_bucket" "bucket_inseguro" {
     name          = "tf-state-lock-test-insecure-bucket"
     location      = "US"
     force_destroy = true

     # FALLA DE SEGURIDAD 3: No se bloquea el acceso público y no se activa Uniform Bucket-Level Access
     # Tampoco cuenta con encriptación administrada por el cliente (KMS) ni versionamiento.
   }

   # Recurso 4: Instancia de máquina virtual con IP externa y metadatos sensibles
   resource "google_compute_instance" "vm_insegura" {
     name         = "vm-servidor-datos"
     machine_type = "e2-micro"
     zone         = "us-central1-a"

     boot_disk {
       initialize_params {
         image = "debian-cloud/debian-11"
       }
     }

     network_interface {
       network    = google_compute_network.vpc_red.id
       subnetwork = google_compute_subnetwork.subnet_test.id

       # FALLA DE SEGURIDAD 4: Habilitar IP pública efímera de forma directa (access_config presente)
       access_config {
         // Genera IP externa pública
       }
     }

     # FALLA DE SEGURIDAD 5: Metadata con llaves API / secretos hardcodeados
     metadata = {
       api-key-secreta = "AIzaSyD-InsecureFakeApiKeyForTestingPurposes"
     }
   }
   ```

3. Crea un archivo `variables.tf` para declarar las variables necesarias de este módulo:
   ```hcl
   # /home/usuario/terraform-gcp-labs/modules/compute/variables.tf

   variable "gcp_region" {
     type        = string
     description = "Región predeterminada para el despliegue de recursos en GCP"
     default     = "us-central1"
   }
   ```

4. Crea un archivo `outputs.tf` básico para las salidas de la arquitectura:
   ```hcl
   # /home/usuario/terraform-gcp-labs/modules/compute/outputs.tf

   output "vpc_id" {
     value       = google_compute_network.vpc_red.id
     description = "ID único de la red VPC simulada"
   }

   output "vm_external_ip" {
     value       = google_compute_instance.vm_insegura.network_interface[0].access_config[0].nat_ip
     description = "Dirección IP pública asignada a la máquina de prueba"
   }
   ```

**Output Esperado**:
Archivos creados correctamente dentro de `/home/usuario/terraform-gcp-labs/modules/compute`.

**Verificación**:
```bash
ls -la /home/usuario/terraform-gcp-labs/modules/compute/
```
Deberás observar los archivos `main.tf`, `variables.tf`, y `outputs.tf` reflejados en la salida de terminal.

---

### Paso 2: Análisis estático con tflint

**Objetivo**: Evaluar la calidad, el estilo de codificación del lenguaje HCL y detectar advertencias generales del proveedor de GCP mediante `tflint`.

**Instrucciones**:

1. Asegúrate de estar en el directorio raíz del módulo (`/home/usuario/terraform-gcp-labs/modules/compute`).
2. Inicializa la configuración de `tflint` ejecutando un análisis local básico sobre el directorio del módulo actual. Como `tflint` puede validar reglas generales, ejecuta el comando de inicialización del linter:
   ```bash
   tflint --init
   ```
3. Ejecuta el análisis de calidad con `tflint`:
   ```bash
   tflint
   ```

**Output Esperado**:
```text
(Si no se ha configurado un archivo de reglas tflint.hcl complejo, el linter completará de forma exitosa sin mostrar errores graves o enfocándose únicamente en advertencias generales si el código cumple sintácticamente con la declaración nativa).
```

4. Para forzar una validación de estilo nativa sobre problemas del proveedor, crea un archivo de configuración `.tflint.hcl` en el directorio actual:
   ```hcl
   # /home/usuario/terraform-gcp-labs/modules/compute/.tflint.hcl
   config {
     module = true
     force  = false
   }

   plugin "terraform" {
     enabled = true
     preset  = "recommended"
   }
   ```
5. Inicializa `tflint` nuevamente para que lea la configuración y ejecute el escaneo:
   ```bash
   tflint --init
   tflint
   ```

**Verificación**:
Si hay errores menores de estilo o llamadas incorrectas, `tflint` los imprimirá en la consola con el número de línea. Si la sintaxis básica y nomenclatura del proveedor de GCP de Terraform son válidas de forma nativa, no habrá salidas de bloqueo (indicando que tflint comprueba principalmente sintaxis y llamadas válidas antes de aplicar reglas profundas de seguridad).

---

### Paso 3: Auditoría y escaneo de vulnerabilidades con tfsec

**Objetivo**: Ejecutar el escáner de seguridad proactivo `tfsec` para descubrir los fallos críticos de seguridad de infraestructura sembrados en el código.

**Instrucciones**:

1. Ejecuta el binario de `tfsec` apuntando directamente al módulo de cómputo donde guardamos el código:
   ```bash
   tfsec /home/usuario/terraform-gcp-labs/modules/compute
   ```

**Output Esperado**:
`tfsec` analizará los recursos definidos y mostrará un informe estructurado detallando las vulnerabilidades encontradas, su gravedad (Critical, High, Medium, Low) y las reglas de CIS violadas. La terminal mostrará un reporte similar a este:

```text
  Result #1 HIGH Subnetwork does not have flow logs enabled. 
  ────────────────────────────────────────────────────────────────────────────────
    /home/usuario/terraform-gcp-labs/modules/compute/main.tf:19-24
  ────────────────────────────────────────────────────────────────────────────────
    See https://aquasecurity.github.io/tfsec/v1.28.5/checks/google/compute/enable-vpc-flow-logs/ for more info.
  ────────────────────────────────────────────────────────────────────────────────

  Result #2 CRITICAL Firewall rule allows inbound traffic from anywhere to port 22. 
  ────────────────────────────────────────────────────────────────────────────────
    /home/usuario/terraform-gcp-labs/modules/compute/main.tf:27-39
  ────────────────────────────────────────────────────────────────────────────────
    See https://aquasecurity.github.io/tfsec/v1.28.5/checks/google/compute/no-public-ingress/ for more info.
  ────────────────────────────────────────────────────────────────────────────────

  Result #3 CRITICAL Bucket does not have public access prevention. 
  ────────────────────────────────────────────────────────────────────────────────
    /home/usuario/terraform-gcp-labs/modules/compute/main.tf:42-47
  ────────────────────────────────────────────────────────────────────────────────
    See https://aquasecurity.github.io/tfsec/v1.28.5/checks/google/storage/enable-ubla/ for more info.
  ────────────────────────────────────────────────────────────────────────────────

  Result #4 CRITICAL Instance has an external IP address assigned. 
  ────────────────────────────────────────────────────────────────────────────────
    /home/usuario/terraform-gcp-labs/modules/compute/main.tf:50-71
  ────────────────────────────────────────────────────────────────────────────────
    See https://aquasecurity.github.io/tfsec/v1.28.5/checks/google/compute/no-public-ip/ for more info.
  ────────────────────────────────────────────────────────────────────────────────

  Result #5 CRITICAL Metadata key 'api-key-secreta' contains a sensitive pattern/value. 
  ────────────────────────────────────────────────────────────────────────────────
    /home/usuario/terraform-gcp-labs/modules/compute/main.tf:73-75
  ────────────────────────────────────────────────────────────────────────────────

  BREADCRUMB: 5 potential problems detected.
```

2. **Remediación del código**: Corrige las brechas de seguridad identificadas modificando tu archivo `/home/usuario/terraform-gcp-labs/modules/compute/main.tf`. Reemplaza el archivo por una versión segura y hardening de GCP:

   ```hcl
   # /home/usuario/terraform-gcp-labs/modules/compute/main.tf (Corregido y Asegurado)

   terraform {
     required_version = ">= 1.8.2"
     required_providers {
       google = {
         source  = "hashicorp/google"
         version = "5.25.0"
       }
     }
   }

   # Red VPC y Subred con LOGS DE FLUJO ACTIVADOS
   resource "google_compute_network" "vpc_red" {
     name                    = "vpc-seguridad-test"
     auto_create_subnetworks = false
   }

   resource "google_compute_subnetwork" "subnet_test" {
     name          = "subnet-seguridad-test"
     ip_cidr_range = "10.0.1.0/24"
     region        = var.gcp_region
     network       = google_compute_network.vpc_red.id

     # SOLUCIÓN DE SEGURIDAD 1: Logs de flujo habilitados
     log_config {
       aggregation_interval = "INTERVAL_5_SEC"
       flow_sampling        = 0.5
       metadata             = "INCLUDE_ALL_METADATA"
     }
   }

   # Firewall con ORIGEN RESTRINGIDO (Únicamente la VPN o rangos corporativos internos, ej: 10.0.0.0/8)
   resource "google_compute_firewall" "firewall_inseguro" {
     name    = "fw-allow-ssh-restricted"
     network = google_compute_network.vpc_red.name

     allow {
       protocol = "tcp"
       ports    = ["22"]
     }

     # SOLUCIÓN DE SEGURIDAD 2: Solo permitir tráfico de administración interno
     source_ranges = ["10.0.0.0/8"]
   }

   # Bucket de GCS con Control de Acceso Uniforme, Encriptación (por defecto) y Bloqueo de Acceso Público
   resource "google_storage_bucket" "bucket_inseguro" {
     name                        = "tf-state-lock-test-secure-bucket"
     location                    = "US"
     force_destroy               = true
     
     # SOLUCIÓN DE SEGURIDAD 3: Habilitar Uniform Bucket-Level Access y bloquear acceso público
     uniform_bucket_level_access = true
     public_access_prevention    = "enforced"

     versioning {
       enabled = true
     }
   }

   # Instancia de VM sin IP externa (Sin access_config)
   resource "google_compute_instance" "vm_insegura" {
     name         = "vm-servidor-datos"
     machine_type = "e2-micro"
     zone         = "us-central1-a"

     boot_disk {
       initialize_params {
         image = "debian-cloud/debian-11"
       }
     }

     network_interface {
       network    = google_compute_network.vpc_red.id
       subnetwork = google_compute_subnetwork.subnet_test.id
       
       # SOLUCIÓN DE SEGURIDAD 4: Se eliminó access_config para evitar la exposición a internet público.
     }

     # SOLUCIÓN DE SEGURIDAD 5: Metadata libre de secretos expuestos en texto plano.
     metadata = {
       rol-servidor = "servidor-procesamiento-interno"
     }
   }
   ```

3. Vuelve a ejecutar el comando de análisis de seguridad con `tfsec`:
   ```bash
   tfsec /home/usuario/terraform-gcp-labs/modules/compute
   ```

**Verificación**:
La terminal de `tfsec` debe devolver un mensaje de éxito indicando que no se han detectado brechas de seguridad potenciales (`No problems detected!`).

---

### Paso 4: Análisis complementario de cumplimiento con checkov

**Objetivo**: Correr un análisis complementario utilizando la biblioteca de auditoría `checkov` para validar cumplimiento normativo más estricto de GCP.

**Instrucciones**:

1. En el entorno de laboratorio (asumiendo que `checkov` está disponible a nivel de sistema o pre-instalado, o bien omitiéndose si el software de sistema de la máquina no posee Python, ejecutando la validación del binario):
   ```bash
   # Comprobar si checkov está disponible
   if command -v checkov &> /dev/null; then
       checkov -d /home/usuario/terraform-gcp-labs/modules/compute
   else
       echo "La herramienta checkov no está instalada localmente, simulando resultado..."
   fi
   ```
2. Si tienes `checkov` instalado por medio de Python pip (`pip install checkov==3.2.100`), la terminal ejecutará un framework que evalúa las políticas CIS. Al correr sobre el código ya corregido en el Paso 3, observará cómo las políticas críticas de bucket uniform access y firewalls ahora pasan de forma satisfactoria.

---

### Paso 5: Generación de documentación técnica con terraform-docs

**Objetivo**: Automatizar la creación de la ficha técnica del módulo de Terraform para que quede completamente documentado en formato Markdown, siguiendo las mejores prácticas de gobernanza e infraestructura madura.

**Instrucciones**:

1. Asegúrate de estar en el directorio del módulo:
   ```bash
   cd /home/usuario/terraform-gcp-labs/modules/compute
   ```
2. Ejecuta el comando de `terraform-docs` para generar la tabla de documentación de variables, recursos y salidas:
   ```bash
   terraform-docs markdown table --output-file README.md --output-mode inject ./
   ```
3. Alternativamente, si el archivo `README.md` no existía, créalo de forma automática usando la salida estándar redirigida:
   ```bash
   terraform-docs markdown table ./ > README.md
   ```

**Output Esperado**:
Un archivo `README.md` es generado localmente en `/home/usuario/terraform-gcp-labs/modules/compute/README.md`.

**Verificación**:
Inspecciona el contenido del documento generado con el comando `cat`:
```bash
cat README.md
```

El contenido resultante deberá coincidir estructuralmente con una tabla limpia de autogeneración técnica:

```markdown
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.8.2 |
| <a name="requirement_google"></a> [google](#requirement\_google) | 5.25.0 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_google"></a> [google](#provider\_google) | 5.25.0 |

## Resources

| Name | Type |
|------|------|
| [google_compute_firewall.firewall_inseguro](https://registry.terraform.io/providers/hashicorp/google/5.25.0/docs/resources/compute_firewall) | resource |
| [google_compute_instance.vm_insegura](https://registry.terraform.io/providers/hashicorp/google/5.25.0/docs/resources/compute_instance) | resource |
| ... | ... |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_gcp_region"></a> [gcp\_region](#input\_gcp\_region) | Región predeterminada para el despliegue de recursos en GCP | `string` | `"us-central1"` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_vpc_id"></a> [vpc\_id](#output\_vpc\_id) | ID único de la red VPC simulada |
| ... | ... |
```

---

## Validación y Pruebas

Para validar el éxito del laboratorio e implementar un control de calidad medible que sirva como evidencia técnica de cumplimiento, se deben realizar las siguientes pruebas de consistencia:

### Prueba de Regresión de Seguridad (Análisis Inverso)
Ejecuta el escáner de seguridad `tfsec` redirigiendo la salida a un archivo JSON para auditorías externas de control de calidad de Pipelines de CI/CD:

```bash
tfsec /home/usuario/terraform-gcp-labs/modules/compute --format json --out report-seguridad.json
```

Verifica que el archivo se haya generado y que el conteo de vulnerabilidades sea cero analizando su tamaño o contenido con un parseador como `jq` (si está instalado) o leyendo su estructura:

```bash
cat report-seguridad.json | grep -i "results"
```

### Caso de Prueba Adversario: Intento de bypass mediante comentarios de evasión

En las metodologías DevOps es crítico validar escenarios adversarios. Algunos ingenieros intentan forzar despliegues inseguros omitiendo alertas del linter mediante directivas del compilador (comentarios mágicos).

1. Edita temporalmente el archivo `/home/usuario/terraform-gcp-labs/modules/compute/main.tf` e introduce intencionalmente una regla insegura, pero agregando un bypass para engañar a `tfsec`:

   ```hcl
   # Regla de prueba bypass
   #tfsec:ignore:google-compute-no-public-ingress
   resource "google_compute_firewall" "firewall_bypass" {
     name    = "fw-bypass-ssh"
     network = google_compute_network.vpc_red.name

     allow {
       protocol = "tcp"
       ports    = ["22"]
     }
     source_ranges = ["0.0.0.0/0"] # Esto es inseguro, pero ignorado por el tag superior
   }
   ```

2. Ejecuta `tfsec` de nuevo:
   ```bash
   tfsec /home/usuario/terraform-gcp-labs/modules/compute
   ```

3. **Análisis e Interpretación Obligatoria del Resultado**: 
   * La herramienta `tfsec` mostrará un conteo exitoso de análisis pero registrará una alerta de "Ignored" (Línea evadida de forma consciente). 
   * **Criterio de Evaluación Humana**: En un pipeline empresarial, el uso indebido de directivas `#tfsec:ignore` sin una justificación aprobada por el equipo de seguridad de la información constituye un fallo de conformidad. Un revisor humano o un script de control automatizado debe analizar el archivo `.json` de salida para asegurarse de que no se hayan inyectado evasiones injustificadas en el repositorio de IaC.

4. Elimina el bloque `firewall_bypass` de tu archivo `main.tf` antes de continuar para conservar un código limpio y seguro.

---

## Solución de Problemas

A continuación, se presentan dos problemas operacionales reales con su correspondiente explicación técnica de causa y solución detallada.

### Problema 1: Fallo al descargar o ejecutar el binario de tfsec en sistemas con arquitecturas heterogéneas

- **Síntoma**: Al ejecutar `tfsec --version` o correr un escaneo, la terminal arroja el error:
  `bash: ./tfsec: cannot execute binary file: Exec format error`
- **Causa**: Se descargó un ejecutable compilado para una arquitectura de procesador incompatible. Por ejemplo, se descargó el binario de arquitectura `linux_amd64` (x86 de 64 bits) en un servidor de desarrollo basado en procesadores ARM64 (como las máquinas virtuales de la serie Tau T2A en GCP o Macs con Apple Silicon utilizando contenedores Linux).
- **Resolución**: Detecta la arquitectura nativa del sistema con `uname -m`. Si la salida arroja `aarch64` o `arm64`, vuelve a descargar el binario correcto apuntando a la arquitectura de destino:
  ```bash
  curl -sSLo /home/usuario/terraform-gcp-labs/bin/tfsec https://github.com/aquasecurity/tfsec/releases/download/v1.28.5/tfsec-linux-arm64
  chmod +x /home/usuario/terraform-gcp-labs/bin/tfsec
  ```

### Problema 2: El plugin de tflint falla al inicializar las reglas del proveedor de GCP

- **Síntoma**: Al ejecutar `tflint --init`, el proceso se detiene de forma indefinida o muestra un error de conexión de red:
  `Error: Failed to install plugin 'google'. Connection refused/timeout.`
- **Causa**: Un entorno de desarrollo local detrás de un servidor proxy corporativo o firewall restrictivo bloquea las peticiones externas al dominio de descargas de GitHub APIs.
- **Resolución**: Configura la variable de entorno del proxy correspondiente en la sesión de terminal antes de realizar la inicialización de tflint:
  ```bash
  export HTTP_PROXY="http://tu-proxy-corporativo.com:8080"
  export HTTPS_PROXY="http://tu-proxy-corporativo.com:8080"
  tflint --init
  ```
  Si el entorno carece por completo de conexión a Internet a nivel corporativo, desactiva temporalmente el bloque de plugin externo de GCP en el archivo `.tflint.hcl` borrando las líneas correspondientes a la regla y utiliza `tflint` solo para validaciones de formato de sintaxis HCL estándar sin plugins remotos.

---

## Limpieza

Para evitar acumulación de archivos innecesarios que confundan los entornos lógicos en prácticas posteriores y mantener limpia la estación de trabajo `/home/usuario/terraform-gcp-labs/`, sigue las siguientes directrices de remediación de archivos locales:

1. Elimina el archivo `.json` temporal generado como reporte de auditoría:
   ```bash
   rm -f /home/usuario/terraform-gcp-labs/modules/compute/report-seguridad.json
   ```
2. Asegúrate de que las herramientas descargadas no queden huérfanas en directorios temporales:
   ```bash
   rm -f /home/usuario/terraform-gcp-labs/modules/compute/.tflint.hcl
   ```
3. Dado que este laboratorio no realizó llamadas de aprovisionamiento reales (como `terraform apply`) en la nube de Google Cloud Platform para mitigar de forma activa costos operacionales innecesarios, **no se requiere ejecutar comandos de destrucción en GCP** (`terraform destroy`). El estado de GCP no ha sido modificado.

---

## Resumen

En esta práctica, has avanzado de forma significativa en la adopción del paradigma **Shift-Left** para el diseño de infraestructura como código madura en Google Cloud Platform. 

- Aprendiste a instalar y orquestar herramientas de análisis estático como `tflint` y `tfsec` directamente desde tu espacio de trabajo `/home/usuario/terraform-gcp-labs/`.
- Identificaste y mitigaste fallas de seguridad críticas en recursos de cómputo y almacenamiento antes de que fueran aprovisionados en el entorno de GCP.
- Descubriste cómo las herramientas automatizadas como `terraform-docs` reducen la carga operativa de los desarrolladores manteniendo la documentación de los módulos en sincronía exacta con los inputs, outputs y recursos de Terraform.

### Enlaces de Referencia Adicionales
- [Documentación del motor de análisis estático tfsec](https://aquasecurity.github.io/tfsec/v1.28.5/)
- [Prácticas recomendadas para el aseguramiento de cargas de trabajo en GCP (CISO Guide)](https://cloud.google.com/security/infrastructure)
- [Repositorio Oficial de Integraciones de terraform-docs](https://terraform-docs.io/)

---

# Generar doc automática con terraform-docs

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 20 minutos |
| **Complejidad** | Fácil |
| **Nivel de Bloom** | Aplicar |

## Descripción General

En esta práctica de laboratorio, diseñarás e implementarás un módulo de red personalizado para Google Cloud Platform (GCP) que incluye una VPC, subredes y reglas de firewall. A través de este proceso, aprenderás a estructurar de manera óptima las entradas (inputs) y salidas (outputs) utilizando metadatos descriptivos completos y tipos de datos explícitos. 

Posteriormente, configurarás e integrarás la herramienta de código abierto `terraform-docs` mediante un archivo de control centralizado `.terraform-docs.yml`. Esto te permitirá generar y actualizar automáticamente la documentación técnica del módulo en un archivo `README.md` dinámico y estandarizado, eliminando la necesidad de actualizaciones manuales propensas a errores de formato o sincronización.

[VISUAL: 10-11-0001 - Flujo de generación de documentación automatizada con terraform-docs a partir de metadatos HCL]

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Diseñar y estructurar un módulo de Terraform para GCP con tipado estricto, descripciones detalladas y valores por defecto óptimos.
- [ ] Instalar y configurar `terraform-docs` mediante un archivo de parámetros estructurado `.terraform-docs.yml`.
- [ ] Generar un archivo `README.md` auto-documentado utilizando tablas dinámicas para inputs, outputs, recursos y requisitos.
- [ ] Ejecutar pruebas de inyección y validación de sintaxis para evaluar la robustez y la seguridad de la documentación autogenerada.

## Prerrequisitos

Antes de comenzar esta práctica, asegúrate de cumplir con los siguientes requisitos conceptuales y técnicos:

### 1. Conocimientos Previos
- Comprensión básica de la sintaxis HCL (HashiCorp Configuration Language), específicamente la declaración de variables (`variable`) y salidas (`output`).
- Familiaridad con la terminal de comandos Linux y el formato Markdown.

### 2. Software y Herramientas Requeridas
Para asegurar la reproducibilidad del laboratorio, debes contar con las siguientes versiones exactas del software instalado en tu estación de trabajo:
- **HashiCorp Terraform CLI**: Versión `1.8.5` (Arquitectura Linux x86_64, Licencia: Mozilla Public License v2.0).
  - [Enlace de descarga oficial de Terraform v1.8.5](https://releases.hashicorp.com/terraform/1.8.5/)
- **terraform-docs**: Versión `v0.18.0` (Arquitectura Linux x86_64, Licencia: MIT License).
  - [Enlace de descarga oficial de terraform-docs v0.18.0](https://github.com/terraform-docs/terraform-docs/releases/tag/v0.18.0)
- **Google Cloud SDK (gcloud CLI)**: Versión `472.0.0` o superior (Opcional para este laboratorio de documentación local, pero requerido para validar compatibilidad de tipos).

---

## Entorno de Laboratorio

La estación de trabajo preestablecida utiliza la ruta `/home/usuario/terraform-gcp-labs/` como el directorio raíz para todo el desarrollo.

### Estructura de Directorios del Proyecto
Al finalizar, tu espacio de trabajo local deberá tener exactamente la siguiente estructura:

```text
/home/usuario/terraform-gcp-labs/
└── lab10-00-02/
    ├── .terraform-docs.yml
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
```

### Comandos de Preparación del Entorno

1. Inicia sesión en la terminal de tu estación de trabajo.
2. Crea el directorio de trabajo específico para este laboratorio:
   ```bash
   mkdir -p /home/usuario/terraform-gcp-labs/lab10-00-02
   cd /home/usuario/terraform-gcp-labs/lab10-00-02
   ```

3. Instala `terraform-docs` versión `v0.18.0` localmente en caso de que no esté disponible en tu sistema:
   ```bash
   # Descargar el binario oficial compilado para Linux x86_64
   curl -Lo terraform-docs.tar.gz https://github.com/terraform-docs/terraform-docs/releases/download/v0.18.0/terraform-docs-v0.18.0-linux-amd64.tar.gz
   
   # Extraer el binario
   tar -xzf terraform-docs.tar.gz terraform-docs
   
   # Moverlo a un directorio ejecutable local del usuario o del sistema
   mkdir -p /home/usuario/.local/bin
   mv terraform-docs /home/usuario/.local/bin/
   
   # Agregar al PATH si no está configurado (puedes añadir esto a ~/.bashrc)
   export PATH="/home/usuario/.local/bin:$PATH"
   
   # Limpiar el archivo comprimido descargado
   rm -f terraform-docs.tar.gz
   ```

4. Valida la instalación correcta de ambas herramientas:
   ```bash
   terraform -version
   terraform-docs --version
   ```

   **Salida esperada:**
   ```text
   Terraform v1.8.5
   on linux_amd64
   
   terraform-docs version v0.18.0 2d9a60e linux/amd64
   ```

---

## Instrucciones Paso a Paso

Sigue detenidamente las instrucciones para diseñar el módulo de infraestructura de red y automatizar su documentación.

### Paso 1: Inicialización del directorio de trabajo y estructura del módulo

**Objective**: Crear los archivos fuente vacíos para el módulo de red y establecer el punto de partida estructural de las plantillas de Terraform.

**Instructions**:
1. Asegúrate de estar posicionado en el directorio del laboratorio `/home/usuario/terraform-gcp-labs/lab10-00-02`.
2. Ejecuta el comando de creación para los archivos clave de Terraform:
   ```bash
   touch main.tf variables.tf outputs.tf README.md
   ```

**Expected output**:
La ejecución de `ls -la` en el directorio actual debe retornar la lista de archivos creados con tamaño de 0 bytes:
```text
total 8
drwxr-xr-x 2 usuario usuario 4096 Oct 24 10:00 .
drwxr-xr-x 3 usuario usuario 4096 Oct 24 10:00 ..
-rw-r--r-- 1 usuario usuario    0 Oct 24 10:00 main.tf
-rw-r--r-- 1 usuario usuario    0 Oct 24 10:00 outputs.tf
-rw-r--r-- 1 usuario usuario    0 Oct 24 10:00 README.md
-rw-r--r-- 1 usuario usuario    0 Oct 24 10:00 variables.tf
```

**Verification**: No debe haber errores en la terminal. El comando `touch` se completa de forma silenciosa.

---

### Paso 2: Creación del código HCL del módulo de red de GCP

**Objective**: Desarrollar la lógica del módulo de red de GCP incluyendo tipos estrictos, descripciones minuciosas y valores por defecto lógicos para facilitar la posterior auto-documentación.

**Instructions**:
1. Abre el archivo `variables.tf` con tu editor de texto favorito (por ejemplo, `nano` o `vim`) y define las variables de entrada de tu módulo con el siguiente contenido:

   ```hcl
   # /home/usuario/terraform-gcp-labs/lab10-00-02/variables.tf

   variable "project_id" {
     type        = string
     description = "El ID del proyecto de Google Cloud Platform (GCP) donde se desplegarán los recursos de red de este módulo."
   }

   variable "network_name" {
     type        = string
     description = "El nombre único que se asignará a la VPC (Virtual Private Cloud)."
     default     = "vpc-custom-main"
   }

   variable "routing_mode" {
     type        = string
     description = "El modo de enrutamiento de red para la VPC. Los valores válidos son 'REGIONAL' o 'GLOBAL'."
     default     = "REGIONAL"

     validation {
       condition     = contains(["REGIONAL", "GLOBAL"], var.routing_mode)
       error_message = "El modo de enrutamiento (routing_mode) debe ser 'REGIONAL' o 'GLOBAL'."
     }
   }

   variable "subnets" {
     type = list(object({
       subnet_name           = string
       subnet_ip             = string
       subnet_region         = string
       private_ip_google_access = bool
     }))
     description = "Lista detallada de subredes personalizadas que se crearán dentro de la VPC."
     default = [
       {
         subnet_name              = "subnet-dev-us-central1"
         subnet_ip                = "10.10.10.0/24"
         subnet_region            = "us-central1"
         private_ip_google_access = true
       }
     ]
   }

   variable "firewall_rules" {
     type = list(object({
       name      = string
       direction = string
       priority  = number
       ranges    = list(string)
       allow = list(object({
         protocol = string
         ports    = list(string)
       }))
     }))
     description = "Conjunto de reglas de firewall personalizadas asociadas a la VPC para controlar el tráfico entrante/saliente."
     default     = []
   }
   ```

2. Guarda y cierra el archivo `variables.tf`.

3. Abre el archivo `main.tf` y escribe la lógica de provisión de la red VPC, subredes y reglas de firewall. El código del módulo aprovecha la estructura de variables anterior de manera óptima:

   ```hcl
   # /home/usuario/terraform-gcp-labs/lab10-00-02/main.tf

   terraform {
     required_version = ">= 1.8.5"
     required_providers {
       google = {
         source  = "hashicorp/google"
         version = ">= 5.25.0"
       }
     }
   }

   # Creación de la VPC
   resource "google_compute_network" "vpc_network" {
     name                    = var.network_name
     project                 = var.project_id
     auto_create_subnetworks = false
     routing_mode            = var.routing_mode
   }

   # Creación de múltiples subredes dinámicas
   resource "google_compute_subnetwork" "subnets" {
     for_each = { for sub in var.subnets : sub.subnet_name => sub }

     name                     = each.value.subnet_name
     ip_cidr_range            = each.value.subnet_ip
     region                   = each.value.subnet_region
     network                  = google_compute_network.vpc_network.id
     project                  = var.project_id
     private_ip_google_access = each.value.private_ip_google_access
   }

   # Creación dinámica de reglas de firewall
   resource "google_compute_firewall" "firewall_rules" {
     for_each = { for rule in var.firewall_rules : rule.name => rule }

     name        = each.value.name
     network     = google_compute_network.vpc_network.name
     project     = var.project_id
     direction   = each.value.direction
     priority    = each.value.priority
     source_ranges = each.value.direction == "INGRESS" ? each.value.ranges : null
     destination_ranges = each.value.direction == "EGRESS" ? each.value.ranges : null

     dynamic "allow" {
       for_each = each.value.allow
       content {
         protocol = allow.value.protocol
         ports    = allow.value.ports
       }
     }
   }
   ```

4. Guarda y cierra el archivo `main.tf`.

5. Abre el archivo `outputs.tf` y define los parámetros de salida resultantes del aprovisionamiento:

   ```hcl
   # /home/usuario/terraform-gcp-labs/lab10-00-02/outputs.tf

   output "network_id" {
     value       = google_compute_network.vpc_network.id
     description = "El identificador completamente calificado (URI) de la red VPC creada."
   }

   output "network_name" {
     value       = google_compute_network.vpc_network.name
     description = "El nombre de la red VPC creada."
   }

   output "subnets_map" {
     value       = { for name, subnet in google_compute_subnetwork.subnets : name => subnet.id }
     description = "Un mapa asociativo que mapea los nombres de las subredes creadas con sus respectivos identificadores (URIs) en GCP."
   }
   ```

6. Guarda y cierra el archivo `outputs.tf`.

7. Ejecuta la validación sintáctica nativa de Terraform para certificar la consistencia lógica del código HCL:
   ```bash
   terraform fmt
   terraform validate
   ```

**Expected output**:
La salida en consola de `terraform validate` debe indicar que la configuración es correcta:
```text
Success! The configuration is valid.
```

**Verification**: Si recibes errores de sintaxis, asegúrate de haber copiado con exactitud todos los bloques de código y llaves de cierre (`}`) en cada uno de los archivos.

---

### Paso 3: Configuración del archivo de control .terraform-docs.yml

**Objective**: Definir los parámetros de formato, orden, encabezados y filtros que `terraform-docs` empleará para maquetar el `README.md`.

**Instructions**:
1. En el mismo directorio, crea y abre un archivo de configuración llamado `.terraform-docs.yml`:
   ```bash
   nano .terraform-docs.yml
   ```

2. Escribe la siguiente especificación YAML de configuración:

   ```yaml
   # /home/usuario/terraform-gcp-labs/lab10-00-02/.terraform-docs.yml
   formatter: "markdown table"

   version: ""

   header-from: main.tf

   content: |
     # Módulo de Red Personalizado de GCP

     Este módulo de Terraform permite aprovisionar redes virtuales privadas de tipo VPC, subredes dinámicas locales y configurar el juego de reglas de firewall de forma estructurada en Google Cloud Platform.

     ## Uso Típico de Ejemplo

     ```hcl
     module "mi_red_segura" {
       source     = "./modulo-red"
       project_id = "mi-proyecto-gcp"
       
       subnets = [
         {
           subnet_name              = "subred-segura"
           subnet_ip                = "10.0.1.0/24"
           subnet_region            = "us-central1"
           private_ip_google_access = true
         }
       ]
     }
     ```

     {{ .Requirements }}

     {{ .Providers }}

     {{ .Resources }}

     {{ .Inputs }}

     {{ .Outputs }}

     ## Licencia
     Este módulo se distribuye bajo términos de licencia MIT. Consulte con el equipo de DevOps corporativo.

   sections:
     show:
       - requirements
       - providers
       - resources
       - inputs
       - outputs

   sort:
     enabled: true
     by: name

   settings:
     anchor: true
     color: true
     default: true
     description: true
     escape: true
     html: true
     indent: 2
     lockfile: true
     required: true
     sensitive: true
     type: true
   ```

3. Guarda y cierra el archivo `.terraform-docs.yml`.

**Expected output**: El archivo de configuración queda listo para estructurar la salida Markdown de forma estandarizada e interactiva.

**Verification**: Puedes validar que el YAML es correcto con la sintaxis visual de tu editor de código o ejecutando:
```bash
cat .terraform-docs.yml
```

---

### Paso 4: Generación y actualización automatizada de la documentación

**Objective**: Compilar dinámicamente el archivo `README.md` final interpretando las variables y salidas de los archivos HCL usando el motor de `terraform-docs`.

**Instructions**:
1. Ejecuta el comando de generación de documentación utilizando el archivo de configuración `.terraform-docs.yml` que acabas de crear:
   ```bash
   terraform-docs markdown table --config .terraform-docs.yml . > README.md
   ```

2. Revisa el contenido generado en el archivo `README.md` mediante la consola o utilizando un editor de texto:
   ```bash
   cat README.md
   ```

**Expected output**:
El contenido del archivo `README.md` debe estructurarse limpiamente con tablas generadas automáticamente, como se muestra en la siguiente plantilla de previsualización:

```markdown
## Módulo de Red Personalizado de GCP

Este módulo de Terraform permite aprovisionar redes virtuales privadas de tipo VPC, subredes dinámicas locales y configurar el juego de reglas de firewall de forma estructurada en Google Cloud Platform.

## Uso Típico de Ejemplo

```hcl
module "mi_red_segura" {
  source     = "./modulo-red"
  project_id = "mi-proyecto-gcp"
  
  subnets = [
    {
      subnet_name              = "subred-segura"
      subnet_ip                = "10.0.1.0/24"
      subnet_region            = "us-central1"
      private_ip_google_access = true
    }
  ]
}
```

## Requirements

| Name | Version |
|------|---------|
| <a id="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.8.5 |
| <a id="requirement_google"></a> [google](#requirement\_google) | >= 5.25.0 |

## Providers

| Name | Version |
|------|---------|
| <a id="provider_google"></a> [google](#provider\_google) | >= 5.25.0 |

## Resources

| Name | Type |
|------|------|
| [google_compute_firewall.firewall_rules](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_firewall) | resource |
| [google_compute_network.vpc_network](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_network) | resource |
| [google_compute_subnetwork.subnets](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_subnetwork) | resource |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a id="input_firewall_rules"></a> [firewall\_rules](#input\_firewall\_rules) | Conjunto de reglas de firewall personalizadas asociadas a la VPC para controlar el tráfico entrante/saliente. | <pre>list(object({<br>    name      = string<br>    direction = string<br>    priority  = number<br>    ranges    = list(string)<br>    allow = list(object({<br>      protocol = string<br>      ports    = list(string)<br>    }))<br>  }))</pre> | `[]` | no |
| <a id="input_network_name"></a> [network\_name](#input\_network\_name) | El nombre único que se asignará a la VPC (Virtual Private Cloud). | `string` | `"vpc-custom-main"` | no |
| <a id="input_project_id"></a> [project\_id](#input\_project\_id) | El ID del proyecto de Google Cloud Platform (GCP) donde se desplegarán los recursos de red de este módulo. | `string` | n/a | yes |
| <a id="input_routing_mode"></a> [routing\_mode](#input\_routing\_mode) | El modo de enrutamiento de red para la VPC. Los valores válidos son 'REGIONAL' o 'GLOBAL'. | `string` | `"REGIONAL"` | no |
| <a id="input_subnets"></a> [subnets](#input\_subnets) | Lista detallada de subredes personalizadas que se crearán dentro de la VPC. | <pre>list(object({<br>    subnet_name           = string<br>    subnet_ip             = string<br>    subnet_region         = string<br>    private_ip_google_access = bool<br>  }))</pre> | <pre>[<br>  {<br>    "private_ip_google_access": true,<br>    "subnet_ip": "10.10.10.0/24",<br>    "subnet_name": "subnet-dev-us-central1",<br>    "subnet_region": "us-central1"<br>  }<br>]</pre> | no |

## Outputs

| Name | Description |
|------|-------------|
| <a id="output_network_id"></a> [network\_id](#output\_network\_id) | El identificador completamente calificado (URI) de la red VPC creada. |
| <a id="output_network_name"></a> [network\_name](#output\_network\_name) | El nombre de la red VPC creada. |
| <a id="output_subnets_map"></a> [subnets\_map](#output\_subnets_map) | Un mapa asociativo que mapea los nombres de las subredes creadas con sus respectivos identificadores (URIs) en GCP. |

## Licencia
Este módulo se distribuye bajo términos de licencia MIT. Consulte con el equipo de DevOps corporativo.
```

**Verification**: Observa que todos los campos complejos del tipo de datos (como la lista de objetos complejos de `firewall_rules` y `subnets`) fueron convertidos automáticamente a etiquetas de bloque de código `<pre>` HTML para mantener la legibilidad de su estructura.

---

## Validación y Pruebas

Para garantizar la estabilidad y confiabilidad técnica del flujo de documentación implementado, realiza las siguientes actividades de evaluación medible:

### 1. Validación de Presencia de Campos Clave
Ejecuta el siguiente bloque de comandos bash para verificar de manera automatizada que las tablas contienen la información técnica generada correctamente:

```bash
## Verificar la existencia de la documentación de Inputs
grep -q "## Inputs" README.md && echo "Sección Inputs: OK" || echo "Sección Inputs: ERROR"

## Verificar la detección de la variable crítica obligatoria project_id
grep -q "project_id" README.md && echo "Variable project_id detectada: OK" || echo "Variable project_id: ERROR"

## Verificar si el output de red mapeada está expuesto
grep -q "subnets_map" README.md && echo "Output subnets_map detectado: OK" || echo "Output subnets_map: ERROR"
```

**Salida esperada:**
```text
Sección Inputs: OK
Variable project_id detectada: OK
Output subnets_map detectado: OK
```

### 2. Prueba de Robustez contra Inyecciones de Código (Caso Adversario)
Un riesgo potencial es la inyección de código malicioso o descripciones que rompan el formateo visual del `README.md` debido a caracteres especiales o HTML sin escapar.

1. Abre `variables.tf` y agrega una descripción adversaria a una variable nueva de prueba para evaluar la respuesta de escape en `terraform-docs`:

   ```hcl
   # Añadir esto al final de variables.tf para pruebas
   variable "adversarial_test" {
     type        = string
     description = "Prueba de <script>alert('Ataque');</script> y un markdown **negrita** sin cerrar <div style='color:red;'>Inyección."
     default     = "safe"
   }
   ```

2. Ejecuta nuevamente la generación de documentación utilizando la configuración guardada:
   ```bash
   terraform-docs markdown table --config .terraform-docs.yml . > README.md
   ```

3. Busca en el `README.md` el resultado de la variable generada:
   ```bash
   grep -A 2 "adversarial_test" README.md
   ```

**Resultado esperado**: Los caracteres especiales de HTML son transformados a texto plano inocuo o escapados según la regla `settings.escape: true` de tu configuración, de modo que el navegador o el renderizador de Markdown de portales como GitHub o GitLab muestren el texto como comentario seguro en lugar de romper el formato CSS o ejecutar scripts.

4. Limpia la variable de prueba del archivo `variables.tf` y vuelve a generar el README limpio:
   - Abre `variables.tf`, elimina el bloque `variable "adversarial_test"`.
   - Vuelve a ejecutar:
     ```bash
     terraform-docs markdown table --config .terraform-docs.yml . > README.md
     ```

---

## Solución de Problemas

A continuación, se describen dos fallos operativos habituales, sus causas y cómo solucionarlos rápidamente.

### Caso 1: Comando no encontrado (`terraform-docs: command not found`)
- **Síntoma**: Al ejecutar `terraform-docs` se muestra el error `bash: terraform-docs: command not found`.
- **Causa**: El ejecutable binario de la herramienta no fue descargado en una ubicación mapeada en la variable de entorno `$PATH` del usuario actual.
- **Resolución**: 
  Puedes ejecutar la herramienta llamando de forma directa a su ruta absoluta de instalación o bien, agregar temporalmente la ruta correspondiente al contexto actual de ejecución de bash con el siguiente comando:
  ```bash
  export PATH="/home/usuario/.local/bin:$PATH"
  ```
  Para validar que la ruta se agregó de forma correcta, ejecuta `which terraform-docs`. Debería retornar `/home/usuario/.local/bin/terraform-docs`.

### Caso 2: El README.md no se actualiza o tiene formato incorrecto
- **Síntoma**: La salida del README.md no hereda la cabecera personalizada ni la estructura del archivo YAML `.terraform-docs.yml`.
- **Causa**: Se omitió la opción `--config` o se apuntó a un archivo inexistente, lo que provocó que `terraform-docs` utilizara los valores de configuración por defecto.
- **Resolución**: Asegúrate de que el archivo de configuración `.terraform-docs.yml` se encuentre ubicado exactamente en el directorio raíz de ejecución y de pasar el parámetro `--config .terraform-docs.yml` de la siguiente manera:
  ```bash
  terraform-docs markdown table --config /home/usuario/terraform-gcp-labs/lab10-00-02/.terraform-docs.yml /home/usuario/terraform-gcp-labs/lab10-00-02/ > README.md
  ```

---

## Limpieza

Puesto que esta práctica ha consistido exclusivamente en la creación de plantillas de configuración y la generación local de documentación estática mediante análisis de sintaxis HCL, **no se han aprovisionado recursos activos en GCP**, evitando así cargos de facturación.

Para mantener limpio tu espacio de trabajo local, ejecuta las siguientes tareas:

1. Si deseas limpiar el archivo temporal del binario o de configuración antes de archivar tus laboratorios:
   ```bash
   # Asegurar que no hay procesos de terraform corriendo
   # (No se requiere destrucción en la nube dado que no se ejecutó 'terraform apply')
   ```

2. Es recomendable preservar los archivos de código fuente (`main.tf`, `variables.tf`, `outputs.tf`, `.terraform-docs.yml` y `README.md`) para usarlos como base reutilizable de módulos en laboratorios posteriores. Si aun así deseas limpiar completamente el espacio de trabajo local, elimina el directorio:
   ```bash
   rm -rf /home/usuario/terraform-gcp-labs/lab10-00-02
   ```

---

## Resumen

En esta práctica de laboratorio has implementado los principios del **desplazamiento a la izquierda (*shift-left*)** aplicados a la mantenibilidad del código de infraestructura en la nube. 

A través de la configuración de la herramienta `terraform-docs` (versión `v0.18.0`), lograste:
1. **Estandarizar el diseño de módulos**: Desarrollaste variables y salidas sólidas, con tipos y metadatos que actúan como especificación de contrato técnico del módulo de red.
2. **Automatizar el ciclo documental**: Estableciste una plantilla automatizable mediante `.terraform-docs.yml` que convierte el código de infraestructura directamente en un entregable listo para ser publicado en repositorios de control de código de tu organización.
3. **Optimizar la consistencia**: Redujiste la probabilidad de discrepancias de información entre el estado real del código HCL y su documentación visible para otros ingenieros de software, eliminando por completo la documentación técnica manual.

### Recursos Adicionales de Consulta
- [Documentación Oficial de terraform-docs](https://terraform-docs.io/)
- [Prácticas recomendadas para módulos de Terraform (HashiCorp)](https://developer.hashicorp.com/terraform/language/modules/develop)

---

# Estimar costos con infracost

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 25 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar |

## Descripción General

En este laboratorio, aprenderás a integrar **Infracost** dentro de tu flujo de trabajo de Terraform para estimar de manera proactiva los costos mensuales de la infraestructura en Google Cloud Platform (GCP). Implementando la estrategia de *Shift-Left* (desplazamiento a la izquierda), analizarás y compararás financieramente dos escenarios de aprovisionamiento de recursos (Compute Engine y discos persistentes) directamente desde tu terminal local mediante la generación de planes de Terraform en formato JSON.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Configurar y autenticar de manera local la interfaz de línea de comandos (CLI) de Infracost.
- [ ] Exportar planes de ejecución de Terraform en formato JSON compatibles con analizadores estáticos.
- [ ] Comparar de forma automatizada las diferencias de costo mensual en GCP al modificar atributos de infraestructura crítica (tipo de máquina e IOPS/tipo de disco).
- [ ] Generar informes visuales en formato HTML para compartirlos con tomadores de decisiones.

## Prerrequisitos

Para completar este laboratorio con éxito, necesitas:

1. **Conocimientos Teóricos**:
   - Entendimiento del ciclo de vida básico de Terraform (`init`, `plan`, `show`).
   - Familiaridad con los conceptos de instancias de Google Compute Engine y tipos de almacenamiento en bloque persistente (HDD vs. SSD).

2. **Cuentas y Accesos**:
   - Una cuenta activa en [GCP (Google Cloud Platform)](https://cloud.google.com/) con permisos para realizar consultas sobre recursos.
   - Una cuenta registrada (gratuita) en [Infracost](https://www.infracost.io/) para la obtención de una API Key de desarrollo.

## Entorno de Laboratorio

Este laboratorio está diseñado para ejecutarse en el entorno unificado preinstalado. A continuación se especifican los componentes clave:

### Hardware y Sistema Operativo
* **Arquitectura**: CPU x86_64 o ARM64 (2 núcleos mínimos).
* **RAM**: Mínimo 8 GB.
* **Directorio de Trabajo**: `/home/usuario/terraform-gcp-labs/` (Variable constante del entorno).

### Herramientas y Versiones de Software

| Herramienta | Versión Declarada | Arquitectura / Edición | Enlace Oficial de Descarga |
| :--- | :--- | :--- | :--- |
| **Terraform CLI** | `1.8.2` | Linux x86_64 / Community | [Descargar Terraform 1.8.2](https://releases.hashicorp.com/terraform/1.8.2/) |
| **Infracost CLI** | `v0.10.36` | Linux x86_64 / Community | [Descargar Infracost v0.10.36](https://github.com/infracost/infracost/releases/tag/v0.10.36) |
| **Google Cloud Provider**| `5.25.0` | Terraform Registry Plugin | [Provider Registry GCP 5.25.0](https://registry.terraform.io/providers/hashicorp/google/5.25.0) |
| **Google Cloud CLI** | `472.0.0` | Linux x86_64 / SDK | [SDK Release Notes](https://cloud.google.com/sdk/docs/release-notes) |

### Preparación del Directorio de Trabajo

Ejecuta los siguientes comandos en tu terminal para garantizar que el directorio de la práctica exista y esté completamente limpio:

```bash
## Crear el directorio del laboratorio
mkdir -p /home/usuario/terraform-gcp-labs/practica12

## Posicionarse en el directorio de trabajo
cd /home/usuario/terraform-gcp-labs/practica12
```

---

## Instrucciones Paso a Paso

### Paso 1: Autenticación y Configuración de Infracost CLI

Antes de que Infracost pueda realizar consultas a la base de datos de precios de GCP, debes registrar la API Key en tu perfil local de usuario.

1. Verifica la versión de la CLI de Infracost instalada en el sistema:
   ```bash
   infracost --version
   ```
   *Salida esperada:* `Infracost v0.10.36`

2. Autentica tu máquina local con los servidores de Infracost. Este comando abrirá una ventana de navegador web para iniciar sesión o registrarse:
   ```bash
   infracost auth login
   ```
   *Nota operativa:* Si estás utilizando un entorno de terminal sin interfaz gráfica, puedes obtener tu clave directamente desde el [Infracost Dashboard](https://dashboard.infracost.io/) e inicializarla de manera persistente exportando la variable de entorno en tu sesión de shell:
   ```bash
   # Reemplaza 'ico-tu-clave-aqui' con tu clave real obtenida de la consola web de Infracost
   export INFRACOST_API_KEY="ico-tu-clave-aqui"
   ```

3. Verifica que la autenticación sea exitosa intentando realizar una consulta de prueba rápida sobre la configuración:
   ```bash
   infracost configure get api_key
   ```

---

### Paso 2: Declaración de la Infraestructura Base (Escenario A: Configuración de Alto Costo)

Comenzaremos construyendo la infraestructura base que consta de una red VPC, una subred, una máquina virtual de la serie N1 (`n1-standard-1`) y un volumen de disco persistente tipo HDD de gran tamaño.

1. Crea el archivo de definición de variables `variables.tf`:
   ```bash
   cat << 'EOF' > variables.tf
   variable "project_id" {
     type        = string
     description = "El ID del proyecto de Google Cloud de destino"
   }

   variable "region" {
     type        = string
     default     = "us-central1"
     description = "Región estándar para el despliegue de recursos"
   }

   variable "zone" {
     type        = string
     default     = "us-central1-a"
     description = "Zona estándar para el cómputo"
   }
   EOF
   ```

2. Escribe la plantilla de infraestructura de Terraform `main.tf` con los recursos correspondientes al **Escenario A**:
   ```bash
   cat << 'EOF' > main.tf
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

   # Red VPC y Subred base
   resource "google_compute_network" "vpc_network" {
     name                    = "lab12-vpc"
     auto_create_subnetworks = false
   }

   resource "google_compute_subnetwork" "subnet" {
     name          = "lab12-subnet"
     ip_cidr_range = "10.0.1.0/24"
     region        = var.region
     network       = google_compute_network.vpc_network.id
   }

   # Instancia de Cómputo - Escenario A (Costoso)
   resource "google_compute_instance" "vm_instance" {
     name         = "lab12-vm-expensive"
     machine_type = "n1-standard-1"
     zone         = var.zone

     boot_disk {
       initialize_params {
         image = "debian-cloud/debian-11"
         size  = 100
         type  = "pd-standard"
       }
     }

     network_interface {
       network    = google_compute_network.vpc_network.id
       subnetwork = google_compute_subnetwork.subnet.id
     }
   }

   # Disco Persistente Adicional - Escenario A (HDD Sobredimensionado)
   resource "google_compute_disk" "persistent_disk" {
     name  = "lab12-disk-expensive"
     type  = "pd-standard" # HDD Estándar
     size  = 500           # 500 GB
     zone  = var.zone
   }

   resource "google_compute_attached_disk" "attached_disk" {
     disk     = google_compute_disk.persistent_disk.id
     instance = google_compute_instance.vm_instance.id
   }
   EOF
   ```

3. Inicializa el directorio de trabajo para descargar el proveedor oficial de Google Cloud v5.25.0:
   ```bash
   terraform init
   ```

---

### Paso 3: Generación del Plan de Terraform y Análisis Inicial con Infracost

Para evaluar los costos de la infraestructura declarada sin realizar ningún cambio ni despliegue real en GCP, debemos instruir a Terraform para que calcule el plan de ejecución y luego exportarlo en formato JSON legible por Infracost.

1. Asegúrate de que tu variable de entorno del ID del proyecto esté configurada:
   ```bash
   export TF_VAR_project_id="tu-proyecto-gcp" # Sustituye con tu ID real de proyecto GCP
   ```

2. Genera el plan binario de Terraform:
   ```bash
   terraform plan -out=tfplan
   ```

3. Convierte el plan binario en formato JSON estándar:
   ```bash
   terraform show -json tfplan > tfplan.json
   ```

4. Ejecuta un análisis de costos detallado sobre la especificación JSON generada:
   ```bash
   infracost breakdown --path tfplan.json
   ```

5. **Salida Esperada en Consola (Ejemplo Visual)**:
   Infracost identificará los recursos de Compute Engine y de almacenamiento persistente asociados, desplegando una tabla detallada con precios de la API comercial de GCP:

   ```text
   Project: /home/usuario/terraform-gcp-labs/practica12

    Name                                                 Monthly Qty  Unit          Monthly Cost

    google_compute_disk.persistent_disk
    └─ Storage (pd-standard)                                     500  GB                  $20.00

    google_compute_instance.vm_instance
    ├─ Instance usage (n1-standard-1)                            730  hours               $24.27
    └─ Boot storage (pd-standard)                                100  GB                   $4.00

    OVERALL TOTAL                                                                         $48.27
   ```

---

### Paso 4: Refactorización del Código para Optimización de Costos (Escenario B)

Con los datos financieros en mano, es evidente que el disco sobredimensionado y la máquina virtual N1 pueden optimizarse utilizando la serie de propósito general moderna `e2-micro` y reduciendo el almacenamiento a un disco SSD de menor tamaño, garantizando mejores IOPS pero optimizando el gasto total.

1. Edita el archivo `main.tf` para implementar los siguientes cambios de diseño:
   * Cambiar `machine_type` a `"e2-micro"`.
   * Modificar el tipo del disco de arranque a `"pd-balanced"`.
   * Modificar la definición de `google_compute_disk.persistent_disk` para reducir el tamaño a `50` GB y cambiar el tipo a `"pd-ssd"` (Disco de estado sólido).

   Aplica la modificación directamente ejecutando:
   ```bash
   cat << 'EOF' > main.tf
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

   resource "google_compute_network" "vpc_network" {
     name                    = "lab12-vpc"
     auto_create_subnetworks = false
   }

   resource "google_compute_subnetwork" "subnet" {
     name          = "lab12-subnet"
     ip_cidr_range = "10.0.1.0/24"
     region        = var.region
     network       = google_compute_network.vpc_network.id
   }

   # Instancia de Cómputo - Escenario B (Optimizado)
   resource "google_compute_instance" "vm_instance" {
     name         = "lab12-vm-optimized"
     machine_type = "e2-micro"      # Reducido de n1-standard-1
     zone         = var.zone

     boot_disk {
       initialize_params {
         image = "debian-cloud/debian-11"
         size  = 20                # Reducido de 100GB
         type  = "pd-balanced"     # Optimizado a balanceado
       }
     }

     network_interface {
       network    = google_compute_network.vpc_network.id
       subnetwork = google_compute_subnetwork.subnet.id
     }
   }

   # Disco Persistente Adicional - Escenario B (SSD Pequeño)
   resource "google_compute_disk" "persistent_disk" {
     name  = "lab12-disk-optimized"
     type  = "pd-ssd"              # Cambiado de pd-standard a pd-ssd
     size  = 50                    # Reducido de 500GB a 50GB
     zone  = var.zone
   }

   resource "google_compute_attached_disk" "attached_disk" {
     disk     = google_compute_disk.persistent_disk.id
     instance = google_compute_instance.vm_instance.id
   }
   EOF
   ```

---

### Paso 5: Generación del Informe Comparativo e Informe HTML

Ahora generaremos el plan del escenario optimizado y lo compararemos de forma directa con el escenario anterior utilizando comandos nativos de comparación diferencial de Infracost.

1. Genera el nuevo plan de Terraform:
   ```bash
   terraform plan -out=tfplan_opt
   ```

2. Exporta el plan nuevo a formato JSON:
   ```bash
   terraform show -json tfplan_opt > tfplan_opt.json
   ```

3. Compara de forma directa las diferencias de costos mensuales utilizando el operador `--compare-to`:
   ```bash
   infracost diff --path tfplan_opt.json --compare-to tfplan.json
   ```

   *Salida Esperada (Ejemplo)*:
   El reporte mostrará el impacto neto del cambio, destacando la reducción porcentual del costo del despliegue:
   ```text
   ~ google_compute_disk.persistent_disk
     +$8.50 ($0.17/GB -> $0.17/GB)   # Cambio de costo por tipo de disco pd-ssd
     -$18.00 (500 -> 50)             # Reducción por volumen

   ~ google_compute_instance.vm_instance
     -$16.89 ($24.27 -> $7.38)       # Ahorro de cambio a e2-micro

   Monthly cost change: -$26.39 (-54.7%)
   ```

4. Genera un reporte dinámico y visual en formato HTML para que sea almacenable en tus artefactos locales o de CI/CD:
   ```bash
   infracost breakdown --path tfplan_opt.json --format html > report_opt.html
   ```

---

## Validación y Pruebas

Para garantizar que el proceso se haya ejecutado con éxito y de acuerdo a los estándares operativos del laboratorio, realiza las siguientes actividades de validación:

1. **Confirmación de Archivos Generados**:
   Verifica la existencia y el tamaño del reporte interactivo HTML:
   ```bash
   ls -la report_opt.html
   ```

2. **Verificación del Reporte HTML**:
   Comprueba que el archivo HTML contenga el costo mensual aproximado del plan optimizado ejecutando el comando:
   ```bash
   grep -o "Monthly Cost" report_opt.html || grep -o "Total" report_opt.html
   ```
   *Resultado esperado:* Debe dar salida a coincidencias en el archivo de marcado indicando que el informe contiene los contenedores de datos financieros.

3. **Prueba Adversaria (Caso Incompatible / Entrada Inválida)**:
   Infracost está estructurado para fallar tempranamente si el archivo JSON provisto no es una estructura de plan de Terraform válida.
   Ejecuta el comando pasando un archivo vacío o un texto plano malicioso:
   ```bash
   echo "Not a JSON file" > invalid_plan.json
   infracost breakdown --path invalid_plan.json
   ```
   *Resultado de Seguridad Esperado:* Infracost debe retornar un código de salida de error (`exit code > 0`) y mostrar en consola un mensaje indicando que no se ha podido decodificar la especificación del plan de Terraform: `Error: Could not parse Terraform JSON`. Esto garantiza que los flujos automatizados de validación no procesarán información errónea ni ocultarán advertencias críticas.

---

## Solución de Problemas

A continuación se describen los dos incidentes más comunes que pueden surgir durante el desarrollo de esta práctica:

### Problema 1: Error `Invalid API Key` o problemas de consulta remota en Infracost CLI
- **Síntoma**: Al ejecutar `infracost breakdown`, se muestra el error `Error: Your API key is invalid or is not authorized`.
- **Causa**: La clave de API de Infracost se ha copiado de manera errónea, ha expirado, o no está siendo leída correctamente por el shell desde las variables globales.
- **Resolución**:
  1. Limpia y vuelve a configurar la API Key ejecutando:
     ```bash
     infracost configure set api_key "tu_clave_real_de_infracost"
     ```
  2. Verifica el valor registrado actualmente con:
     ```bash
     infracost configure get api_key
     ```

### Problema 2: El reporte de Infracost muestra todos los recursos con costo de `$0.00`
- **Síntoma**: Se genera el reporte en consola u HTML pero el total acumulado de dólares mensuales es `$0.00` a pesar de tener recursos declarados.
- **Causa**: El archivo JSON analizado no contiene recursos estimados o el proveedor configurado de Google Cloud no contiene la variable `project_id` obligatoria, impidiendo que el motor de Terraform renderice los detalles de precios por zona.
- **Resolución**:
  1. Asegúrate de generar el plan de Terraform inyectando explícitamente el ID del proyecto utilizando: `terraform plan -var="project_id=$TF_VAR_project_id" -out=tfplan`.
  2. Confirma que el JSON generado posee definiciones de recursos inspeccionando con:
     ```bash
     grep -i "resource_changes" tfplan.json
     ```

---

## Limpieza

Dado que en esta práctica enfocada al análisis proactivo de costos de infraestructura no hemos ejecutado ningún comando `terraform apply`, no se ha aprovisionado ningún recurso físico en tu proyecto de GCP. Sin embargo, para mantener el espacio de trabajo local ordenado, ejecuta la limpieza de artefactos temporales y binarios generados:

```bash
## Eliminar planes binarios e informes JSON locales
rm -f tfplan tfplan_opt tfplan.json tfplan_opt.json invalid_plan.json

## Mantener el informe HTML si deseas verificarlo externamente o removerlo
rm -f report_opt.html

## Confirmar estado local limpio de Terraform
terraform state list
```
*Resultado esperado:* No se deben listar recursos aprovisionados en el estado local.

---

## Resumen

En este laboratorio has implementado un flujo de trabajo avanzado para la gobernanza financiera de la infraestructura (FinOps) utilizando **Infracost CLI v0.10.36** y **Terraform v1.8.2**. 

### Conceptos Clave Consolidados
1. **Flujo Shift-Left**: La detección y prevención de excesos presupuestarios se realiza sobre el archivo de plan (`tfplan.json`) antes de ejecutar cualquier cambio destructivo o costoso en producción.
2. **Optimización de Recursos**: Se logró migrar una arquitectura con un costo proyectado de `$48.27` a un escenario optimizado de `$21.88` (un ahorro de más del **50%**) modificando el dimensionamiento y las series de disco bajo directivas de análisis proactivo de costos.
3. **Generación de Reportes**: La capacidad de exportar estados de costos en formato HTML interactivo simplifica la comunicación técnica hacia el negocio y consolida el uso de Terraform dentro de pipelines empresariales regulados.

### Recursos Adicionales recomendados
* [Documentación oficial del Proveedor de Google en Infracost](https://www.infracost.io/docs/providers/google/)
* [Prácticas recomendadas para arquitecturas Compute Engine de costo optimizado (GCP)](https://cloud.google.com/compute/docs/tutorials/cost-optimization)

---

# Crear un wrapper básico con Terragrunt

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 30 minutos |
| **Dificultad** | Alta (Hard) |
| **Nivel de Bloom** | Crear |

## Descripción General

En esta práctica de laboratorio, el estudiante refactorizará por completo su espacio de trabajo de Terraform local en una estructura de directorios jerárquica y modularizada utilizando **Terragrunt** (un envoltorio o *wrapper* dinámico para Terraform). El objetivo es implementar los principios DRY (*Don't Repeat Yourself*) abstrayendo y centralizando la configuración del backend remoto de Google Cloud Storage (GCS), la herencia del proveedor (`google`) y el pasaje de variables específicas por entorno para múltiples entornos lógicos independientes (Desarrollo y Producción).

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Diseñar una estructura de directorios jerárquica para múltiples entornos (Dev/Prod) sin duplicar código de infraestructura de Terraform.
- [ ] Configurar un archivo raíz `terragrunt.hcl` para heredar proveedores de GCP y estados remotos de Cloud Storage de manera dinámica con bloqueo de estado (*State Locking*).
- [ ] Desplegar la infraestructura modular de las prácticas previas en entornos aislados de GCP usando comandos nativos de Terragrunt.

## Prerrequisitos

Para completar este laboratorio de forma exitosa, debes disponer de:
1. **Conocimientos previos:** Conceptos sobre el ciclo de vida del estado de Terraform, modularización avanzada y el patrón DRY (*Don't Repeat Yourself*).
2. **Acceso a GCP:** Una cuenta activa de Google Cloud Platform con permisos de Propietario o Editor sobre un proyecto específico.
3. **Variables de entorno configuradas:** Variable `TF_VAR_project_id` definida con el ID del proyecto destino de GCP en la terminal de trabajo.

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Requisito Mínimo | Requisito Recomendado |
| :--- | :--- | :--- |
| **Arquitectura de CPU** | x86_64 o ARM64 (mínimo 2 núcleos) | x86_64 o ARM64 (4 núcleos) |
| **Memoria RAM** | 8 GB | 16 GB |
| **Espacio en Disco** | 10 GB Libres | 20 GB Libres |
| **Ancho de banda** | 10 Mbps de bajada/subida | 100 Mbps sin restricciones de puerto (22, 80, 443) |

### Requisitos de Software

| Herramienta / API | Versión Exacta | Enlace de Descarga / Fuente Oficial | Tipo de Licencia |
| :--- | :--- | :--- | :--- |
| **HashiCorp Terraform** | 1.8.2 | [Descarga Terraform 1.8.2](https://releases.hashicorp.com/terraform/1.8.2/) | BSL 1.1 |
| **Terragrunt CLI** | v0.58.12 | [Descarga Terragrunt v0.58.12](https://github.com/gruntwork-io/terragrunt/releases/tag/v0.58.12) | MIT |
| **Google Cloud SDK** | 472.0.0 (gcloud) | [Descarga Google Cloud SDK v472.0.0](https://cloud.google.com/sdk/docs/release-notes?hl=es-419) | Apache 2.0 |
| **Google Cloud Provider**| 5.25.0 (Terraform) | [Registro del Proveedor Google v5.25.0](https://registry.terraform.io/providers/hashicorp/google/5.25.0) | MPL 2.0 |

> **Nota de Licencia y Herramientas:** Terragrunt se ejecuta bajo licencia MIT. Las herramientas complementarias para análisis de LLM o prompts de IA (como Microsoft 365 Copilot o Copilot Chat) requieren configuraciones con licencias empresariales que respeten la privacidad del código para evitar la fuga de estados sensibles de infraestructura.

### Comandos de Preparación del Entorno

Asegura que tu sesión de terminal apunte al directorio raíz designado para el laboratorio y que las credenciales de GCP estén activas:

```bash
## Definir y exportar la variable de entorno obligatoria
export TF_VAR_project_id="tu-proyecto-id-gcp" # Reemplaza con tu ID de proyecto real

## Crear y posicionarse en el directorio de trabajo del laboratorio
mkdir -p /home/usuario/terraform-gcp-labs/
cd /home/usuario/terraform-gcp-labs/
```

---

## Instrucciones Paso a Paso

### Paso 1: Crear la estructura de directorios del proyecto

**Objetivo:** Crear un esquema de carpetas estandarizado que separe los módulos de Terraform (el código de infraestructura reutilizable) de los entornos operativos manejados por Terragrunt.

1. Ejecuta el siguiente comando para estructurar jerárquicamente el espacio de trabajo:

```bash
mkdir -p /home/usuario/terraform-gcp-labs/modules/gcp_infra
mkdir -p /home/usuario/terraform-gcp-labs/environments/dev
mkdir -p /home/usuario/terraform-gcp-labs/environments/prod
```

2. Verifica que la estructura coincida con el estándar jerárquico esperado por Terragrunt ejecutando:

```bash
tree /home/usuario/terraform-gcp-labs/
```

**Resultado esperado:**
La salida en consola debe estructurarse como se muestra a continuación:
```text
/home/usuario/terraform-gcp-labs/
├── environments
│   ├── dev
│   └── prod
└── modules
    └── gcp_infra
```

**Verificación:** Si el comando `tree` no está instalado en la máquina virtual o estación local, ejecuta `find . -maxdepth 3 -type d` para corroborar el árbol de directorios creado.

---

### Paso 2: Crear el módulo base de infraestructura de Terraform

**Objetivo:** Definir el módulo de infraestructura de Terraform (`gcp_infra`) de forma genérica. Este módulo no contendrá declaraciones de backend ni proveedores hardcodeados, de modo que dependa en su totalidad del wrapper de orquestación.

1. Crea el archivo de declaración de variables del módulo:

```bash
cat <<'EOF' > /home/usuario/terraform-gcp-labs/modules/gcp_infra/variables.tf
variable "project_id" {
  type        = string
  description = "El ID de proyecto asignado en Google Cloud Platform."
}

variable "region" {
  type        = string
  default     = "us-central1"
  description = "La región predeterminada para el despliegue."
}

variable "zone" {
  type        = string
  default     = "us-central1-a"
  description = "La zona predeterminada de cómputo."
}

variable "environment" {
  type        = string
  description = "El nombre lógico del entorno (ej. dev, prod)."
}

variable "machine_type" {
  type        = string
  description = "El tipo de máquina virtual (Compute Engine) de GCP."
}

variable "subnet_cidr" {
  type        = string
  description = "Rango de direcciones CIDR para la subred de este entorno."
}
EOF
```

2. Crea el archivo de definición de recursos del módulo (`main.tf`), declarando una red VPC personalizada, una subred lógica aislada y una instancia de máquina virtual básica:

```bash
cat <<'EOF' > /home/usuario/terraform-gcp-labs/modules/gcp_infra/main.tf
terraform {
  required_version = ">= 1.8.2"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "5.25.0"
    }
  }
}

resource "google_compute_network" "vpc" {
  name                    = "vpc-${var.environment}"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnet" {
  name          = "subnet-${var.environment}"
  ip_cidr_range = var.subnet_cidr
  region        = var.region
  network       = google_compute_network.vpc.id
}

resource "google_compute_instance" "vm" {
  name         = "vm-${var.environment}"
  machine_type = var.machine_type
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
      size  = 10
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.subnet.id
    # Sin dirección IP pública externa para alinearse a las mejores prácticas de seguridad
  }
}
EOF
```

3. Genera los valores de salida del módulo (`outputs.tf`):

```bash
cat <<'EOF' > /home/usuario/terraform-gcp-labs/modules/gcp_infra/outputs.tf
output "network_self_link" {
  value       = google_compute_network.vpc.self_link
  description = "Enlace persistente de la VPC creada en este entorno."
}

output "vm_internal_ip" {
  value       = google_compute_instance.vm.network_interface[0].network_ip
  description = "IP privada de la instancia de máquina virtual creada."
}
EOF
```

**Resultado esperado:**
Los tres archivos de Terraform (`main.tf`, `variables.tf`, y `outputs.tf`) quedan almacenados dentro del directorio `/home/usuario/terraform-gcp-labs/modules/gcp_infra`.

**Verificación:** Ejecuta el comando `terraform validate` en la carpeta del módulo para confirmar que es sintácticamente correcto (requiere ejecutar primero `terraform init` solo a fines de prueba rápida si se desea, aunque no es estrictamente necesario, ya que Terragrunt se encargará de inicializarlo más adelante).

---

### Paso 3: Configurar el archivo raíz `terragrunt.hcl`

**Objetivo:** Crear un archivo de configuración raíz que defina el backend de GCS dinámicamente utilizando variables de entorno globales, evitando la duplicación de código de backend e inyectando de forma automática el bloque del proveedor `google`.

1. Crea el archivo de configuración raíz de Terragrunt:

```bash
cat <<'EOF' > /home/usuario/terraform-gcp-labs/environments/terragrunt.hcl
## Generar dinámicamente la configuración del backend remoto de Google Cloud Storage (GCS)
remote_state {
  backend = "gcs"
  config = {
    bucket   = "tf-state-lock-${get_env("TF_VAR_project_id", "default-proj-id")}"
    prefix   = "${path_relative_to_include()}/terraform.tfstate"
    project  = get_env("TF_VAR_project_id", "default-proj-id")
    location = "us-central1"
  }
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
}

## Generar automáticamente el bloque del proveedor de Google Cloud Platform (GCP)
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "google" {
  project = "${get_env("TF_VAR_project_id", "default-proj-id")}"
  region  = "us-central1"
}
EOF
}
EOF
```

> **Análisis Técnico:** El método `path_relative_to_include()` de Terragrunt le indicará al backend que las rutas del estado remoto en GCS se organicen automáticamente según la carpeta del entorno (por ejemplo, `dev/terraform.tfstate` y `prod/terraform.tfstate`), aislando de forma completa los entornos de trabajo. El método `get_env` lee dinámicamente la variable de entorno del shell local `TF_VAR_project_id`.

**Resultado esperado:** El archivo `terragrunt.hcl` en la carpeta raíz `environments/` queda escrito con las funciones dinámicas listas para ser consumidas por las carpetas secundarias.

**Verificación:** Inspecciona el archivo ejecutando `cat /home/usuario/terraform-gcp-labs/environments/terragrunt.hcl` para certificar la consistencia del contenido.

---

### Paso 4: Configurar los entornos de Desarrollo y Producción

**Objetivo:** Crear los archivos `terragrunt.hcl` de nivel inferior para el entorno de desarrollo (`dev`) y producción (`prod`). Estos heredarán la configuración del backend/proveedor raíz y pasarán parámetros específicos de máquina virtual y direccionamiento IP sin redefinir archivos de Terraform.

1. Crea el archivo Terragrunt para el entorno de **Desarrollo (Dev)** utilizando el tipo de máquina recomendado para mitigar costos (`e2-micro`):

```bash
cat <<'EOF' > /home/usuario/terraform-gcp-labs/environments/dev/terragrunt.hcl
## Incluir configuraciones heredadas del directorio raíz superior
include "root" {
  path = find_in_parent_folders()
}

## Referenciar la fuente del módulo de infraestructura local
terraform {
  source = "../../modules//gcp_infra"
}

## Definir las variables específicas para este entorno (Entradas)
inputs = {
  project_id   = get_env("TF_VAR_project_id", "")
  environment  = "dev"
  machine_type = "e2-micro"
  subnet_cidr  = "10.10.1.0/24"
}
EOF
```

2. Crea el archivo Terragrunt para el entorno de **Producción (Prod)** utilizando el tipo de máquina simulado para producción (`e2-small`):

```bash
cat <<'EOF' > /home/usuario/terraform-gcp-labs/environments/prod/terragrunt.hcl
## Incluir configuraciones heredadas del directorio raíz superior
include "root" {
  path = find_in_parent_folders()
}

## Referenciar la fuente del módulo de infraestructura local
terraform {
  source = "../../modules//gcp_infra"
}

## Definir las variables específicas para este entorno (Entradas)
inputs = {
  project_id   = get_env("TF_VAR_project_id", "")
  environment  = "prod"
  machine_type = "e2-small"
  subnet_cidr  = "10.20.1.0/24"
}
EOF
```

> **Nota sobre la Sintaxis de Origen:** El uso del doble slash (`//`) en la propiedad `source` (ej. `../../modules//gcp_infra`) es una convención de Terragrunt que le indica dónde finaliza el repositorio/directorio principal de módulos y dónde comienza la ruta del submódulo para la copia local del contexto de trabajo de Terraform.

**Resultado esperado:** Se generan de forma limpia los archivos `terragrunt.hcl` en cada una de las subcarpetas de entorno con parámetros aislados y únicos.

**Verificación:** Ejecuta `cat /home/usuario/terraform-gcp-labs/environments/dev/terragrunt.hcl` y `cat /home/usuario/terraform-gcp-labs/environments/prod/terragrunt.hcl` para revisar la parametrización diferenciada.

---

### Paso 5: Inicializar y Desplegar el Entorno de Desarrollo (Dev)

**Objetivo:** Ejecutar Terragrunt para descargar el módulo, generar dinámicamente los archivos `backend.tf` y `provider.tf`, crear de forma automática el bucket de GCS de almacenamiento de estado, e inicializar y aplicar la configuración en el entorno de Desarrollo.

1. Navega hacia el entorno de desarrollo:

```bash
cd /home/usuario/terraform-gcp-labs/environments/dev
```

2. Ejecuta la inicialización de Terragrunt:

```bash
terragrunt init
```

*Nota:* Si el bucket remoto de GCS (ej: `tf-state-lock-tu-proyecto-id-gcp`) no existe aún, Terragrunt detectará de forma automática esta carencia e imprimirá un prompt interactivo preguntando si deseas crearlo. Presiona `y` y luego `Enter` para autorizar a Terragrunt a crear el bucket dinámicamente en us-central1.

3. Genera un plan de ejecución para corroborar los recursos de desarrollo:

```bash
terragrunt plan
```

4. Aplica el despliegue en Google Cloud Platform:

```bash
terragrunt apply --auto-approve
```

**Resultado esperado:**
La salida final debe desplegar 3 recursos (VPC, Subred, e Instancia VM de desarrollo) e imprimir los valores definidos en `outputs.tf`:
```text
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

network_self_link = "https://www.googleapis.com/compute/v1/projects/..."
vm_internal_ip = "10.10.1.2"
```

**Verificación:** Revisa la existencia de la subcarpeta oculta de caché `.terragrunt-cache` en la ruta `/home/usuario/terraform-gcp-labs/environments/dev` y valida que allí se hayan auto-generado los archivos `backend.tf` y `provider.tf` con las configuraciones heredadas de la raíz.

---

### Paso 6: Inicializar y Desplegar el Entorno de Producción (Prod)

**Objetivo:** Desplegar el entorno productivo de manera aislada y simultánea, demostrando cómo Terragrunt hereda de manera transparente el mismo backend común (pero con diferente prefijo de estado) sin interferencias.

1. Navega hacia el entorno de producción:

```bash
cd /home/usuario/terraform-gcp-labs/environments/prod
```

2. Inicializa Terragrunt en el nuevo entorno:

```bash
terragrunt init
```

*Nota:* En esta ocasión, no preguntará si deseas crear el bucket, ya que Terragrunt detectará que el bucket global `tf-state-lock-[PROJECT_ID]` fue creado en el paso anterior. Solo inicializará el nuevo prefijo `prod/terraform.tfstate` de forma dinámica.

3. Aplica los recursos de producción:

```bash
terragrunt apply --auto-approve
```

**Resultado esperado:**
El comando aplicará un despliegue aislado de los 3 recursos de producción con un direccionamiento diferente e imprimirá sus propios outputs:
```text
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

network_self_link = "https://www.googleapis.com/compute/v1/projects/..."
vm_internal_ip = "10.20.1.2"
```

**Verificación:** Utiliza la CLI de Google Cloud para verificar que la instancia `vm-prod` de tamaño `e2-small` se encuentre levantada en tu proyecto de GCP ejecutando: `gcloud compute instances list`.

---

## Validación y Pruebas

Para comprobar que el despliegue con Terragrunt se ha ejecutado de forma correcta bajo estándares DRY de nivel empresarial, realiza las siguientes actividades de control:

### 1. Validación de Estados en Google Cloud Storage
Ejecuta la herramienta de comandos `gcloud` para listar los objetos dentro del bucket de almacenamiento remoto:

```bash
## Listar los archivos del bucket dinámico de GCS
gsutil ls -r gs://tf-state-lock-${TF_VAR_project_id}/
```

**Resultado esperado en consola:**
```text
gs://tf-state-lock-[PROJECT_ID]/dev/:
gs://tf-state-lock-[PROJECT_ID]/dev/terraform.tfstate

gs://tf-state-lock-[PROJECT_ID]/prod/:
gs://tf-state-lock-[PROJECT_ID]/prod/terraform.tfstate
```
Esto certifica que un único bucket centralizado aloja ambos estados con aislamiento total de rutas (prefixes) dinámicas.

### 2. Prueba de Caso Adversario (Simulacro de Error de Inyección)
Un error frecuente en proyectos automatizados es la ausencia de variables de entorno de control en máquinas locales o agentes de CI/CD. 

Ejecuta el siguiente test destructivo de entorno para evaluar la robustez del código de Terragrunt ante un escenario de fallo:

```bash
## Limpiar de manera temporal la variable de entorno obligatoria
unset TF_VAR_project_id

## Intentar ejecutar el plan de Terragrunt en desarrollo
cd /home/usuario/terraform-gcp-labs/environments/dev
terragrunt plan
```

**Comportamiento esperado del sistema ante fallas:**
Terragrunt detendrá inmediatamente la compilación dinámica de los bloques `.tf` arrojando un error de inicialización o plan, ya que la cadena generada intentará resolver el bucket `"tf-state-lock-default-proj-id"`. Si ese proyecto por defecto no existe en tu cuenta de Google Cloud, el comando fallará de forma limpia antes de realizar modificaciones en nubes reales, bloqueando el error humano.

**Cómo mitigar el fallo de caso adversario:**
Restaura tu variable local y todo volverá a funcionar perfectamente:
```bash
export TF_VAR_project_id="tu-proyecto-id-gcp"
```

---

## Solución de Problemas

A continuación, se describen los dos problemas técnicos más comunes que pueden ocurrir durante esta práctica, sus causas subyacentes y sus respectivas soluciones operativas:

### Problema 1: Error "AccessDeniedException" o Fallo al crear el Bucket de Backend Remoto de GCS
* **Síntoma:** Al ejecutar `terragrunt init`, el proceso falla con el error: `AccessDeniedException: 403 Caller does not have storage.buckets.create privilege.` o con un error indicando que el nombre del bucket no se encuentra disponible globalmente.
* **Causa:** El usuario autenticado en la gcloud CLI no tiene suficientes privilegios de IAM de almacenamiento (`Storage Admin` o `Owner`) en el proyecto GCP actual, o el ID del proyecto no es único globalmente y colisiona con el bucket de otro inquilino de Google Cloud.
* **Solución:** 
  1. Ejecuta `gcloud auth application-default login` en tu terminal para garantizar que Terraform y Terragrunt utilicen credenciales autorizadas con permisos completos.
  2. Si la colisión se debe a que el ID del proyecto no es único, actualiza la variable de entorno `TF_VAR_project_id` asignando un ID de proyecto alternativo que poseas en tu consola de Google Cloud Platform y vuelve a intentar el despliegue.

### Problema 2: Error "VPC CIDR block overlap" u Horquillado de IPs
* **Síntoma:** El comando `terragrunt apply` falla en producción indicando un error del API de GCP de tipo `InvalidValue` relacionado con el parámetro `ipCidrRange` del recurso `google_compute_subnetwork`.
* **Causa:** Los rangos CIDR especificados en los archivos `inputs` de Terragrunt se solapan debido a un error de copia de código (por ejemplo, haber configurado `"10.10.1.0/24"` tanto en desarrollo como en producción).
* **Solución:** Abre el archivo `/home/usuario/terraform-gcp-labs/environments/prod/terragrunt.hcl` y valida que la propiedad `subnet_cidr` esté establecida en un segmento de red totalmente diferente e independiente, como `"10.20.1.0/24"`. Tras corregir el archivo, guarda los cambios y ejecuta `terragrunt apply` nuevamente.

---

## Limpieza

Para evitar costos continuos en tu cuenta de Google Cloud Platform de acuerdo a las directrices operativas establecidas, debes desaprovisionar todos los recursos creados durante esta sesión antes de dar por terminado el laboratorio:

1. Desmantela de manera interactiva el entorno de **Producción**:
```bash
cd /home/usuario/terraform-gcp-labs/environments/prod
terragrunt destroy --auto-approve
```

2. Desmantela de manera interactiva el entorno de **Desarrollo**:
```bash
cd /home/usuario/terraform-gcp-labs/environments/dev
terragrunt destroy --auto-approve
```

**Resultado esperado:**
Ambos comandos deben reportar el desmontaje exitoso de todos los recursos aprovisionados:
```text
Destroy complete! Resources: 3 destroyed.
```

3. (Opcional) Elimina de manera manual el bucket de backend remoto de GCS si deseas limpiar tu cuenta de GCP en su totalidad (esta acción eliminará los históricos del estado remoto):
```bash
## CUIDADO: Este paso elimina de forma definitiva los archivos de estado persistentes de este laboratorio
gsutil rm -r gs://tf-state-lock-${TF_VAR_project_id}
```

---

## Resumen

En esta práctica de laboratorio, has diseñado y desplegado exitosamente una arquitectura jerárquica DRY en la nube mediante el uso de **Terragrunt** como wrapper avanzado de orquestación de Terraform:

1. **Principio DRY (Don't Repeat Yourself):** Lograste centralizar la declaración repetitiva de backends de Google Cloud Storage y proveedores en un único archivo raíz `terragrunt.hcl`.
2. **Heredabilidad Dinámica:** Utilizaste variables del entorno del sistema (`get_env`) y rutas dinámicas (`path_relative_to_include`) para segmentar los archivos de estado remoto en directorios aislados de un mismo bucket central de GCS.
3. **Parametrización Específica por Entorno:** Configuraste de forma independiente entornos lógicos de desarrollo y producción inyectando variables directamente a través del bloque `inputs` de Terragrunt, desplegando diferentes tipos de máquinas virtuales y direccionamiento CIDR a partir del mismo código base de Terraform.

### Recursos Adicionales de Consulta
* [Documentación Oficial de Gruntwork Terragrunt](https://terragrunt.gruntwork.io/docs/)
* [Sintaxis de Bloques de Configuración de Terragrunt](https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/)
* [Repositorio del Proveedor de Google Cloud en Terraform](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
