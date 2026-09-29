# Crear pipeline que haga plan en pull requests y apply en merges.

## Metadatos

| Dimensión | Detalle |
| :--- | :--- |
| **Duración** | 65 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel Bloom** | Crear (Create) |

## Descripción General

En este laboratorio práctico, automatizarás por completo el ciclo de vida de aprovisionamiento de infraestructura en Google Cloud Platform (GCP) mediante un pipeline de Integración Continua y Despliegue Continuo (CI/CD) con **GitHub Actions**. Tomando como base la estructura modular desarrollada en la Práctica 6, configurarás un repositorio de Git, establecerás un backend remoto seguro en Google Cloud Storage (GCS) con bloqueo de estado (*State Locking*), y programarás un flujo de trabajo declarativo en YAML. 

El pipeline ejecutará análisis sintácticos, validaciones de formato y planes de ejecución (`terraform plan`) cada vez que un desarrollador abra un *Pull Request* (PR) hacia la rama principal (`main`). Únicamente cuando este PR sea aprobado y fusionado (*merged*), el pipeline desencadenará de manera no interactiva la aplicación de los cambios (`terraform apply`), eliminando la necesidad de interactuar manualmente con la infraestructura desde terminales locales.

[VISUAL: 07-01-0001 - Diagrama conceptual del ciclo de vida de una Pull Request con Terraform CI/CD, que ilustra el flujo desde la rama de características (Feature Branch), pasando por la validación sintáctica y de formato, la generación del plan de ejecución expuesto en el log de la pipeline, la aprobación del equipo, el merge a la rama principal y, finalmente, el despliegue automatizado en GCP]

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Diseñar un flujo automatizado de CI/CD utilizando GitHub Actions para la Infraestructura como Código (IaC) de Terraform.
- [ ] Configurar autenticación segura hacia GCP mediante llaves de cuenta de servicio (Service Account Keys) almacenadas en GitHub Secrets.
- [ ] Establecer bifurcaciones lógicas de ejecución: disparar `terraform plan` en Pull Requests y `terraform apply` solo en fusiones (merges) a la rama `main`.
- [ ] Implementar la ejecución no interactiva de Terraform en entornos de automatización utilizando banderas de control de estado y variables de entorno.

## Prerrequisitos

### Requisitos de Conocimiento
- Familiaridad con comandos fundamentales de Git (`git init`, `add`, `commit`, `push`, `checkout`, `merge`).
- Comprensión de los comandos principales de Terraform (`init`, `fmt`, `validate`, `plan`, `apply`, `destroy`).
- Entendimiento básico del formato YAML para definir flujos de trabajo (workflows).

