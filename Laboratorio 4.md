# Laboratorio 4 - - Implementación de DevSecOps

![cloudlogo](Images/cloud_computing.jpg)

- [Laboratorio 4 - - Implementación de DevSecOps](#laboratorio-4-----implementación-de-devsecops)
  - [Objetivo](#objetivo)
  - [Herramientas a usar](#herramientas-a-usar)
  - [| Checkov | https://www.checkov.io/ |  |](#-checkov--httpswwwcheckovio---)
    - [Herramientas Adicionales](#herramientas-adicionales)
  - [Requisitos](#requisitos)
    - [Creación de cuentas en los servicios en línea de las herramientas](#creación-de-cuentas-en-los-servicios-en-línea-de-las-herramientas)
    - [Creación del entorno de Azure DevOps](#creación-del-entorno-de-azure-devops)
  - [Procedimiento](#procedimiento)
    - [Laboratorios de seguridad](#laboratorios-de-seguridad)

## Objetivo

En este laboratorio se tienen los siguientes objetivos

1. Realizar el flujo completo de creación y despliegue de una aplicación en contenedores con ayuda de las herramientas automatizadas
2. Incluir elementos de análisis de seguridad dentro de flujo automatizado de despliegue

## Herramientas a usar

Para la ejecución de los siguientes laboratorio se hará uso de las siguientes herramientas

| Nombre | Dirección Web | Logo |
|---------|---------|---------|
| Azure | <https://portal.azure.com>| ![AzureLogo](./Images/Microsoft-Azure-Symbol.png) |
| Azure DevOps | <https://dev.azure.com> | ![DevOpsLogo](./Images/DevOpsLogo.png) |
| Snyk | <https://app.snyk.io/login> | ![SnykLogo](./Images/Snyk.png)|
| Sonarcloud | <https://sonarcloud.io/login> | ![Sonarlogo](./Images/SonarQube_Logo.png) |
| Trivy | <https://trivy.dev/> | ![TrivyLogo](./Images/Trivy.png) |
| Mend | <https://www.mend.io/> | ![MendLogo](./Images/MendLogo.png) |
| .NET Framework 6.0 | <https://dotnet.microsoft.com/es-es/download/dotnet/6.0> | ![Dotnetlogo](./Images/dotnet-6.0-logo.png) |
| Visual Studio Code | <https://code.visualstudio.com/download> | ![VSCodeLogo](Images/vscode.png) |
| GIT for Windows | <https://gitforwindows.org/> | ![GITLogo](Images/git_logo.png)|
| node.js| <https://nodejs.org/en> | ![nodejslogo](./Images/nodejslogo.jpg) |
| Checkov | <https://www.checkov.io/> | ![CheckovLogo](Images/Checkov.png) |
---

### Herramientas Adicionales

1. Explorador de Internet de su preferencia (Chrome, Edge, o Firefox)
2. Acceso por Internet a uno de los proveedores de servicio de computación en la nube

## Requisitos

Para la ejecución del laboratorio es indispensable realizar los siguientes pasos para poder continuar

### Creación de cuentas en los servicios en línea de las herramientas

1. Vaya la página de [GitHub](https://github.com/) y realice la creación de una cuenta con la ayuda de un correo personal, siga los pasos que le sean indicados.
2. Ya con la cuenta de GitHub creada, abra en el mismo navegador en pestañas independientes los siguientes repositorios:
   1. <https://github.com/malevarro/reactjs-shopping-cart>
   2. <https://github.com/MicrosoftDocs/mslearn-tailspin-spacegame-web>

3. Siga las instrucciones de la siguiente guía para realizar la copia (Fork/Bifurcación) de cada uno de los repositorios a su cuenta de GitHub

   - [Bifurcar (Fork) un repositorio](https://docs.github.com/es/get-started/quickstart/fork-a-repo?tool=webui)

4. Una vez haya creado la cuenta en GitHub ingrese a las siguientes URL en el mismo navegador en pestañas diferentes y seleccione la opción de registro con __GitHub__ y de esta forma integrar todas los servicios bajo una misma cuenta.

   | Aplicación | URL de Registro |
   | --- | --- |
   | Snyk | <https://app.snyk.io/login> |
   | Sonarcloud | <https://sonarcloud.io/login> |

### Creación del entorno de Azure DevOps

> __NOTA:__ Recuerde crear el registro en todo los servicios con la misma cuenta que tiene para Azure.

1. Vaya a la siguiente pagina que indica el procedimiento de registro de [Azure DevOps](https://www.devjev.nl/posts/2022/how-to-create-an-organization-in-azure-devops/#what-is-an-azure-devops-organization)
2. Siga los pasos indicados en el video.
3. Una vez registrados en Azure DevOps, abra el [Portal de Azure](https://portal.azure.com)
4. Dentro del portal haga la búsqueda de  Azure DevOps Organizations

![devops1](./Images/devops1.png)

5. En la nueva ventana haga clic en _My Azure DevOps Organizations_

![devops2](./Images/devops2.png)

6. En la nueva ventana verifique que en la sección de _Azure DevOps Organizations_ exista alguna, haga click encima de ella para entrar a configurar su entorno

![devops3](./Images/devops3.png)

7. Una vez haya ingresado a la organización, siga los pasos en la siguiente [guía](https://www.devjev.nl/posts/2022/how-to-create-a-new-project-in-azure-devops/) para realizar la creación de un proyecto.
8. Siga los pasos de la siguiente [guía](https://www.devjev.nl/posts/2023/how-to-create-a-new-azure-service-connection-in-azure-devops/#how-to-create-a-new-azure-service-connection-in-azure-devops) para la integración de Azure DevOps con el portal de Azure y se permita la creación de componentes en la nube.

## Procedimiento

Realizar el siguiente conjunto de actividades para el desarrollo del laboratorio.

> __Nota:__ Recuerde documentar por medio de pantallazos la ejecución de las diferentes actividades con el fin de realizar un documento que quede como evidencia del trabajo en equipo. Este documento es el que deberá ser cargado en el espacio de Google Classroom provisto para ello.

### Laboratorios de seguridad

1. Siga la siguiente guía para realiza la [Automatización de implementaciones de contenedores de Docker con Azure Pipelines](https://learn.microsoft.com/en-us/training/modules/deploy-docker/?WT.mc_id=devopsgen_inproduct-15843-dabrady) y de esta forma despleguar una aplicación bajo este esquema en la nube de azure.

> __NOTA:__ No ejecute el último paso de borrado de los recursos de Azure DevOps en este momento. esta tarea la debe realizar hasta finalizar todo los laboratorios.

2. Integre Snyk con Azure DevOps de acuerdo con la guía del siguiente [enlace](https://docs.snyk.io/scm-ide-and-ci-cd-integrations/snyk-ci-cd-integrations/azure-pipelines-integration)
