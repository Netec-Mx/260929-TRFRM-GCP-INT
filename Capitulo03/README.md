# Migrar terraform.tfstate a Google Cloud Storage con control de bloqueo activado.

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 45 minutos |
| **Dificultad** | Media |
| **Nivel de Bloom** | Aplicar |

## Descripción General

En este laboratorio práctico, migrarás de manera segura el archivo de estado de Terraform (`terraform.tfstate`) desde tu entorno local hacia un almacenamiento remoto gestionado y seguro en **Google Cloud Storage (GCS)**. Utilizando el código modularizado previamente, configurarás el bloque de backend remoto de Terraform, habilitarás el control de versiones en el bucket de GCS para asegurar un historial de cambios, y validarás el mecanismo crítico de control de concurrencia y bloqueo de estado (*State Locking*). Esto garantizará la integridad del estado de la infraestructura en entornos de desarrollo colaborativos.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Crear y configurar un bucket de Google Cloud Storage (GCS) optimizado para el almacenamiento seguro del estado de Terraform.
- [ ] Implementar el bloque de configuración `backend "gcs"` en la arquitectura modular de Terraform.
- [ ] Ejecutar el proceso de migración interactiva del estado local al backend remoto sin pérdida de datos.
- [ ] Validar y auditar el funcionamiento del mecanismo de bloqueo de estado (*State Locking*) en escenarios de ejecución concurrente.

## Prerrequisitos

Para completar este laboratorio con éxito, necesitas contar con:
1. **Código Base Desarrollado:** El directorio `/home/usuario/terraform-gcp-labs/` debe contener la estructura de archivos modularizada del laboratorio anterior (módulos funcionales de red y cómputo que desplieguen al menos una instancia de máquina virtual con tipo `e2-micro`).
2. **Permisos en Google Cloud Platform:** Acceso a un proyecto de GCP activo con el rol de **Administrador de Almacenamiento (Storage Admin)** y **Administrador de Compute Engine (Compute Admin)**.
3. **Autenticación Activa:** Haber configurado las credenciales por defecto de la aplicación en tu entorno local mediante el SDK de Google Cloud.

## Entorno de Laboratorio

El entorno de trabajo debe estar preconfigurado con las siguientes especificaciones técnicas de hardware y software para garantizar la reproducibilidad de la práctica:

### Requisitos de Hardware del Sistema
- **Memoria RAM:** Mínimo 8 GB (16 GB recomendados).
- **CPU:** Arquitectura x86_64 o ARM64 con un mínimo de 2 núcleos de procesamiento.
- **Espacio en Disco:** 10 GB de espacio libre dedicado.
- **Conectividad:** Conexión estable a Internet (mínimo 10 Mbps de subida y bajada) sin restricciones en los puertos 80, 443 y 22.

### Herramientas y Versiones de Software Utilizadas

