# Validar y desplegar una configuración básica de Terraform en Google Cloud aplicando su flujo esencial.

## Metadatos

| Característica | Detalle |
| :--- | :--- |
| **Duración** | 25 minutos |
| **Complejidad** | Fácil |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio inicial, el estudiante configurará y desplegará una infraestructura monolítica básica en Google Cloud Platform (GCP) utilizando un único directorio de trabajo local. Se construirá una red de nube privada virtual (VPC) personalizada, una subred ubicada en la región `us-central1` y una máquina virtual de cómputo con un tipo de máquina optimizado para costos. A través de este ejercicio práctico, se experimentará el ciclo de vida de desarrollo de infraestructura de extremo a extremo: inicialización, validación estática, planificación de recursos, despliegue real e inspección del archivo de estado local resultante.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Configurar el proveedor oficial de Google Cloud en Terraform especificando versión, proyecto, región y zona predeterminadas.
- [ ] Definir recursos básicos de red (VPC y subred) y cómputo (instancia de Compute Engine) mediante código HCL válido.
- [ ] Ejecutar y analizar de manera crítica el flujo de comandos esencial: `terraform init`, `terraform validate`, `terraform plan`, `terraform apply` y `terraform destroy`.
- [ ] Inspeccionar y comprender la estructura interna de metadatos almacenada dentro del archivo `terraform.tfstate` generado de forma local.

## Prerrequisitos

Para completar este laboratorio con éxito, se requiere:

1. **Conocimientos Previos**:
   - Comprensión básica del modelo declarativo de infraestructura de Terraform frente al enfoque imperativo.
   - Familiaridad con los conceptos esenciales de redes en la nube (VPCs, rangos CIDR) e instancias de máquinas virtuales.
   - Manejo básico de la terminal de comandos Linux.

2. **Acceso a Sistemas y Cuentas**:
   - Una cuenta activa de Google Cloud Platform (GCP) con permisos de Administrador de Compute (o rol de Propietario/Editor) en un proyecto específico.
   - Conexión a Internet de banda ancha (mínimo 10 Mbps) sin bloqueos en los puertos salientes TCP 80, 443 y SSH (22).

