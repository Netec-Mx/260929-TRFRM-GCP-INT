# Usar secretos desde Google Cloud Secret Manager y validar que no se expongan.

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 45 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar (Nivel 3) |

## Descripción General

Este laboratorio práctico guía al estudiante en la implementación de una estrategia robusta y segura para la gestión de credenciales e información sensible dentro de Google Cloud Platform (GCP). El estudiante aprovisionará un secreto y su respectiva versión en **Google Cloud Secret Manager** utilizando Terraform CLI. Posteriormente, recuperará dicho valor de forma dinámica a través de un bloque de datos (*data source*), lo inyectará de manera segura en un recurso simulado y aplicará técnicas de aserción avanzada como bloques de validación `precondition` en HCL. Durante todo el proceso, se validarán los mecanismos para prevenir la fuga de información confidencial en los logs de la consola y se analizará el comportamiento de almacenamiento en texto plano dentro del archivo de estado (`.tfstate`).

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:

- [ ] Aprovisionar un secreto y su versión correspondiente en Google Cloud Secret Manager mediante plantillas declarativas de Terraform.
- [ ] Consumir un secreto de forma dinámica usando un origen de datos (*data source*) y aplicarlo en un recurso dependiente.
- [ ] Prevenir la exposición de secretos en la salida estándar de la consola utilizando la directiva `sensitive = true` en variables y salidas (*outputs*).
- [ ] Implementar un bloque `precondition` de validación en tiempo de ejecución para asegurar la estructura de la contraseña antes de desplegar recursos críticos.
- [ ] Analizar el archivo de estado de Terraform (`.tfstate`) para comprender dónde se almacenan físicamente los datos sensibles y cómo protegerlo en entornos operativos reales.

## Prerrequisitos

Para completar este laboratorio de forma exitosa, requieres:

1. **Conocimientos Previos:**
   - Comprensión básica del ciclo de vida de Terraform (`init`, `plan`, `apply`, `destroy`).
   - Familiaridad con el uso de la terminal de comandos de Linux.
   - Entendimiento del concepto de datos sensibles (*sensitive outputs*) en HCL.

2. **Acceso y Autenticación:**
   - Una cuenta activa de Google Cloud Platform con permisos de Propietario (`Owner`) o Administrador de Secret Manager (`roles/secretmanager.admin`).
   - El SDK de Google Cloud (`gcloud` CLI) correctamente autenticado en tu terminal.

## Entorno de Laboratorio

Este laboratorio se realiza dentro de un entorno controlado. Para garantizar la consistencia, se han definido las siguientes variables, herramientas de software y rutas estandarizadas.

### Herramientas y Versiones Oficiales