| Software / Herramienta | Versión Exacta | Arquitectura / Distribución | Enlace de Descarga / Fuente Oficial |
| :--- | :--- | :--- | :--- |
| **HashiCorp Terraform CLI** | `v1.8.2` | Linux / x86_64 | [HashiCorp Releases v1.8.2](https://releases.hashicorp.com/terraform/1.8.2/) |
| **Google Cloud SDK (gcloud CLI)**| `v472.0.0` | Linux / x86_64 | [Google Cloud SDK Archives](https://cloud.google.com/sdk/docs/downloads-versioned-archives) |
| **Google Cloud Provider para TF** | `v5.25.0` | Provider Plugin | [Terraform Registry - Google Provider v5.25.0](https://registry.terraform.io/providers/hashicorp/google/5.25.0) |

### Comandos de Inicialización del Entorno

Antes de iniciar la ejecución de los pasos del laboratorio, abre una terminal en tu estación de trabajo y ejecuta la siguiente secuencia de comandos para definir el directorio de trabajo y autenticar tu sesión de GCP:

```bash
## Crear y acceder al directorio de trabajo estándar
mkdir -p /home/usuario/terraform-gcp-labs/
cd /home/usuario/terraform-gcp-labs/

## Autenticar la CLI de Google Cloud con tus credenciales de usuario
gcloud auth login

## Configurar las credenciales por defecto de la aplicación (ADC) para Terraform
gcloud auth application-default login
```

Asegúrate de exportar la variable de entorno que define el ID de tu proyecto de GCP (reemplaza `TU_PROJECT_ID_AQUI` con tu identificador real de GCP):

```bash
export TF_VAR_project_id="TU_PROJECT_ID_AQUI"
gcloud config set project $TF_VAR_project_id
```

---

## Instrucciones Paso a Paso

### Paso 1: Preparación del Entorno y Verificación del Estado Local

**Objetivo:** Asegurar que el entorno de Terraform tiene un estado local consistente antes de iniciar la migración hacia Google Cloud Storage.

**Instrucciones:**

1. Navega al directorio donde se encuentra tu configuración modular:
   ```bash
   cd /home/usuario/terraform-gcp-labs/
   ```

2. Verifica los archivos de configuración existentes en tu directorio. Deberías tener una estructura similar a la siguiente:
   ```bash
   ls -la
   ```

3. Confirma que tu configuración actual de Terraform es consistente y que los recursos ya están desplegados localmente ejecutando un plan:
   ```bash
   terraform plan -var-file="terraform.tfvars"
   ```
   *Nota: Si aún no has desplegado los recursos en local, ejecuta `terraform apply -var-file="terraform.tfvars" -auto-approve` para simular un estado real inicial que migrar.*

**Resultado Esperado:**
El comando `terraform plan` debe ejecutarse con éxito y mostrar un mensaje indicando que no hay cambios pendientes o detallando los recursos que ya están administrados en tu archivo `terraform.tfstate` local.

```text
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
```

---

### Paso 2: Creación del Bucket de GCS para el Estado Remoto

**Objetivo:** Crear un bucket de Google Cloud Storage que cumpla con los requisitos de nomenclatura globalmente única, control de versiones para recuperación ante desastres y accesibilidad restringida.

**Instrucciones:**

1. Define el nombre del bucket utilizando la estructura requerida para asegurar su unicidad global:
   ```bash
   # Generar el nombre del bucket dinámicamente basado en tu project_id
   export BUCKET_NAME="tf-state-lock-${TF_VAR_project_id}"
   echo "El nombre del bucket será: $BUCKET_NAME"
   ```

2. Crea el bucket de almacenamiento en la región estándar `us-central1` utilizando la herramienta `gcloud CLI`:
   ```bash
   gcloud storage buckets create gs://$BUCKET_NAME \
       --project=$TF_VAR_project_id \
       --location=us-central1 \
       --uniform-bucket-level-access
   ```

3. Habilita el control de versiones en el bucket creado para garantizar que cada modificación en el estado genere un respaldo histórico recuperable:
   ```bash
   gcloud storage buckets update gs://$BUCKET_NAME --versioning
   ```

**Resultado Esperado:**
La terminal debe confirmar la creación exitosa del bucket y la actualización de su configuración con el control de versiones activo.

```text
Creating gs://tf-state-lock-mi-proyecto-gcp/...
   ...
Updating gs://tf-state-lock-mi-proyecto-gcp/...
```

Para verificar que el control de versiones está efectivamente activo, ejecuta:
```bash
gcloud storage buckets describe gs://$BUCKET_NAME --format="yaml(versioning)"
```
Salida esperada:
```yaml
versioning:
  enabled: true
```

---

### Paso 3: Configuración del Bloque de Backend en Terraform

**Objetivo:** Modificar los archivos de configuración para declarar el uso de Google Cloud Storage como el motor de almacenamiento persistente del estado de Terraform.

**Instrucciones:**

1. Crea un nuevo archivo llamado `backend.tf` dentro de `/home/usuario/terraform-gcp-labs/` para aislar la configuración del backend.
   *Importante: Los bloques de configuración de `backend` en Terraform no permiten el uso de interpolación de variables locales o de entorno (como `var.project_id`). Por lo tanto, debes escribir directamente el valor estático del nombre del bucket.*

2. Utiliza tu editor de texto preferido (por ejemplo, `nano`) para crear el archivo:
   ```bash
   nano backend.tf
   ```

3. Pega la siguiente configuración reemplazando el marcador `TU_PROJECT_ID_AQUI` por el valor de tu identificador de proyecto real de Google Cloud:
   ```hcl
   terraform {
     backend "gcs" {
       bucket = "tf-state-lock-TU_PROJECT_ID_AQUI"
       prefix = "terraform/state"
     }
   }
   ```

4. Guarda el archivo (`Ctrl + O` en nano, luego presiona `Enter` para confirmar, y sal con `Ctrl + X`).

**Resultado Esperado:**
El archivo `backend.tf` debe estar guardado en el directorio de trabajo actual. Puedes verificar su contenido ejecutando:
```bash
cat backend.tf
```
La salida debe reflejar exactamente el bloque configurado con el identificador estático de tu proyecto en el campo `bucket`.

---

### Paso 4: Inicialización y Migración Interactiva del Estado

**Objetivo:** Inicializar la configuración del backend de Terraform y transferir el estado de infraestructura existente desde tu máquina local (`terraform.tfstate`) hacia el bucket en GCS sin interrumpir la consistencia de los recursos reales.

**Instrucciones:**

1. Ejecuta el comando de inicialización de Terraform. Al detectar que existe un estado local anterior (`terraform.tfstate`) y una nueva definición de backend, Terraform te solicitará confirmar si deseas migrar el historial existente de forma interactiva:
   ```bash
   terraform init -migrate-state
   ```

2. Lee con atención el mensaje interactivo que se muestra en la terminal:
   ```text
   Do you want to copy existing state to the new backend?
     Pre-existing state was found while migrating the previous "local" backend to the
     new "gcs" backend. An existing state was also found on the "gcs" backend.
     ...
     Do you want to copy this state to the new "gcs" backend?
     Enter a value:
   ```

3. Escribe **`yes`** y presiona `Enter` para autorizar la migración segura de los datos hacia Google Cloud Storage.

**Resultado Esperado:**
Terraform transferirá el estado local y configurará el backend GCS. Deberías ver un mensaje de éxito similar al siguiente:

```text
Successfully configured the backend "gcs"! Terraform will automatically
use this backend from now on. You may now begin working with Terraform.
```

Comprueba que el archivo local `terraform.tfstate` ya no contiene la información del estado activo. Verás que se ha creado un archivo de respaldo vacío o se ha desactivado el rastreo local:
```bash
## El archivo local antiguo se renombrará como respaldo
ls -la terraform.tfstate*
```

---

### Paso 5: Simulación y Verificación de Bloqueo de Estado (State Locking)

**Objetivo:** Comprobar empíricamente que Google Cloud Storage bloquea de forma segura el archivo de estado para evitar la ejecución concurrente de cambios sobre la misma infraestructura.

**Instrucciones:**

1. Abre dos terminales independientes y colócalas en paralelo en tu pantalla. En ambas terminales, navega al mismo directorio de trabajo:
   ```bash
   cd /home/usuario/terraform-gcp-labs/
   ```

2. **Terminal 1:** Inicia una operación interactiva que mantenga el estado bloqueado de forma prolongada sin aplicar ningún cambio real. Para lograr esto, ejecutaremos una instrucción de despliegue (`apply`), la cual adquirirá el bloqueo del estado mientras espera la confirmación manual de aprobación del operador:
   ```bash
   terraform apply -var-file="terraform.tfvars"
   ```
   *No respondas a la pregunta de confirmación "Do you want to perform these actions?" todavía. Mantén esta terminal en espera.*

3. **Terminal 2:** Mientras la Terminal 1 continúa en espera con el bloqueo activo, intenta realizar de forma simultánea una solicitud de planificación en la segunda terminal:
   ```bash
   terraform plan -var-file="terraform.tfvars"
   ```

4. Observa el comportamiento de la Terminal 2.

**Resultado Esperado:**
La Terminal 2 debe fallar de inmediato o denegar la solicitud de planificación, mostrando un error explícito de adquisición de bloqueo de estado (*state lock*). Este es el comportamiento óptimo que previene colisiones operativas en producción.

```text
Acquiring state lock. This may take a few moments...
╷
│ Error: Error acquiring the state lock
│ 
│ Error message: writing "gs://tf-state-lock-mi-proyecto-gcp/terraform/state/default.tflock" failed: 412 Precondition Failed
│ Lock Info:
│   ID:        1714589250123456
│   Path:      gs://tf-state-lock-mi-proyecto-gcp/terraform/state/default.tflock
│   Operation: Apply
│   Who:       usuario@estacion-trabajo
│   Version:   1.8.2
│   Created:   2024-05-01 10:00:00.123456 +0000 UTC
│   Info:      
│ 
│ Terraform acquires a state lock to protect the state from being written
│ by multiple users at the same time. Please resolve the issue above and try
│ again.
╵
```

5. **Terminal 1:** Escribe `no` en la Terminal 1 para cancelar el plan de ejecución de forma segura y liberar el bloqueo.
6. **Terminal 2:** Intenta ejecutar nuevamente `terraform plan -var-file="terraform.tfvars"`. Ahora el comando se ejecutará normalmente, ya que el bloqueo fue liberado al terminar la sesión en la Terminal 1.

---

## Validación y Pruebas

Para asegurar la correcta implementación y robustez de la migración del estado remoto, ejecuta los siguientes pasos de control de calidad:

### 1. Verificación del Estado Remoto mediante gcloud CLI
Para comprobar que el archivo de estado se ha migrado físicamente a la nube, ejecuta el siguiente comando para inspeccionar los objetos dentro de tu bucket:
```bash
gcloud storage objects list gs://tf-state-lock-${TF_VAR_project_id}/ --recursive
```
**Resultado Esperado:** Debe aparecer listado el archivo `default.tfstate` en la ruta correspondiente al prefijo establecido en `backend.tf`:
```text
gs://tf-state-lock-mi-proyecto-gcp/terraform/state/default.tfstate
```

### 2. Validación de Cifrado y Control de Versiones
Para inspeccionar que se guardan diferentes versiones históricas cuando se actualiza la infraestructura (por ejemplo, al crear o eliminar un recurso de red o cómputo menor):
```bash
gcloud storage objects list gs://tf-state-lock-${TF_VAR_project_id}/ --recursive --versions
```
Debe listar múltiples versiones del archivo `default.tfstate` si se han aplicado cambios previos, acompañadas de sus respectivos IDs de versión (`generation`).

### 3. Prueba Adversaria / Caso de Borde (Edge-Case Simulation)
**Escenario de Prueba:** ¿Qué sucede si una ejecución de Terraform se interrumpe de forma abrupta (por ejemplo, corte de energía o pérdida de conexión de red a mitad de un proceso de despliegue) y el estado queda persistentemente bloqueado de forma "huérfana" en GCS?

1. Simula este fallo: Ejecuta `terraform apply -var-file="terraform.tfvars"` en la Terminal 1. Cuando aparezca el prompt de confirmación, simula una interrupción forzando el cierre de la terminal o deteniendo bruscamente el proceso de Terraform con `Ctrl + \` (SIGQUIT) o matando el hilo desde tu terminal de control.
2. Abre una nueva terminal e intenta ejecutar `terraform plan -var-file="terraform.tfvars"`. Observarás que el estado permanece bloqueado con un ID de bloqueo (`ID: <LOCK_ID>`), ya que el proceso original no pudo enviar la señal de liberación de manera limpia.
3. **Acción de Mitigación Segura:** Para resolver esta situación de forma profesional sin destruir el bucket ni el estado:
   - Identifica el ID del bloqueo que se muestra en el mensaje de error (por ejemplo, `1714589250123456`).
   - Ejecuta el comando de desbloqueo manual forzado:
     ```bash
     terraform force-unlock 1714589250123456
     ```
     *Nota: Sustituye `1714589250123456` por el ID numérico real proporcionado en el mensaje de error de bloqueo.*
   - Confirma la acción escribiendo `yes`.
4. Verifica que tras la ejecución de este comando, la infraestructura vuelve a ser operable y los futuros planes se procesan sin errores de exclusión mutua.

---

## Solución de Problemas

### Caso 1: Error al adquirir el bloqueo del estado (412 Precondition Failed)
- **Síntoma:** Al ejecutar cualquier comando de planificación o despliegue (`plan`/`apply`), Terraform se detiene inmediatamente mostrando un mensaje de error con código HTTP `412 Precondition Failed` y haciendo referencia a un archivo `.tflock`.
- **Causa:** Un desarrollador del equipo (o un pipeline automatizado de CI/CD) está ejecutando de forma activa una operación sobre la misma infraestructura, o un proceso anterior de Terraform finalizó de manera abrupta antes de que pudiera eliminar el archivo de bloqueo temporal.
- **Solución:** 
  1. Verifica con el equipo si hay despliegues concurrentes activos.
  2. Si estás seguro de que no hay procesos activos y el bloqueo está "huérfano" debido a un fallo de conexión o caída de la terminal, recupera el **ID de bloqueo** del mensaje de error y ejecuta el comando de desbloqueo:
     ```bash
     terraform force-unlock <ID_DEL_BLOQUEO>
     ```

### Caso 2: Error `Access Denied` durante la migración al backend
- **Síntoma:** Al ejecutar `terraform init -migrate-state`, la CLI falla mostrando un error de denegación de acceso o permisos insuficientes para escribir en el bucket de GCS.
- **Causa:** Las credenciales activas del SDK de Google Cloud (`gcloud auth application-default login`) pertenecen a una cuenta de servicio o de usuario que no tiene asignados los permisos requeridos sobre el bucket de almacenamiento recién creado.
- **Solución:**
  1. Asegúrate de haber iniciado sesión con la cuenta de GCP correcta.
  2. Comprueba que posees el rol `roles/storage.admin` o `roles/storage.objectAdmin` en el bucket o a nivel de proyecto. Puedes otorgarte el permiso mediante `gcloud`:
     ```bash
     gcloud projects add-iam-policy-binding $TF_VAR_project_id \
         --member="user:$(gcloud config get-value account)" \
         --role="roles/storage.admin"
     ```
  3. Ejecuta nuevamente `gcloud auth application-default login` para actualizar tus credenciales locales de Terraform y vuelve a intentar la inicialización.

---

## Limpieza

> **Nota de Continuidad sobre la Infraestructura:** De acuerdo con las pautas de este curso, la infraestructura aprovisionada durante los Laboratorios 1 al 4 debe mantenerse para permitir la continuidad lógica de las prácticas de desarrollo y el análisis de cambios futuros. Sin embargo, si decides suspender la sesión de prácticas o necesitas limpiar los recursos del proveedor para evitar cargos acumulados no deseados:

1. Si deseas destruir la infraestructura gestionada, asegúrate de utilizar el respectivo workspace y de indicarle el archivo de variables:
   ```bash
   terraform destroy -var-file="terraform.tfvars" -auto-approve
   ```

2. Una vez destruida la infraestructura de cómputo y red, puedes proceder a eliminar el bucket de Google Cloud Storage que contiene el estado de forma manual utilizando la CLI (atención: esto eliminará de forma permanente todo el historial de cambios de tu infraestructura):
   ```bash
   # Vaciar y eliminar el bucket de almacenamiento persistente
   gcloud storage rm --recursive gs://tf-state-lock-${TF_VAR_project_id}
   ```

3. Limpia los archivos temporales de configuración locales que apuntaban al backend remoto en caso de requerir un reinicio de laboratorio:
   ```bash
   rm -rf .terraform/
   rm -f .terraform.lock.hcl
   rm -f backend.tf
   ```

---

## Resumen

En este laboratorio práctico has logrado implementar las mejores prácticas operativas y de seguridad recomendadas por HashiCorp y Google Cloud para la administración del estado en entornos reales de producción:

- **Migración del Estado:** Lograste mover de forma exitosa y sin pérdida de datos el archivo `terraform.tfstate` desde un almacenamiento local frágil y poco seguro hacia la nube.
- **Seguridad e Integridad:** Habilitaste el control de versiones en Google Cloud Storage para salvaguardar tu infraestructura frente a corrupciones involuntarias, garantizando que el archivo de estado actúe como una única fuente de verdad fiable.
- **Exclusión Mutua (State Locking):** Validaste cómo el backend de GCS previene errores catastróficos causados por la colisión de ejecuciones concurrentes en entornos colaborativos o de integración continua (CI/CD).

### Recursos de Lectura Recomendados
* [Documentación oficial de Terraform sobre el Backend GCS](https://developer.hashicorp.com/terraform/language/settings/backends/gcs)
* [Prácticas recomendadas de Google Cloud para administrar el estado de Terraform](https://cloud.google.com/docs/terraform/resource-management/store-state)