3. **Software Requerido e Instalaciones Exactas**:
   - **HashiCorp Terraform v1.8.2** (Edición Linux x86_64). [Enlace de descarga oficial de HashiCorp](https://releases.hashicorp.com/terraform/1.8.2/)
   - **Google Cloud SDK (gcloud CLI) v472.0.0** (Edición Linux x86_64). [Enlace de descarga oficial de Google Cloud](https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-472.0.0-linux-x86_64.tar.gz)
   - **Proveedor de Google Cloud para Terraform v5.25.0**. [Enlace de documentación del proveedor en Terraform Registry](https://registry.terraform.io/providers/hashicorp/google/5.25.0)

## Entorno de Laboratorio

Todas las actividades de este laboratorio se realizarán en una terminal Linux local dentro del directorio raíz estándar del curso.

### Variables de Entorno y Directorio de Trabajo

| Parámetro | Valor Preestablecido |
| :--- | :--- |
| **Directorio de Trabajo** | `/home/usuario/terraform-gcp-labs/lab-01-00-01` |
| **Región de GCP** | `us-central1` |
| **Zona de GCP** | `us-central1-a` |
| **Tipo de Instancia VM** | `e2-micro` |
| **Variable del ID de Proyecto** | `TF_VAR_project_id` |

Ejecute los siguientes comandos en su terminal local para inicializar el entorno y preparar su sesión autenticada con GCP:

```bash
## Crear el directorio del laboratorio 01-00-01 y moverse a él
mkdir -p /home/usuario/terraform-gcp-labs/lab-01-00-01
cd /home/usuario/terraform-gcp-labs/lab-01-00-01

## Autenticar la sesión de gcloud con Application Default Credentials (ADC)
gcloud auth application-default login
```

*(Siga las instrucciones en pantalla en su navegador web para otorgar acceso de API a las credenciales locales de Terraform)*.

Defina la variable de entorno con el identificador de su proyecto de GCP de la siguiente forma (reemplace `TU_ID_DE_PROYECTO_GCP` por su ID real):

```bash
export TF_VAR_project_id="TU_ID_DE_PROYECTO_GCP"
```

## Instrucciones Paso a Paso

### Paso 1: Configurar el entorno de trabajo y la autenticación

**Objetivo**: Asegurar que las variables de entorno de Terraform estén correctamente cargadas en la sesión de terminal activa y verificar la comunicación básica con la API de Google Cloud mediante gcloud CLI.

**Instrucciones**:

1. Verifique que la variable de entorno `TF_VAR_project_id` esté configurada.
2. Ejecute un comando de prueba con `gcloud` para verificar que tiene acceso al proyecto seleccionado.

```bash
## Comprobar que la variable de entorno no esté vacía
echo "Proyecto seleccionado: ${TF_VAR_project_id}"

## Listar de forma resumida la cuenta de servicios activa configurada en el entorno
gcloud config list
```

**Resultado esperado**:
La consola retornará el ID del proyecto asignado a la variable de entorno y listará la cuenta del usuario autenticado que utilizarán las APIs de GCP bajo las credenciales de aplicación por defecto.

**Verificación**:
```bash
if [ -z "$TF_VAR_project_id" ]; then echo "ERROR: La variable TF_VAR_project_id no está configurada."; else echo "OK: Variable lista."; fi
```

---

### Paso 2: Crear el archivo de configuración monolítico (main.tf)

**Objetivo**: Escribir una plantilla de configuración monolítica en HCL que declare los bloques de proveedor necesarios para GCP versión 5.25.0, una red de tipo VPC, una subred personalizada y una máquina virtual de desarrollo.

**Instrucciones**:

1. Dentro del directorio `/home/usuario/terraform-gcp-labs/lab-01-00-01`, cree un archivo llamado `main.tf`.
2. Escriba el bloque de configuración de Terraform definiendo la versión exacta del binario (`1.8.2`) y la versión exacta del proveedor de Google (`5.25.0`).
3. Defina la red VPC con asignación de subredes manual (`auto_create_subnetworks = false`).
4. Agregue una subred en la región `us-central1` con el bloque CIDR `10.10.1.0/24`.
5. Agregue la instancia de VM `e2-micro` con imagen de sistema operativo Debian 11 de arranque.

Escriba el siguiente código en `/home/usuario/terraform-gcp-labs/lab-01-00-01/main.tf`:

```hcl
## ==========================================
## REQUERIMIENTOS DE TERRAFORM Y PROVEEDORES
## ==========================================
terraform {
  required_version = ">= 1.8.2"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "5.25.0"
    }
  }
}

## ==========================================
## CONFIGURACIÓN DEL PROVEEDOR GOOGLE CLOUD
## ==========================================
provider "google" {
  project = var.project_id
  region  = "us-central1"
  zone    = "us-central1-a"
}

## ==========================================
## DECLARACIÓN DE VARIABLES DE ENTRADA
## ==========================================
variable "project_id" {
  type        = string
  description = "El ID del proyecto de Google Cloud donde se desplegarán los recursos."
}

## ==========================================
## RECURSOS DE INFRAESTRUCTURA (RED Y CÓMPUTO)
## ==========================================

## 1. Red de Nube Privada Virtual (VPC)
resource "google_compute_network" "vpc_network" {
  name                    = "lab01-vpc"
  auto_create_subnetworks = false
}

## 2. Subred Personalizada dentro de la VPC
resource "google_compute_subnetwork" "custom_subnet" {
  name          = "lab01-subnet"
  ip_cidr_range = "10.10.1.0/24"
  region        = "us-central1"
  network       = google_compute_network.vpc_network.id
}

## 3. Instancia de Cómputo Básica (Virtual Machine)
resource "google_compute_instance" "vm_instance" {
  name         = "lab01-vm-instancia"
  machine_type = "e2-micro"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.custom_subnet.id

    # Asignar una IP externa pública efímera
    access_config {
      # Dejar vacío para asignación dinámica
    }
  }

  metadata = {
    entorno = "desarrollo"
  }
}

## ==========================================
## VARIABLES DE SALIDA (OUTPUTS)
## ==========================================
output "vm_public_ip" {
  value       = google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip
  description = "Dirección IP pública asignada a la instancia de cómputo."
}
```

**Resultado esperado**:
El archivo `main.tf` se guardará con codificación UTF-8 en la ruta indicada, listo para ser parseado por el compilador interno de HCL de Terraform.

**Verificación**:
```bash
## Validar la existencia física del archivo main.tf en la ruta correcta
ls -la /home/usuario/terraform-gcp-labs/lab-01-00-01/main.tf
```

---

### Paso 3: Inicializar y validar la configuración de Terraform

**Objetivo**: Ejecutar la fase de inicialización de dependencias para descargar el proveedor de GCP especificado y verificar que la sintaxis de HCL escrita no contenga errores semánticos ni estructurales.

**Instrucciones**:

1. Ejecute `terraform init` para que el motor lea el bloque de proveedores y descargue la versión `5.25.0` del proveedor `google`.
2. Ejecute `terraform validate` para confirmar la corrección estructural de los recursos definidos.

```bash
## Inicializar el directorio de trabajo actual
terraform init

## Validar la estructura del código en busca de errores sintácticos
terraform validate
```

**Resultado esperado**:
- El comando `init` creará la carpeta oculta `.terraform` y descargará el plugin del proveedor de Google compatible con la arquitectura de su procesador.
- El comando `validate` retornará un mensaje indicando que la configuración es totalmente válida.

```text
Success! The configuration is valid.
```

**Verificación**:
Verifique que se haya descargado el plugin del proveedor ejecutando:
```bash
ls -la .terraform/providers/registry.terraform.io/hashicorp/google/
```

---

### Paso 4: Generar y analizar el plan de ejecución

**Objetivo**: Generar una vista previa segura de las modificaciones pendientes en Google Cloud sin realizar llamadas de mutación reales en las APIs.

**Instrucciones**:

1. Ejecute el comando `terraform plan` pasando implícitamente la variable de entorno `TF_VAR_project_id`.
2. Analice detalladamente la salida en su terminal para identificar cuántos recursos serán creados, modificados o destruidos.

```bash
terraform plan
```

**Resultado esperado**:
La salida de la terminal debe listar detalladamente la red VPC (`google_compute_network`), la subred (`google_compute_subnetwork`) y la VM (`google_compute_instance`) con un resumen final indicando:

```text
Plan: 3 to add, 0 to change, 0 to destroy.
```

*Nota: No se mostrará ninguna dirección IP pública en el output `vm_public_ip` de forma anticipada porque el valor es dinámico y solo lo asigna GCP durante la creación física de la instancia. Se mostrará en su lugar el indicador `(known after apply)`.*

**Verificación**:
Inspeccione que no existan errores de permisos de lectura de la API de GCP durante el análisis planificado.

---

### Paso 5: Aplicar el plan y desplegar los recursos en GCP

**Objetivo**: Enviar las declaraciones del plan a las APIs de Google Cloud, aprovisionar los recursos físicos reales y recibir la confirmación de despliegue.

**Instrucciones**:

1. Ejecute `terraform apply` para aplicar los cambios de infraestructura.
2. Ingrese el valor afirmativo requerido `yes` cuando la consola le pida interactividad.

```bash
terraform apply
```

*(Espere aproximadamente entre 1 y 2 minutos mientras GCP provisiona la interfaz de red virtual y la instancia de cómputo)*.

**Resultado esperado**:
La ejecución finalizará exitosamente mostrando el ID de la instancia de máquina virtual creada, el rango de red configurado y expondrá el output con la IP pública asignada dinámicamente por Google Cloud.

```text
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:
vm_public_ip = "34.135.XX.XX"
```

**Verificación**:
Use la interfaz de línea de comandos de gcloud para verificar que los recursos se crearon directamente en su consola web del proyecto de Google Cloud:

```bash
gcloud compute instances list --filter="name=lab01-vm-instancia"
```

---

### Paso 6: Inspeccionar el archivo de estado local (terraform.tfstate)

**Objetivo**: Analizar la estructura JSON del archivo que Terraform utiliza como "fuente única de la verdad" local para rastrear el mapeo de recursos declarados frente a la infraestructura física real.

**Instrucciones**:

1. Localice el archivo `terraform.tfstate` generado automáticamente en su directorio raíz local.
2. Utilice herramientas del sistema como `cat` o un visor de JSON estructurado para leer el contenido de metadatos.

```bash
## Comprobar la existencia del archivo de estado local
ls -la terraform.tfstate

## Visualizar el esquema JSON interno del archivo de estado de forma resumida
cat terraform.tfstate | grep -E '"version"|"serial"|"lineage"|"type"|"name"' | head -n 25
```

**Resultado esperado**:
Observará que el archivo es un JSON estándar que contiene información detallada sobre la versión de Terraform utilizada (`1.8.2`), las claves únicas de recursos, ID específicos devueltos por Google Cloud, direcciones IP internas y otros atributos devueltos por la API que no estaban detallados en el archivo original `main.tf`.

**Verificación**:
Asegúrese de que los recursos listados en el `terraform.tfstate` coincidan con el inventario aprovisionado en el paso anterior.

---

## Validación y Pruebas

Para asegurar la calidad técnica de este laboratorio y el cumplimiento exacto de las pautas descritas, realice el siguiente conjunto de pruebas estructuradas:

### Prueba de Consistencia de Estado Local vs API de GCP
Ejecute el comando `terraform refresh` o `terraform plan` para evaluar si el estado local se mantiene en perfecta sincronía con los cambios de la infraestructura. El resultado del plan no debe sugerir ningún cambio adicional de recursos.

```bash
terraform plan
```
*Resultado Esperado de la Validación*: "No changes. Your infrastructure matches the configuration."

### Caso Adversario: Comprobación de Robustez frente a Variables Vacías
Simule un error de configuración común (como la ausencia de autenticación o variables nulas) anulando temporalmente la variable de entorno de configuración de proyecto antes de lanzar un plan de ejecución, observando el manejo robusto de excepciones de Terraform:

```bash
## Guardar temporalmente la variable válida
TEMP_PROJECT_ID=$TF_VAR_project_id

## Anular la variable de entorno
unset TF_VAR_project_id

## Ejecutar el plan para forzar el fallo esperado
terraform plan -input=false
```

*Resultado Esperado de la Validación Adversaria*:
La ejecución de `terraform plan` debe fallar inmediatamente y mostrar de manera limpia un mensaje de error semántico de variable no asignada:
```text
Error: No value for required variable
on main.tf line ...:
var.project_id is required, but no value was given.
```

*Acción de Mitigación*: Restaure la variable limpia original antes de proceder con las fases posteriores del ciclo de vida de laboratorios:
```bash
export TF_VAR_project_id=$TEMP_PROJECT_ID
```

---

## Solución de Problemas

En esta sección se listan los dos problemas comunes de configuración inicial y de infraestructura junto con sus causas y planes de mitigación.

### Problema 1: Error de Acceso Prohibido de la API de Google (403 Forbidden)
- **Síntoma**: Al ejecutar `terraform apply`, el proceso se detiene con el siguiente error en consola:
  ```text
  Error: Error creating Network: googleapi: Error 403: Compute API has not been used in project [...] before or it is disabled.
  ```
- **Causa**: La API de Google Compute Engine (`compute.googleapis.com`) no se encuentra habilitada de forma predeterminada en el proyecto nuevo asignado a la cuenta de GCP.
- **Resolución**: Active la API correspondiente directamente mediante gcloud CLI ejecutando el siguiente comando en su terminal y espere 60 segundos antes de volver a lanzar `terraform apply`:
  ```bash
  gcloud services enable compute.googleapis.com --project=$TF_VAR_project_id
  ```

### Problema 2: Error de Bloqueo de Recursos o Cuota Excedida (Quota Exceeded)
- **Síntoma**: Durante el aprovisionamiento de la máquina virtual, Terraform arroja el mensaje:
  ```text
  Error: Error creating Instance: googleapi: Error 403: Quota 'CPUS' exceeded. Limit: 0.0 in region us-central1.
  ```
- **Causa**: La cuenta o proyecto de GCP utilizada se encuentra bajo una suscripción gratuita muy restrictiva o una directiva de cuota de recursos regional que bloquea la creación de núcleos de cómputo en `us-central1`.
- **Resolución**: Asegúrese de estar utilizando el tipo de máquina especificado en los requerimientos del entorno (`e2-micro`). Si el error de cuota persiste, intente reasignar una región diferente que posea cuotas libres (ej. `us-east1` y `us-east1-b`) modificando manualmente las líneas de región y zona del archivo `main.tf` y reejecutando `terraform apply`.

---

## Limpieza

Para evitar cargos continuos e innecesarios en la nube de Google Cloud, destruiremos la infraestructura construida en esta práctica específica de flujo básico, garantizando que el entorno local se mantenga limpio para futuros despliegues modulares.

1. Ejecute el comando de destrucción de recursos de Terraform:
```bash
terraform destroy -auto-approve
```

2. Verifique la salida final para confirmar la eliminación total de los tres recursos aprovisionados:
```text
Destroy complete! Resources: 3 destroyed.
```

3. Compruebe mediante gcloud CLI que la máquina virtual ya no existe en su proyecto de nube:
```bash
gcloud compute instances list --filter="name=lab01-vm-instancia"
```
*(El comando debe retornar una lista vacía de recursos)*.

*Nota Importante: Conserve el código del archivo `main.tf` intacto dentro del directorio `/home/usuario/terraform-gcp-labs/lab-01-00-01` ya que este servirá como base directa de comparación o refactorización para la siguiente práctica de arquitectura modular.*

---

## Resumen

En esta primera práctica guiada, has completado los siguientes logros técnicos fundamentales:
- **Declarado una estructura HCL formal** especificando de forma explícita las dependencias del proveedor de Google Cloud versión `5.25.0` y la versión de motor de ejecución `1.8.2`.
- **Implementado el flujo de ciclo de vida nativo**: Desde la inicialización local de plugins de proveedor (`init`), validación estructural de la configuración (`validate`), planificación predictiva de infraestructura (`plan`), hasta el despliegue transaccional seguro (`apply`) y la destrucción controlada (`destroy`).
- **Analizado la naturaleza del estado local de Terraform**, determinando la forma en que la herramienta correlaciona los bloques de código JSON dentro del archivo `terraform.tfstate` con las identidades únicas que devuelven las APIs de los recursos de Google Cloud.

### Recursos Adicionales de Estudio
- [Documentación oficial de Terraform: Flujo de Trabajo Core](https://developer.hashicorp.com/terraform/intro/core-workflow)
- [Documentación de Recursos de Red VPC de GCP en Terraform Registry](https://registry.terraform.io/providers/hashicorp/google/5.25.0/docs/resources/compute_network)
- [Guías de Inicio de Google Cloud con Terraform](https://cloud.google.com/docs/terraform)