### Requisitos de Acceso y Entorno
- Una cuenta activa en [GitHub](https://github.com/).
- Acceso a una consola web de Google Cloud Platform (GCP) con un proyecto activo y permisos de Propietario (Owner) o Editor sobre el mismo.
- Una estación de trabajo configurada con acceso a internet de banda ancha (mínimo 10 Mbps de bajada/subida).

## Entorno de Laboratorio

Para garantizar la reproducibilidad completa del laboratorio, se definen los siguientes componentes de hardware y versiones exactas de software que deben ser ejecutados en la máquina de trabajo:

### Especificaciones de Hardware (Mínimas y Recomendadas)
- **CPU:** Arquitectura x86_64 o ARM64 (mínimo 2 núcleos, recomendado 4 núcleos).
- **Memoria RAM:** Mínimo 8 GB (Recomendado 16 GB).
- **Almacenamiento:** Al menos 10 GB de espacio libre en disco.
- **Sistema Operativo:** Ubuntu 22.04 LTS o superior, macOS 13+, o Windows 10/11 con WSL2 (Ubuntu 22.04).

### Especificaciones de Software (Versiones Estrictas)

| Software / Proveedor | Versión Exacta | Arquitectura / Distribución | Enlace de Descarga / Licencia |
| :--- | :--- | :--- | :--- |
| **HashiCorp Terraform** | `1.8.2` | Linux x86_64 / macOS / Windows | [HashiCorp Releases](https://releases.hashicorp.com/terraform/1.8.2/) <br> *Licencia: Mozilla Public License v2.0* |
| **Google Cloud CLI (gcloud)** | `472.0.0` | Linux x86_64 / macOS / Windows | [Google Cloud SDK Archives](https://cloud.google.com/sdk/docs/downloads-versioned-archives) <br> *Licencia: Apache License 2.0* |
| **Git CLI** | `2.44.0` | Linux / macOS / Windows | [Git SCM Downloads](https://git-scm.com/downloads) <br> *Licencia: GPL v2.0* |
| **Google Provider para Terraform**| `5.25.0` | N/A (Se descarga vía init) | [Terraform Registry (Google)](https://registry.terraform.io/providers/hashicorp/google/5.25.0) <br> *Licencia: Mozilla Public License v2.0* |

### Constantes del Entorno del Laboratorio
- **Directorio de Trabajo Local:** `/home/usuario/terraform-gcp-labs/` (Si utilizas Windows/macOS, adapta esta ruta a tu directorio de usuario manteniendo la estructura `/terraform-gcp-labs/`).
- **Nomenclatura de Proyecto GCP:** Se utilizará la variable de entorno `TF_VAR_project_id` para inyectar dinámicamente el ID del proyecto de GCP.
- **Región Estándar:** `us-central1`
- **Zona Estándar:** `us-central1-a`
- **Nombre del Bucket de Backend GCS:** `tf-state-lock-[PROJECT_ID]` (Donde `[PROJECT_ID]` debe ser reemplazado por tu ID de proyecto de GCP único globalmente).

---

## Instrucciones Paso a Paso

### Paso 1: Preparación del Código Base y Configuración del Backend Remoto GCS

**Objetivo:** Configurar los archivos de Terraform de la sesión anterior en el directorio local de trabajo, asegurando que el estado de Terraform se almacene remotamente de forma segura en un bucket de Google Cloud Storage con soporte de bloqueo (*State Locking*).

**Instrucciones:**

1. Abre tu terminal de línea de comandos y navega al directorio raíz de trabajo estandarizado:
   ```bash
   mkdir -p /home/usuario/terraform-gcp-labs
   cd /home/usuario/terraform-gcp-labs
   ```

2. Verifica las versiones de las herramientas instaladas localmente para confirmar el cumplimiento de los prerrequisitos:
   ```bash
   terraform version
   gcloud --version
   git --version
   ```

3. Crea el archivo `main.tf` que contendrá la definición de los recursos básicos, el backend de GCS y la declaración del proveedor de Google Cloud. **Nota de diseño:** Para este ejercicio, utilizaremos una máquina `e2-micro` para minimizar costos en GCP.
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
     backend "gcs" {
       bucket = "tf-state-lock-REEMPLAZAR_CON_TU_PROJECT_ID"
       prefix = "terraform/state"
     }
   }

   provider "google" {
     project = var.project_id
     region  = var.region
     zone    = var.zone
   }

   variable "project_id" {
     type        = string
     description = "El ID del proyecto de Google Cloud."
   }

   variable "region" {
     type        = string
     default     = "us-central1"
     description = "Región de GCP estandarizada."
   }

   variable "zone" {
     type        = string
     default     = "us-central1-a"
     description = "Zona de GCP estandarizada."
   }

   resource "google_compute_network" "vpc_network" {
     name                    = "tf-cicd-network"
     auto_create_subnetworks = true
   }

   resource "google_compute_instance" "vm_instance" {
     name         = "tf-cicd-vm"
     machine_type = "e2-micro"
     zone         = var.zone

     boot_disk {
       initialize_params {
         image = "debian-cloud/debian-11"
       }
     }

     network_interface {
       network = google_compute_network.vpc_network.name
       access_config {
         # Ephemeral public IP
       }
     }
   }

   output "instance_ip" {
     value       = google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip
     description = "Dirección IP pública de la instancia creada."
   }
   EOF
   ```

4. Ejecuta un comando sed (o edita manualmente el archivo) para reemplazar el marcador de posición del backend con tu ID de proyecto real de GCP. Asegúrate de que el bucket de almacenamiento de estado ya exista en tu proyecto de GCP (debería haber sido creado siguiendo la nomenclatura `tf-state-lock-[PROJECT_ID]` en prácticas previas; si no existe, puedes crearlo usando la consola de GCP o mediante `gsutil mb gs://tf-state-lock-REEMPLAZAR_CON_TU_PROJECT_ID`).
   ```bash
   # Reemplaza 'mi-proyecto-gcp-id' por tu ID real de GCP
   export MI_PROJECT_ID="mi-proyecto-gcp-id"
   sed -i "s/tf-state-lock-REEMPLAZAR_CON_TU_PROJECT_ID/tf-state-lock-${MI_PROJECT_ID}/g" main.tf
   ```

**Output esperado:**
El archivo `main.tf` creado en `/home/usuario/terraform-gcp-labs/` con la declaración exacta del backend configurado apuntando a tu bucket global único de GCS: `"tf-state-lock-mi-proyecto-gcp-id"`.

**Verificación:**
Inspecciona visualmente el archivo ejecutando:
```bash
grep -A 4 "backend \"gcs\"" main.tf
```
Debes ver el bloque del backend con el nombre correcto de tu bucket y sin marcadores de posición sin resolver.

---

### Paso 2: Creación de la Cuenta de Servicio en GCP y Exportación de Credenciales Seguras

**Objetivo:** Generar una Cuenta de Servicio (Service Account) en GCP con los roles de IAM mínimos necesarios, y extraer una llave criptográfica en formato JSON para la autenticación desatendida del pipeline en GitHub Actions.

**Instrucciones:**

1. Autentícate en tu consola de GCP a través de la terminal local de gcloud CLI (si no lo has hecho ya):
   ```bash
   gcloud auth login
   ```

2. Selecciona tu proyecto activo en la CLI:
   ```bash
   gcloud config set project mi-proyecto-gcp-id
   ```

3. Crea una Cuenta de Servicio dedicada exclusivamente para las operaciones de CI/CD:
   ```bash
   gcloud iam service-accounts create tf-cicd-sa \
       --description="Cuenta de servicio para pipelines de GitHub Actions en Terraform" \
       --display-name="tf-cicd-sa"
   ```

4. Asigna los roles IAM necesarios para que la cuenta de servicio pueda crear instancias de cómputo, administrar redes VPC y escribir en el bucket de estado GCS. Por motivos de simplicidad y control de este laboratorio práctico, le otorgaremos el rol de Editor, limitándolo exclusivamente a este proyecto específico:
   ```bash
   gcloud projects add-iam-policy-binding mi-proyecto-gcp-id \
       --member="serviceAccount:tf-cicd-sa@mi-proyecto-gcp-id.iam.gserviceaccount.com" \
       --role="roles/editor"
   ```

5. Genera y descarga el archivo de llave JSON de manera segura en tu máquina local. **Advertencia de Seguridad:** Nunca guardes este archivo dentro del repositorio de Git ni lo expongas públicamente.
   ```bash
   gcloud iam service-accounts keys create /home/usuario/terraform-gcp-labs/gcp-sa-key.json \
       --iam-account=tf-cicd-sa@mi-proyecto-gcp-id.iam.gserviceaccount.com
   ```

**Output esperado:**
```text
created key [8f3a2b1c0d4e5f...] for [tf-cicd-sa@mi-proyecto-gcp-id.iam.gserviceaccount.com].
```
Un archivo binario/texto estructurado JSON en la ruta `/home/usuario/terraform-gcp-labs/gcp-sa-key.json`.

**Verificación:**
Verifica que el archivo exista localmente y contenga una estructura JSON válida:
```bash
head -n 5 /home/usuario/terraform-gcp-labs/gcp-sa-key.json
```
Debe mostrar los campos de cabecera típicos de una clave privada de Google Cloud Service Account (`"type": "service_account"`, `"project_id"`, etc.).

---

### Paso 3: Inicialización del Repositorio Git y Configuración de Secretos en GitHub

**Objetivo:** Crear un nuevo repositorio en GitHub, inicializar localmente el control de versiones con Git, excluir archivos sensibles mediante `.gitignore` y almacenar de forma segura las credenciales de GCP en los Secretos de GitHub.

**Instrucciones:**

1. Crea un archivo `.gitignore` estandarizado para Terraform y credenciales de GCP para evitar fugas accidentales de secretos en tu repositorio:
   ```bash
   cat << 'EOF' > .gitignore
   # Excluir archivos de configuración locales y caché de proveedores
   .terraform/
   *.tfstate
   *.tfstate.backup
   *.tfvars
   *.tfvars.json

   # Excluir binarios generados por el plan de Terraform
   tfplan
   tfplan.binary

   # Excluir llaves de seguridad locales
   gcp-sa-key.json
   *.json

   # Excluir logs de sistema y variables de entorno
   .env
   crash.log
   EOF
   ```

2. Inicializa el repositorio de Git localmente, añade todos los archivos (excepto los ignorados) y realiza el primer commit:
   ```bash
   git init -b main
   git add .
   git commit -m "chore: inicializar repositorio de infraestructura con backend remoto"
   ```

3. Ve a tu cuenta de GitHub (vía navegador web) y crea un nuevo repositorio llamado `terraform-gcp-cicd-lab`. Asegúrate de crearlo como **Público** o **Privado** y de **no** inicializarlo con un archivo README, `.gitignore` o licencia (deja estas opciones desmarcadas).

4. Vincula tu repositorio local con el repositorio remoto de GitHub. Reemplaza `TU_USUARIO` con tu nombre de usuario real de GitHub:
   ```bash
   git remote add origin https://github.com/TU_USUARIO/terraform-gcp-cicd-lab.git
   ```

5. Configura los secretos del repositorio en la interfaz web de GitHub para que el pipeline automatizado pueda acceder a GCP:
   - Navega en tu navegador a: `https://github.com/TU_USUARIO/terraform-gcp-cicd-lab/settings/secrets/actions`.
   - Haz clic en **New repository secret**.
   - Añade un secreto llamado `GCP_SA_KEY`. En el campo **Value**, pega el contenido completo del archivo local `/home/usuario/terraform-gcp-labs/gcp-sa-key.json`.
   - Añade un segundo secreto llamado `GCP_PROJECT_ID`. En el campo **Value**, ingresa tu ID de proyecto de GCP (`mi-proyecto-gcp-id`).

6. Sube tus cambios iniciales a la rama `main` de GitHub:
   ```bash
   git push -u origin main
   ```

**Output esperado:**
La subida exitosa de los archivos `main.tf` y `.gitignore` al repositorio remoto de GitHub en la rama principal `main`.

**Verificación:**
Actualiza la página de tu repositorio de GitHub en el navegador. Deberías ver los archivos en la rama principal y la sección de "Secrets and variables > Actions" de la configuración de tu repositorio debe mostrar las dos variables declaradas: `GCP_SA_KEY` y `GCP_PROJECT_ID`.

---

### Paso 4: Creación del Workflow de GitHub Actions (`terraform.yml`)

**Objetivo:** Desarrollar e implementar el pipeline declarativo de CI/CD en un archivo YAML bajo el directorio estándar de workflows de GitHub, estructurando las bifurcaciones lógicas de ejecución automatizada.

**Instrucciones:**

1. Crea la estructura de directorios requerida por GitHub Actions para reconocer archivos de workflow en tu repositorio:
   ```bash
   mkdir -p .github/workflows
   ```

2. Crea el archivo de definición del pipeline llamado `terraform.yml`:
   ```bash
   cat << 'EOF' > .github/workflows/terraform.yml
   name: "Terraform CI/CD Pipeline"

   on:
     push:
       branches:
         - main
     pull_request:
       branches:
         - main

   permissions:
     contents: read
     pull-requests: write

   jobs:
     terraform:
       name: "Terraform Execution"
       runs-on: ubuntu-22.04

       env:
         TF_VAR_project_id: ${{ secrets.GCP_PROJECT_ID }}
         GOOGLE_CREDENTIALS: ${{ secrets.GCP_SA_KEY }}

       steps:
         - name: "Checkout Code"
           uses: actions/checkout@v4

         - name: "Setup Terraform"
           uses: hashicorp/setup-terraform@v3
           with:
             terraform_version: "1.8.2"

         - name: "Terraform Format Check"
           id: fmt
           run: terraform fmt -check
           continue-on-error: false

         - name: "Terraform Init"
           id: init
           run: terraform init -input=false

         - name: "Terraform Validate"
           id: validate
           run: terraform validate

         - name: "Terraform Plan"
           id: plan
           if: github.event_name == 'pull_request'
           run: terraform plan -input=false -no-color -out=tfplan
           continue-on-error: false

         - name: "Terraform Apply"
           id: apply
           if: github.ref == 'refs/heads/main' && github.event_name == 'push'
           run: terraform apply -input=false -auto-approve
   EOF
   ```

3. Registra el archivo del workflow en Git, realiza el commit y sube los cambios directamente a `main` para dejar operativo el pipeline base:
   ```bash
   git add .github/workflows/terraform.yml
   git commit -m "feat: añadir workflow de GitHub Actions para Terraform CI/CD"
   git push origin main
   ```

**Output esperado:**
Un archivo YAML de flujo de trabajo válido almacenado en `.github/workflows/terraform.yml` en la rama remota `main`.

**Verificación:**
Navega a la pestaña **Actions** en la página de tu repositorio de GitHub. Deberías ver una ejecución en curso o completada titulada *"Terraform Execution"*. Haz clic en ella y comprueba que se ejecuta en el runner de `ubuntu-22.04` e interactúa con el backend remoto. Como es un push directo a `main`, el paso de `Terraform Plan` se habrá omitido (debido a la condición `if: github.event_name == 'pull_request'`) y se habrá ejecutado exitosamente `Terraform Apply` (aprovisionando la VPC y la máquina virtual en GCP).

---

### Paso 5: Validación del Flujo de Integración Continua (CI) mediante un Pull Request

**Objetivo:** Simular el flujo de trabajo colaborativo de un desarrollador. Crearás una rama de características (*feature branch*), realizarás un cambio en la infraestructura, abrirás un Pull Request y verificarás que el pipeline ejecute solo validaciones y generación de plan de manera automática sin alterar la infraestructura productiva.

**Instrucciones:**

1. Crea y cámbiate a una nueva rama de Git llamada `feature/add-metadata`:
   ```bash
   git checkout -b feature/add-metadata
   ```

2. Modifica la configuración de la máquina virtual de Terraform en el archivo `main.tf` agregando un bloque de metadatos simples para documentar la procedencia del aprovisionamiento:
   ```bash
   # Modificaremos el recurso de la máquina virtual para agregarle metadatos.
   # Abre main.tf y edita el bloque 'google_compute_instance.vm_instance' para incluir el campo 'metadata'
   ```
   Puedes aplicar el cambio ejecutando este bloque en tu terminal para sobreescribir la definición de la instancia en `main.tf` de forma segura:
   ```bash
   cat << 'EOF' > patch.tf
   resource "google_compute_instance" "vm_instance" {
     name         = "tf-cicd-vm"
     machine_type = "e2-micro"
     zone         = var.zone

     boot_disk {
       initialize_params {
         image = "debian-cloud/debian-11"
       }
     }

     network_interface {
       network = google_compute_network.vpc_network.name
       access_config {
         # Ephemeral public IP
       }
     }

     metadata = {
       deployed_by = "github-actions-pipeline"
       environment = "development"
     }
   }
   EOF
   # Reemplazar la definición vieja en main.tf con la nueva parametrizada
   # Usamos un truco simple eliminando el recurso viejo de main.tf y pegando el contenido nuevo
   sed -i '/resource "google_compute_instance" "vm_instance"/,/^}/d' main.tf
   cat patch.tf >> main.tf
   rm patch.tf
   ```

3. Verifica que tu código siga estando correctamente formateado localmente. Si no lo está, la fase de `fmt` fallará en GitHub:
   ```bash
   terraform fmt
   ```

4. Envía los cambios de tu rama al repositorio remoto de GitHub:
   ```bash
   git add main.tf
   git commit -m "feat: agregar metadatos identificadores a la instancia de computo"
   git push origin feature/add-metadata
   ```

5. Ve a la interfaz web de tu repositorio de GitHub y abre un **Pull Request**:
   - Deberías ver una barra amarilla con el botón **Compare & pull request** para la rama recién subida. Haz clic en él.
   - Establece como rama base (`base`) a `main` y como rama de comparación (`compare`) a `feature/add-metadata`.
   - Escribe un título descriptivo y haz clic en **Create pull request**.

**Output esperado:**
La creación del Pull Request desencadenará instantáneamente la ejecución de la tubería de CI en GitHub Actions.

**Verificación:**
- En la página de la Pull Request abierta, observa la sección inferior de estados. Verás que el check *"Terraform Execution"* se está ejecutando.
- Haz clic en **Details** del pipeline en ejecución para ver los logs en tiempo real.
- Confirma que los pasos `Checkout Code`, `Setup Terraform`, `Terraform Format Check`, `Terraform Init` y `Terraform Validate` finalizan con éxito.
- Confirma que el paso `Terraform Plan` se ejecuta y muestra en pantalla los cambios propuestos (la adición de metadatos sin destruir la máquina actual).
- Confirma que el paso `Terraform Apply` se muestra como **Omitido** (*Skipped*). Esto valida el aislamiento estricto de la etapa de CI.

---

### Paso 6: Ejecución del Despliegue Continuo (CD) mediante el Merge a Main

**Objetivo:** Completar el ciclo de vida del flujo de trabajo fusionando los cambios aprobados a la rama productiva, desencadenando la etapa de CD para aplicar las modificaciones directamente en GCP.

**Instrucciones:**

1. Una vez que las validaciones del pipeline en tu Pull Request hayan finalizado exitosamente (todos los checks en verde), regresa a la página de la Pull Request en GitHub.
2. Haz clic en el botón verde **Merge pull request** y posteriormente en **Confirm merge**.
3. Una vez fusionada la rama, regresa a la pestaña **Actions** de tu repositorio.

**Output esperado:**
Se iniciará automáticamente una nueva ejecución del workflow, esta vez con el evento `push` apuntando a la rama `main`.

**Verificación:**
- Haz clic en la ejecución activa del workflow de GitHub Actions asociada al commit de fusión (*Merge commit*).
- Observa que los pasos iniciales se completan.
- Confirma que el paso `Terraform Plan` ahora aparece como **Omitido** (*Skipped*), ya que no estamos en el contexto de una Pull Request.
- Confirma que el paso `Terraform Apply` se ejecuta activamente utilizando la bandera de no interactividad y auto-aprobación (`-auto-approve`).
- Revisa el final de los logs del pipeline para comprobar que el cambio se completó de manera exitosa:
  ```text
  google_compute_instance.vm_instance: Modifying... [id=projects/mi-proyecto-gcp-id/zones/us-central1-a/instances/tf-cicd-vm]
  google_compute_instance.vm_instance: Modifications complete after 8s [id=projects/mi-proyecto-gcp-id/zones/us-central1-a/instances/tf-cicd-vm]
  Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
  ```

---

## Validación y Pruebas

Para garantizar que el pipeline se comporte exactamente de acuerdo con los criterios operativos e identificar fallos funcionales antes de despliegues productivos, debes realizar las siguientes actividades de validación práctica.

### Prueba Automatizada 1: Comprobación de Integridad de la Infraestructura en GCP
Verifica desde tu terminal local de desarrollo que los metadatos agregados mediante el pipeline de CD existan realmente en tu máquina virtual activa de GCP.

* **Comando a ejecutar:**
  ```bash
  gcloud compute instances describe tf-cicd-vm \
      --zone=us-central1-a \
      --format="value(metadata.items)"
  ```
* **Evidencia requerida:** El comando debe retornar la lista de metadatos donde figuren las claves que ingresaste en la rama de características:
  ```text
  [{'key': 'deployed_by', 'value': 'github-actions-pipeline'}, {'key': 'environment', 'value': 'development'}]
  ```

### Prueba Adversaria: Validación ante Errores de Formato o Sintaxis Maliciosa / Errática
Para evaluar la resiliencia de la tubería ante errores del desarrollador antes de realizar una fusión catastrófica en producción, introduce deliberadamente un error de formato y sintaxis en una nueva rama.

1. Regresa a tu terminal local y muévete a la rama `main` para descargar los últimos cambios aplicados:
   ```bash
   git checkout main
   git pull origin main
   ```
2. Crea una rama de prueba de errores:
   ```bash
   git checkout -b test/broken-pipeline
   ```
3. Modifica el archivo `main.tf` de forma que sea sintácticamente incorrecto o no cumpla las directrices de formateo (por ejemplo, remueve llaves de cierre o introduce sangrías desordenadas y una variable inválida):
   ```bash
   # Generamos un error de identación obvio para disparar el formateador
   echo "    # Línea con formato sucio y desalineado" >> main.tf
   # Generamos un error de sintaxis eliminando la última llave de cierre del archivo
   sed -i '$d' main.tf
   ```
4. Sube la rama rota y abre una Pull Request en la interfaz de GitHub:
   ```bash
   git add main.tf
   git commit -m "test: forzar error de compilación en el pipeline"
   git push origin test/broken-pipeline
   ```
5. Abre el Pull Request en GitHub y observa el resultado del pipeline automatizado.

* **Resultado esperado en la interfaz:**
  - El pipeline debe fallar de manera inmediata en la fase de `Terraform Format Check` o `Terraform Validate`.
  - El botón para fusionar el Pull Request debe bloquearse con advertencias visuales que impidan al operador integrar código que comprometa la estabilidad del backend.
  - Esto demuestra de manera medible que el pipeline protege activamente la integridad operacional de la infraestructura en Google Cloud.

---

## Solución de Problemas

A continuación, se describen dos escenarios de falla típicos durante la ejecución de este laboratorio, junto con sus causas y acciones de mitigación detalladas.

### Caso 1: Error en la inicialización de Terraform con código de error "Bucket not found" o "Access denied"
- **Síntoma:** El pipeline falla en el paso `Terraform Init` mostrando el siguiente mensaje de error en los logs de GitHub Actions:
  ```text
  Error: Failed to get existing workspaces: Ba.Error: Failed to list Google Cloud Storage buckets: ... 403 Forbidden
  ```
- **Causa:** La cuenta de servicio `tf-cicd-sa` utilizada por el pipeline de GitHub Actions no cuenta con los permisos necesarios de lectura/escritura en el bucket de GCS declarado para el backend, o el nombre del bucket está mal escrito en el bloque `backend "gcs"` del archivo `main.tf`.
- **Resolución:**
  1. Verifica que el nombre del bucket en `main.tf` coincida exactamente con tu ID de proyecto de GCP y que el recurso exista en la consola de Google Cloud Storage.
  2. Ejecuta en tu terminal el comando de asignación de políticas para asegurar que el rol de Editor esté correctamente asignado a la cuenta de servicio:
     ```bash
     gcloud projects add-iam-policy-binding mi-proyecto-gcp-id \
         --member="serviceAccount:tf-cicd-sa@mi-proyecto-gcp-id.iam.gserviceaccount.com" \
         --role="roles/editor"
     ```

### Caso 2: Error "Terraform Format Check" con código de salida 3 en GitHub Actions
- **Síntoma:** El pipeline falla en la etapa de chequeo de formato (`terraform fmt -check`), impidiendo la generación del plan de ejecución, a pesar de que el código de Terraform es sintácticamente válido.
- **Causa:** El comando `terraform fmt -check` evalúa estrictamente el estilo y la estructura de identación de los archivos de configuración de acuerdo a las convenciones de HashiCorp. Si el desarrollador modificó archivos de texto locales sin aplicar previamente el formateador estandarizado, el workflow detendrá el ciclo de manera intencional.
- **Resolución:**
  1. Ejecuta el formateador de manera local en tu máquina de desarrollo para corregir automáticamente los espaciados, saltos de línea e identación desordenada de tus archivos `.tf`:
     ```bash
     terraform fmt
     ```
  2. Agrega los archivos modificados a la rama de trabajo actual, realiza el commit y sube los cambios para actualizar la ejecución en GitHub Actions:
     ```bash
     git add .
     git commit -m "style: formatear archivos con lineamientos estándar de HashiCorp"
     git push origin <TU_RAMA_DE_CARACTERÍSTICAS>
     ```

---

## Limpieza

Es fundamental liberar los recursos y destruir la infraestructura de desarrollo aprovisionada en Google Cloud Platform una vez finalizadas las pruebas de laboratorio para evitar cargos innecesarios en la facturación del proyecto.

1. Asegúrate de retornar a tu rama de trabajo principal de desarrollo:
   ```bash
   git checkout main
   ```
2. Como buena práctica pedagógica, y para garantizar que las dependencias de estado estén limpias, destruiremos la infraestructura desde la terminal de tu estación de trabajo local. Para ello, debes configurar temporalmente la variable de entorno con la ruta de tu llave local antes de proceder:
   ```bash
   export GOOGLE_CREDENTIALS="/home/usuario/terraform-gcp-labs/gcp-sa-key.json"
   export TF_VAR_project_id="mi-proyecto-gcp-id"
   ```
3. Inicializa Terraform localmente para que descargue el backend y los proveedores en la rama limpia:
   ```bash
   terraform init
   ```
4. Ejecuta el comando de destrucción de recursos de Terraform, asegurándote de validar la salida antes de escribir `yes`:
   ```bash
   terraform destroy
   ```
5. Una vez que Terraform confirme que la máquina virtual y la red VPC han sido completamente destruidas, procede a eliminar el archivo local de la llave de servicio por razones de seguridad:
   ```bash
   rm /home/usuario/terraform-gcp-labs/gcp-sa-key.json
   ```
6. Opcionalmente, puedes eliminar el bucket de almacenamiento de estado GCS (`tf-state-lock-[PROJECT_ID]`) mediante la CLI o la consola web de GCP si no planeas reutilizarlo para laboratorios posteriores.

---

## Resumen

### Logros Clave del Laboratorio
Al completar con éxito este laboratorio, has implementado una solución avanzada de CI/CD para la gestión automatizada de infraestructura de nube utilizando prácticas operativas reales:
- **Automatización Segura:** Diseñaste e implementaste un pipeline robusto de GitHub Actions que controla la evolución del código de Terraform interactuando de forma remota y segura con Google Cloud Platform.
- **Flujo de Trabajo Estilo GitOps:** Configuraste una arquitectura donde las ramas de características protegen el entorno productivo a través de validaciones sintácticas y formateo automatizado (`fmt` y `validate`), limitando la ejecución del plan a los Pull Requests y restringiendo los despliegues de infraestructura real únicamente cuando un cambio es aprobado e integrado a la rama principal `main`.
- **Backend Colaborativo Remoto:** Integiaste el almacenamiento del archivo de estado de Terraform en un bucket de GCS con soporte nativo para *State Locking*, posibilitando la colaboración transparente de múltiples ingenieros sobre un mismo conjunto de recursos en la nube sin riesgos de colisiones o corrupción de datos.

### Recursos de Aprendizaje Adicionales
- [Flujos de Trabajo Automatizados de Terraform con GitHub Actions](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions)
- [Mejores Prácticas de Terraform en Google Cloud](https://cloud.google.com/docs/terraform/best-practices-for-terraform)
- [Guía Oficial de Seguridad de Cuentas de Servicio en GCP](https://cloud.google.com/iam/docs/best-practices-service-accounts)
