# 🛡️ Guía de Laboratorios: Seguridad en la Nube (CS-Specialization)

**Institución:** Escuela de Comunicaciones Militares  
**Programa:** Especialización en Ciberseguridad  
**Instructor:** MSc. Manuel Alejandro Vargas  
**Nube de Referencia:** Microsoft Azure  
**Metodología:** Aprendizaje Basado en Investigación (ABI)

---

## 📋 Descripción General
Este repositorio contiene las guías técnicas detalladas para el desarrollo del componente práctico del módulo **Seguridad en la Nube**. Las actividades están diseñadas para simular escenarios reales de defensa, cumplimiento y respuesta a incidentes utilizando servicios nativos de Azure y herramientas Open Source.

### 🛠️ Prerrequisitos
* **Suscripción de Azure:** (Free Tier o Azure for Students).
* **Azure CLI:** [Instalar aquí](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).
* **Cuenta de GitHub:** Para gestión de código y CI/CD.
* **Conocimientos base:** Redes, Linux básico y Docker.

---

## 🚀 Laboratorio 1: Fundamentos y Gobernanza en la Nube
**Enfoque:** Responsabilidad Compartida y Cumplimiento Normativo.

### 🔗 Recursos Base
> **Nota:** Este laboratorio integra elementos del repositorio del docente.
> * 📄 **Guía de Referencia:** [Cloud-Lab/Laboratorio 1.md](https://github.com/malevarro/Cloud-Lab/blob/main/Laboratorio%201.md)

### 🎯 Objetivos de Aprendizaje
1.  Identificar los controles de seguridad en modelos IaaS, PaaS y SaaS.
2.  Evaluar el cumplimiento normativo (ISO/SOC2) de los servicios de Azure.

### 📝 Actividad: Mapeo de Cumplimiento
1.  **Exploración de Azure Policy:**
    * Acceda al portal de Azure y busque el servicio **Policy**.
    * Asigne la iniciativa *"NIST SP 800-53 Rev. 5"* a su suscripción.
    * **Investigación:** Analice qué recursos actuales aparecen como "Non-compliant".
2.  **Análisis de Responsabilidad (Basado en Lab 1 Referencia):**
    * Siguiendo la estructura del enlace base, despliegue una Máquina Virtual (IaaS) y una SQL Database (PaaS).
    * Complete la matriz de responsabilidad entregada en el anexo del laboratorio, identificando quién gestiona:
        * Parcheo del OS.
        * Cifrado de datos en reposo.
        * Gestión de identidad.

---

## 🚀 Laboratorio 2: Auditoría de Postura de Seguridad (CSPM)
**Enfoque:** Inteligencia de Amenazas y Detección de Brechas.

### 🎯 Objetivos de Aprendizaje
1.  Ejecutar escaneos de seguridad automatizados contra benchmarks internacionales (CIS).
2.  Remediar hallazgos críticos de configuración.

### 📝 Actividad: Auditoría con Herramientas Open Source
1.  **Configuración del Entorno:**
    * Abra **Azure Cloud Shell** (Bash).
2.  **Ejecución de Prowler/Steampipe:**
    * Utilizaremos una herramienta de CSPM para auditar la cuenta.
    ```bash
    # Ejemplo de instalación rápida de Prowler (sujeto a actualización)
    pip install prowler
    prowler azure --list-services
    ```
3.  **Análisis de Hallazgos:**
    * Genere un reporte en formato HTML.
    * Identifique fallos en: *Storage Accounts públicas*, *MFA no habilitado* y *Puertos de gestión abiertos (22/3389)*.

---

## 🚀 Laboratorio 3: Seguridad de Infraestructura y Redes (Zero Trust)
**Enfoque:** Microsegmentación y Protección de Red.

### 🔗 Recursos Base
> **Nota:** Se requiere clonar los scripts de infraestructura del repositorio del docente.
> * 📄 **Guía de Referencia:** [Cloud-Lab/Laboratorio 3.md](https://github.com/malevarro/Cloud-Lab/blob/main/Laboratorio%203.md)

### 🎯 Objetivos de Aprendizaje
1.  Implementar Grupos de Seguridad de Red (NSG) y de Aplicación (ASG).
2.  Diseñar una arquitectura de red bajo principios de Zero Trust.

### 📝 Actividad: Hardening de Red en Azure
1.  **Despliegue de Infraestructura:**
    * Utilice los scripts del enlace base (`Laboratorio 3.md`) para desplegar una VNet con dos subredes (Frontend y Backend).
2.  **Microsegmentación:**
    * Cree un **Network Security Group (NSG)** que *deniegue todo el tráfico* por defecto.
    * Permita únicamente tráfico HTTP (80) hacia la subred Frontend.
    * Permita tráfico SQL (1433) **solo** desde la subred Frontend hacia la Backend (No desde internet).
3.  **Investigación - Just In Time (JIT):**
    * Investigue y active (si es posible en su licencia) o simule el acceso **JIT VM Access** de Microsoft Defender for Cloud para el puerto de administración.

---

## 🚀 Laboratorio 4: DevSecOps y Protección de Datos
**Enfoque:** Seguridad en el Ciclo de Vida del Software (CI/CD) y Secretos.

### 🔗 Recursos Base
> **Nota:** Este laboratorio utiliza flujos de trabajo de GitHub Actions.
> * 📄 **Guía de Referencia:** [Cloud-Lab/Laboratorio 4.md](https://github.com/malevarro/Cloud-Lab/blob/main/Laboratorio%204.md)

### 🎯 Objetivos de Aprendizaje
1.  Detectar credenciales expuestas (Secret Scanning).
2.  Integrar análisis estático (SAST) en un Pipeline.

### 📝 Actividad: Pipeline Seguro con GitHub y Azure
1.  **Gestión de Secretos (Basado en Lab 4 Referencia):**
    * No hardcodee credenciales. Configure **GitHub Secrets** para almacenar `AZURE_CREDENTIALS`.
    * Modifique el archivo YAML del workflow para inyectar estas credenciales como variables de entorno.
2.  **Análisis de Código (IaC):**
    * Integre **Checkov** o **Trivy** en su GitHub Action.
    * El pipeline debe fallar (break build) si detecta que el template de ARM/Terraform intenta crear un Storage Account sin cifrado.
    ```yaml
    # Ejemplo de paso en GitHub Actions
    - name: Run Checkov action
      uses: bridgecrewio/checkov-action@master
      with:
        directory: ./infrastructure
    ```
3.  **Protección de Contenedores:**
    * Suba una imagen a **Azure Container Registry (ACR)** y revise el panel de seguridad de Azure para ver las vulnerabilidades detectadas en la imagen base.

---

## 🚀 Laboratorio 5: Forense y Respuesta a Incidentes
**Enfoque:** Operaciones de Seguridad (SecOps) y Análisis Forense.

### 🎯 Objetivos de Aprendizaje
1.  Investigar incidentes de seguridad utilizando logs de nube.
2.  Desarrollar Playbooks de respuesta.

### 📝 Actividad: "Cacería" de Amenazas en Azure
1.  **Escenario:** Se ha detectado una eliminación masiva de recursos.
2.  **Investigación en Azure Monitor:**
    * Acceda a los **Activity Logs**.
    * Filtre por eventos críticos en las últimas 24 horas.
    * Identifique: *¿Qué usuario (Identity) inició la acción?*, *¿Desde qué dirección IP?*, *¿Cuál fue el User Agent?*.
3.  **Kusto Query Language (KQL):**
    * Ejecute una consulta en Log Analytics para correlacionar eventos:
    ```kusto
    AzureActivity
    | where OperationNameValue contains "Delete"
    | summarize count() by Caller, CallerIpAddress
    | render barchart
    ```
4.  **Entregable Final:** Un "Incidence Response Report" detallando la línea de tiempo del ataque y las medidas de contención propuestas.

---

## 📦 Entrega de Resultados
Para cada sesión, el estudiante debe subir a su repositorio personal:
1.  Código fuente modificado (Scripts, Templates).
2.  Reporte en Markdown (`Reporte_SesionX.md`) con capturas de pantalla de la evidencia.
3.  Conclusiones técnicas basadas en la bibliografía IEEE del curso.

---
*Escuela de Comunicaciones Militares IU CEDOC - Especialización en Ciberseguridad © 2026*