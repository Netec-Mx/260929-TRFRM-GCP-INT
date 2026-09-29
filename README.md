<p align="center">
  <img src="https://raw.githubusercontent.com/Netec-Mx/260929-TRFRM-GCP-INT/main/assets/LogoNetec.png" alt="NETEC" width="180" />
</p>

# Terraform con GCP Intermediate

Este curso intermedio de Terraform está diseñado para usuarios que ya conocen los fundamentos y desean dominar técnicas de reutilización de código, separación de entornos, gestión segura del estado, automatización de despliegues y el uso de herramientas que mejoran la calidad, seguridad y trazabilidad de la infraestructura como código (IaC). A través de prácticas guiadas, se profundiza en la aplicación de buenas prácticas, integración con sistemas externos y preparación para entornos reales colaborativos.

## Accesos rápidos

- [**Setup Guide del curso**](https://github.com/Netec-Mx/260929-TRFRM-GCP-INT/blob/main/SETUP_GUIDE.md)
- [Laboratorios por capítulo](#lista-de-laboratorios)

## Estructura

- `SETUP_GUIDE.md`: guía de instalación y preparación del entorno.
- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [Validar y desplegar una configuración básica de Terraform en Google Cloud aplicando su flujo esencial.](Capitulo01/README.md#validar-y-desplegar-una-configuración-básica-de-terraform-en-google-cloud-aplicando-su-flujo-esencial)
  - Descripción: Validar y desplegar una configuración básica de Terraform en Google Cloud aplicando su flujo esencial.
  - Duración estimada: 25 min
  - [Ver capítulo completo](Capitulo01/README.md)

### Capítulo 2

- [Crear y usar módulos de red y máquinas virtuales.](Capitulo02/README.md#crear-y-usar-módulos-de-red-y-máquinas-virtuales)
  - Descripción: Crear y consumir un módulo local y para redes y VMs, separando recursos por lógica y reutilización.
  - Duración estimada: 55 min
  - [Ver capítulo completo](Capitulo02/README.md)

### Capítulo 3

- [Migrar terraform.tfstate a Google Cloud Storage con control de bloqueo activado.](Capitulo03/README.md#migrar-terraformtfstate-a-google-cloud-storage-con-control-de-bloqueo-activado)
  - Descripción: Migrar terraform.tfstate a Google Cloud Storage con control de bloqueo activado.
  - Duración estimada: 45 min
  - [Ver capítulo completo](Capitulo03/README.md)

### Capítulo 4

- [Crear múltiples entornos (dev, test, prod) con terraform workspace y observar separación del estado.](Capitulo04/README.md#crear-múltiples-entornos-dev-test-prod-con-terraform-workspace-y-observar-separación-del-estado)
  - Descripción: Crear múltiples entornos (dev, test, prod) con terraform workspace y observar separación del estado.
  - Duración estimada: 35 min
  - [Ver capítulo completo](Capitulo04/README.md)

### Capítulo 5

- [Crear archivos .tfvars por entorno y outputs condicionales.](Capitulo05/README.md#crear-archivos-tfvars-por-entorno-y-outputs-condicionales)
  - Descripción: Usar mapas y listas de objetos para definir configuraciones por entorno, y condicionar la salida de variables.
  - Duración estimada: 35 min
  - [Ver capítulo completo](Capitulo05/README.md)

### Capítulo 6

- [Crear múltiples subredes y validar valores de entrada.](Capitulo06/README.md#crear-múltiples-subredes-y-validar-valores-de-entrada)
  - Descripción: Crear múltiples subredes usando for_each, validando entradas con validation {} en variables.
  - Duración estimada: 55 min
  - [Ver capítulo completo](Capitulo06/README.md)

### Capítulo 7

- [Crear pipeline que haga plan en pull requests y apply en merges.](Capitulo07/README.md#crear-pipeline-que-haga-plan-en-pull-requests-y-apply-en-merges)
  - Descripción: Crear pipeline que ejecute terraform plan en PRs y apply en merge, parametrizado por entorno
  - Duración estimada: 65 min
  - [Ver capítulo completo](Capitulo07/README.md)

### Capítulo 8

- [Usar secretos desde Google Cloud Secret Manager y validar que no se expongan.](Capitulo08/README.md#usar-secretos-desde-google-cloud-secret-manager-y-validar-que-no-se-expongan)
  - Descripción: Leer secretos desde GCP Secret Manager usando un provider seguro con variables sensibles.
  - Duración estimada: 45 min
  - [Ver capítulo completo](Capitulo08/README.md)

### Capítulo 9

- [Refactorizar un proyecto real monolítico en modular y validado.](Capitulo09/README.md#refactorizar-un-proyecto-real-monolítico-en-modular-y-validado)
  - Descripción: Refactorizar un proyecto plano a modular, aplicar terraform fmt, validate y documentación en README.md.
  - Duración estimada: 40 min
  - [Ver capítulo completo](Capitulo09/README.md)

### Capítulo 10

- [Escanear código con tfsec](Capitulo10/README.md#escanear-código-con-tfsec)
  - Descripción: Escanear el código con tfsec
  - Duración estimada: 20 min
- [Generar doc automática con terraform-docs](Capitulo10/README.md#generar-doc-automática-con-terraform-docs)
  - Descripción: Usar terraform-docs para generar documentación automática
  - Duración estimada: 20 min
- [Estimar costos con infracost](Capitulo10/README.md#estimar-costos-con-infracost)
  - Descripción: Estimar costos reales con infracost
  - Duración estimada: 25 min
- [Crear un wrapper básico con Terragrunt](Capitulo10/README.md#crear-un-wrapper-básico-con-terragrunt)
  - Descripción: Implementar estructura base con Terragrunt para múltiples entornos
  - Duración estimada: 30 min
  - [Ver capítulo completo](Capitulo10/README.md)

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.

---

*Material didáctico preparado por Global K, S.A. de C.V.*