| Herramienta / Proveedor | Versión Requerida | URL de Descarga / Fuente | Licencia de Uso |
| :--- | :--- | :--- | :--- |
| **Terraform CLI** | v1.8.2 (Linux x86_64) | [HashiCorp Releases v1.8.2](https://releases.hashicorp.com/terraform/1.8.2/) | Business Source License (BSL 1.1) |
| **Google Cloud SDK (gcloud CLI)** | v478.0.0 (Linux x86_64) | [Google Cloud SDK v478.0.0](https://cloud.google.com/sdk/docs/downloads-versioned-archives) | Gratuita (Propiedad de Google) |
| **GCP Provider para Terraform** | v5.25.0 | [Terraform Registry: Google v5.25.0](https://registry.terraform.io/providers/hashicorp/google/5.25.0) | Mozilla Public License 2.0 |
| **Random Provider para Terraform** | v3.6.1 | [Terraform Registry: Random v3.6.1](https://registry.terraform.io/providers/hashicorp/random/3.6.1) | Mozilla Public License 2.0 |

### Variables del Entorno de Trabajo

El directorio de trabajo raíz para la práctica será obligatoriamente `/home/usuario/terraform-gcp-labs/lab08/`. Ejecuta los siguientes comandos para crear y posicionarte en este directorio:

```bash
## Crear la estructura de directorios del laboratorio
mkdir -p /home/usuario/terraform-gcp-labs/lab08/

## Posicionarse en el directorio raíz de la práctica
cd /home/usuario/terraform-gcp-labs/lab08/
```

Configura el ID de tu proyecto de GCP como una variable de entorno para evitar escribirlo de forma estática (*hardcoded*) en el código de configuración de Terraform. Reemplaza `TU_PROYECTO_ID` por tu ID de GCP real:

```bash
export TF_VAR_project_id="TU_PROYECTO_ID"
```

## Instrucciones Paso a Paso

Sigue detenidamente los pasos que se detallan a continuación. Cada sección describe su objetivo, los comandos o código necesarios, los resultados esperados y el método de validación.

---

### Paso 1: Habilitar la API de Secret Manager y configurar el proveedor

**Objetivo:** Activar los servicios necesarios dentro de GCP y definir la configuración estructural inicial de los proveedores de Terraform para garantizar la reproducibilidad.

**Instrucciones:**

1. Ejecuta el comando de la CLI de Google Cloud para habilitar la API de Secret Manager en tu proyecto activo:
   ```bash
   gcloud services enable secretmanager.googleapis.com --project="${TF_VAR_project_id}"
   ```

2. Crea el archivo `providers.tf` para definir los requerimientos estrictos de versión de Terraform y sus proveedores.
   ```bash
   nano providers.tf
   ```

3. Pega el siguiente código HCL en `providers.tf`. Guarda y cierra el editor (`Ctrl+O`, `Enter`, `Ctrl+X`):
   ```hcl
   terraform {
     required_version = "1.8.2"
     required_providers {
       google = {
         source  = "hashicorp/google"
         version = "5.25.0"
       }
       random = {
         source  = "hashicorp/random"
         version = "3.6.1"
       }
     }
   }

   provider "google" {
     project = var.project_id
     region  = "us-central1"
     zone    = "us-central1-a"
   }
   ```

4. Define el archivo de variables base `variables.tf` para soportar el ID del proyecto inyectado por la variable de entorno:
   ```bash
   nano variables.tf
   ```

   Copia y pega la siguiente estructura:
   ```hcl
   variable "project_id" {
     type        = string
     description = "ID del proyecto de Google Cloud (inyectado mediante TF_VAR_project_id)"
   }
   ```

**Resultado esperado:** La consola de comandos debe confirmar la habilitación de la API de Secret Manager sin errores y el directorio debe contener los archivos `providers.tf` y `variables.tf`.

**Verificación:**
Valida que la API de Secret Manager esté activa ejecutando:
```bash
gcloud services list --enabled --filter="name:secretmanager.googleapis.com" --project="${TF_VAR_project_id}"
```
*Deberías ver una salida que muestre `secretmanager.googleapis.com` en la lista.*

---

### Paso 2: Aprovisionar el Secreto y su Versión inicial en HCL

**Objetivo:** Crear un recurso de contraseña aleatoria de manera programática, un contenedor de secreto en GCP y asociar la contraseña como la versión activa número uno.

**Instrucciones:**

1. Crea un nuevo archivo llamado `secrets.tf`:
   ```bash
   nano secrets.tf
   ```

2. Agrega el siguiente código. Este genera una contraseña aleatoria de 24 caracteres utilizando el proveedor `random` (evitando malas prácticas de harcodear contraseñas en código fuente) y luego la provisiona en Secret Manager:
   ```hcl
   # Generar una contraseña aleatoria y compleja de manera segura
   resource "random_password" "db_password" {
     length           = 24
     special          = true
     override_special = "!#$%&*()-_=+[]{}<>:?"
   }

   # Crear el contenedor lógico del secreto en Google Cloud Secret Manager
   resource "google_secret_manager_secret" "app_db_secret" {
     secret_id = "app-db-secret-08"
     labels = {
       entorno = "desarrollo"
       modulo  = "seguridad"
     }

     # Configuración de replicación automática sugerida para GCP
     replication {
       auto {}
     }
   }

   # Definir la versión del secreto cargando el payload con la contraseña generada
   resource "google_secret_manager_secret_version" "app_db_secret_v1" {
     secret      = google_secret_manager_secret.app_db_secret.id
     secret_data = random_password.db_password.result
   }
   ```

3. Guarda los cambios.

**Resultado esperado:** Configuración de Terraform preparada para inicializar, crear una contraseña criptográficamente segura e insertarla automáticamente dentro de Secret Manager.

**Verificación:**
Ejecuta la inicialización de Terraform para descargar los proveedores seleccionados con sus versiones exactas:
```bash
terraform init
```
*Deberías recibir el mensaje de inicialización exitosa:* `Terraform has been successfully initialized!`.

---

### Paso 3: Consumir el secreto mediante un bloque de datos (Data Source) e inyectar validaciones de seguridad

**Objetivo:** Leer el secreto recién creado mediante un origen de datos (`data source`) simulando que es consumido por otra aplicación/recurso, y aplicar un control de seguridad con un bloque `precondition` en HCL.

**Instrucciones:**

1. Crea el archivo de lógica principal de consumo `main.tf`:
   ```bash
   nano main.tf
   ```

2. Añade la configuración que realiza la lectura de la última versión del secreto y simula la creación de una conexión de base de datos con una precondición de seguridad:
   ```hcl
   # Leer la versión más reciente del secreto de forma dinámica
   data "google_secret_manager_secret_version" "db_secret_retrieved" {
     secret     = google_secret_manager_secret.app_db_secret.secret_id
     version    = "latest"
     depends_on = [google_secret_manager_secret_version.app_db_secret_v1]
   }

   # Recurso ficticio o simulador de despliegue para validar el secreto
   resource "terraform_data" "db_connection_simulator" {
     input = {
       host     = "10.128.0.5"
       username = "db_admin_user"
       password = data.google_secret_manager_secret_version.db_secret_retrieved.secret_data
     }

     # Implementación de aserción de seguridad en tiempo de plan/apply
     lifecycle {
       precondition {
         # Validar que la contraseña extraída no sea vacía y tenga una longitud mínima segura
         condition     = length(data.google_secret_manager_secret_version.db_secret_retrieved.secret_data) >= 16
         error_message = "ERROR DE SEGURIDAD: La contraseña recuperada de Secret Manager es demasiado corta (debe tener al menos 16 caracteres)."
       }
     }
   }
   ```

3. Guarda los cambios de `main.tf`.

**Resultado esperado:** La lógica del origen de datos leerá el valor de forma dinámica. El bloque `terraform_data` simulará el despliegue del componente y la `precondition` forzará una restricción estricta de longitud.

**Verificación:**
Ejecuta un plan de validación inicial para constatar que el diseño del flujo sintáctico es correcto:
```bash
terraform plan
```
*El plan de ejecución debe procesarse correctamente indicando que se agregarán 4 recursos (la contraseña aleatoria, el contenedor del secreto, la versión del secreto y el simulador de conexión).*

---

### Paso 4: Declarar salidas sensibles y ejecutar el despliegue

**Objetivo:** Definir salidas (*outputs*) que utilicen la directiva `sensitive = true` para evitar que el valor de la contraseña se filtre en la salida estándar de la consola o en pipelines de Integración Continua (CI/CD). Ejecutar el despliegue de la infraestructura.

**Instrucciones:**

1. Crea el archivo de salidas `outputs.tf`:
   ```bash
   nano outputs.tf
   ```

2. Introduce las siguientes salidas, una para metadatos públicos y otra marcada expresamente como sensible:
   ```hcl
   output "secret_version_name" {
     value       = data.google_secret_manager_secret_version.db_secret_retrieved.name
     description = "Ruta completa de la versión del secreto en GCP."
   }

   output "simulated_db_password" {
     value       = data.google_secret_manager_secret_version.db_secret_retrieved.secret_data
     description = "Contraseña de base de datos recuperada de Secret Manager."
     sensitive   = true
   }
   ```

3. Guarda los cambios.

4. Ejecuta el despliegue real de la configuración usando la confirmación automática:
   ```bash
   terraform apply -auto-approve
   ```

**Resultado esperado:** Terraform creará todos los componentes y los desplegará exitosamente en GCP. Al finalizar el proceso, la terminal mostrará un resumen de la ejecución.

**Verificación:**
Inspecciona visualmente la sección de `Outputs` en tu terminal. Debes observar un formato idéntico al siguiente:
```text
Outputs:

secret_version_name = "projects/1234567890/secrets/app-db-secret-08/versions/1"
simulated_db_password = <sensitive>
```
*Confirma que `simulated_db_password` tiene el valor enmascarado como `<sensitive>`. Esto garantiza que la salida de consola en el flujo de trabajo local o servidor de automatización (CI/CD) no filtre la contraseña.*

---

### Paso 5: Auditar la seguridad del archivo de estado local

**Objetivo:** Analizar el comportamiento del estado de Terraform frente a la manipulación de información secreta para comprender sus riesgos.

**Instrucciones:**

1. Abre el archivo de estado de Terraform (`terraform.tfstate`) que se generó de manera local en tu directorio de trabajo:
   ```bash
   cat terraform.tfstate | grep -i -A 3 "secret_data"
   ```

2. Observa la salida que devuelve el comando.

**Resultado esperado:** El valor de la contraseña se mostrará visible en texto plano en la estructura JSON del estado de Terraform.

```json
"secret_data": "f-P8$U8=7bL?_3!z+Xy1@8qK",
```

**Verificación:**
Reflexiona sobre este hallazgo de seguridad: **La directiva `sensitive = true` solo evita que la contraseña se imprima en las salidas visuales de la terminal, pero no la cifra ni la oculta dentro del archivo de estado**. Por lo tanto, en producción es obligatorio utilizar backends remotos y seguros (como Google Cloud Storage con controles estrictos de IAM y cifrado en reposo) siguiendo el estándar de nombrado `tf-state-lock-[PROJECT_ID]`.

---

## Validación y Pruebas

Para garantizar que el laboratorio funciona correctamente bajo condiciones operativas y evaluar la efectividad de las defensas implementadas, realiza los siguientes controles de calidad y validación adversarial.

### Prueba 1: Recuperar el secreto mediante la CLI de Google Cloud (Validación de Integridad en GCP)

Para certificar que el valor guardado por Terraform en Secret Manager coincide con la contraseña que se generó aleatoriamente, utiliza el comando `gcloud` en tu terminal:

```bash
## Obtener el contenido del secreto directamente de la API de Google Cloud
gcloud secrets versions access 1 --secret="app-db-secret-08" --project="${TF_VAR_project_id}"
```

**Resultado esperado:** La terminal imprimirá en pantalla la misma cadena de caracteres de la contraseña.

### Prueba 2: Caso Adversario — Intentar forzar una contraseña que falle la validación `precondition`

Las validaciones en tiempo de ejecución (`preconditions`) protegen el despliegue de configuraciones que no cumplen con los requisitos mínimos de seguridad. Forzaremos un escenario donde se intente inyectar una contraseña insegura y validaremos que Terraform detenga el despliegue de inmediato.

1. Abre temporalmente el archivo `secrets.tf`:
   ```bash
   nano secrets.tf
   ```

2. Modifica temporalmente la longitud de la contraseña generada por `random_password`, reduciéndola de `24` a un valor inseguro como `8` caracteres:
   ```hcl
   resource "random_password" "db_password" {
     length           = 8  # Modificación forzada para inducir al fallo de seguridad
     special          = true
     override_special = "!#$%&*()-_=+[]{}<>:?"
   }
   ```

3. Guarda los cambios en `secrets.tf`.

4. Intenta ejecutar un plan con la nueva contraseña insegura:
   ```bash
   terraform plan
   ```

**Resultado esperado:** Terraform generará un error crítico durante la fase de planeación y bloqueará el proceso de actualización del simulador de base de datos debido al incumplimiento de la precondición definida en `main.tf`. La salida debe ser similar a:

```text
╷
│ Error: Resource precondition failed
│ 
│   on main.tf line 15, in resource "terraform_data" "db_connection_simulator":
│   15:       precondition {
│     ├────────────────
│     │ data.google_secret_manager_secret_version.db_secret_retrieved.secret_data is "a!B9#c-D"
│ 
│ ERROR DE SEGURIDAD: La contraseña recuperada de Secret Manager es demasiado corta (debe tener al menos 16 caracteres).
╵
```

Esto demuestra que los bloques `precondition` garantizan la seguridad de la infraestructura incluso si se cambia el origen del secreto (por ejemplo, si un administrador de seguridad cambia manualmente el secreto en GCP a una clave débil de forma descuidada).

5. **Restablece tu configuración original:** Abre de nuevo `secrets.tf` y cambia el valor de longitud de la contraseña de vuelta a `24` para restaurar la integridad del estado.
   ```bash
   nano secrets.tf
   ```
   *Restaurar `length = 24`.*

6. Ejecuta `terraform plan` para confirmar que el entorno vuelve a ser estable.

---

## Solución de Problemas

A continuación, se listan dos de las situaciones problemáticas más habituales asociadas a la gestión de secretos con Terraform, sus causas fundamentales y la manera técnica de solventarlas.

### Problema 1: Error de permisos `PermissionDenied` al intentar acceder a la API de Secret Manager

* **Síntomas:** Al ejecutar `terraform apply`, la terminal devuelve un error similar a:
  ```text
  Error: Error creating Secret: googleapi: Error 403: Permission 'secretmanager.secrets.create' denied on resource...
  ```
* **Causa:** La cuenta de servicio de GCP o el usuario con el que se está ejecutando la sesión de Terraform CLI no cuenta con el rol adecuado de control de accesos (IAM) para manipular Secret Manager en el proyecto asignado.
* **Solución:** Otorga el rol necesario a tu identidad. Puedes realizarlo rápidamente con la CLI de `gcloud` asignando el rol de Administrador de Secret Manager (`roles/secretmanager.admin`):
  ```bash
  # Obtener el correo del usuario activo en gcloud CLI
  ACTIVE_USER=$(gcloud config get-value account)

  # Asignar el rol requerido
  gcloud projects add-iam-policy-binding ${TF_VAR_project_id} \
    --member="user:${ACTIVE_USER}" \
    --role="roles/secretmanager.admin"
  ```
  Una vez asignado, vuelve a ejecutar `terraform apply`.

### Problema 2: Error de inconsistencia `DependencyCycle` o error de lectura en tiempo de plan por versión inexistente

* **Síntomas:** Al modificar la estructura, Terraform reporta que el data source `data.google_secret_manager_secret_version.db_secret_retrieved` no puede resolver la versión `"latest"`.
* **Causa:** Este problema ocurre porque el bloque `data` intenta consultar la versión del secreto en la API de GCP *antes* de que la versión haya sido físicamente creada por el bloque `google_secret_manager_secret_version.app_db_secret_v1` en el mismo ciclo. Terraform asume erróneamente que puede resolver la consulta en paralelo.
* **Solución:** Forzar el orden secuencial estricto mediante la adición de la directiva `depends_on` dentro de la definición del origen de datos `data` en tu archivo `main.tf`:
  ```hcl
  data "google_secret_manager_secret_version" "db_secret_retrieved" {
    secret     = google_secret_manager_secret.app_db_secret.secret_id
    version    = "latest"
    # Esta línea obliga a Terraform a crear primero el valor en la API antes de consultarlo
    depends_on = [google_secret_manager_secret_version.app_db_secret_v1]
  }
  ```

---

## Limpieza

Para evitar el consumo no deseado de créditos del proyecto de GCP y mantener la higiene de tu entorno de trabajo una vez concluida la sesión práctica, realiza los siguientes pasos:

1. Ejecuta la destrucción de todos los recursos generados durante la práctica utilizando el flag autoconfirmante:
   ```bash
   terraform destroy -auto-approve
   ```

2. Verifica en la salida estándar de tu terminal que los recursos hayan sido efectivamente borrados. Debes ver una confirmación final similar a esta:
   ```text
   Destroy complete! Resources: 4 destroyed.
   ```

3. (Opcional) Elimina el archivo de estado de Terraform y los subdirectorios temporales de inicialización locales creados en la carpeta de la práctica actual:
   ```bash
   rm -rf .terraform/ .terraform.lock.hcl terraform.tfstate terraform.tfstate.backup
   ```

---

## Resumen

En este laboratorio has completado con éxito el flujo avanzado de configuración para la protección y aprovisionamiento seguro de secretos e información confidencial:

* **Centralización de Secretos:** Implementaste el aprovisionamiento nativo de contenedores y versiones del servicio **Google Cloud Secret Manager** utilizando el proveedor oficial de Google (`google` v5.25.0).
* **Ofuscación de Salidas:** Comprendiste el uso de la directiva `sensitive = true` en outputs para resguardar la confidencialidad de datos sensibles y evitar la fuga involuntaria de credenciales en la interfaz o servidores de CI/CD.
* **Validación de Datos con Preconditions:** Diseñaste aserciones personalizadas (`precondition` en HCL) que evitan fallos silenciosos e impiden despliegues con políticas de contraseñas débiles.
* **Análisis de Estado:** Comprendiste que el estado de Terraform (`.tfstate`) almacena la información sensible en texto plano y, por ende, aprendiste por qué es indispensable proteger tus archivos de estado con backends remotos y cifrados.

### Recursos adicionales para profundizar:
- [Documentación oficial del Recurso de Secretos en Terraform](https://registry.terraform.io/providers/hashicorp/google/5.25.0/docs/resources/secret_manager_secret)
- [Guía de Google Cloud sobre Secret Manager](https://cloud.google.com/secret-manager/docs)
- [Buenas prácticas de HashiCorp para gestionar datos sensibles](https://developer.hashicorp.com/terraform/tutorials/configuration-language/sensitive-variables)
