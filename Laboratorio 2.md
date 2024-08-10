# Laboratorio 2 - Configuración Inicial de los Servicios

![cloudlogo](Images/cloud_computing.jpg)

- [Laboratorio 2 - Configuración Inicial de los Servicios](#laboratorio-2---configuración-inicial-de-los-servicios)
  - [Objetivo](#objetivo)
  - [Herramientas a usar](#herramientas-a-usar)
  - [Procedimiento](#procedimiento)
    - [Integración de aplicaciones](#integración-de-aplicaciones)
      - [Preparación del ambiente](#preparación-del-ambiente)
      - [Integración vía SAML](#integración-vía-saml)
    - [Implementación de redes virtuales](#implementación-de-redes-virtuales)

## Objetivo

Por medio de la ejecución de este conjunto de actividades se espera afianzar los conceptos de despliegue de componentes en un __CSP__ e ir aplicando elementos de seguridad.

## Herramientas a usar

A continuación se listan las herramientas a utilizar para el laboratorio:

| Nombre | Sitio Web | Logo |
| --- | --- | --- |
| Azure | <https://portal.azure.com/> | ![AzureLogo](Images/Microsoft-Azure-Symbol.png)|
| Visual Studio Code | <https://code.visualstudio.com/download> | ![VSCodeLogo](Images/vscode.png) |
| GIT for Windows | <https://gitforwindows.org/> | ![GITLogo](Images/git_logo.png)|
| Node JS for Windows | <https://nodejs.org/> | ![NodeJSLogo](Images/nodejslogo.jpg)|
| ngrok | <https://ngrok.com/> | ![NodeJSLogo](Images/ngrok-blue-med.png)|

1. Explorador de Internet de su preferencia (Chrome, Edge, o Firefox)
2. Acceso por Internet a uno de los proveedores de servicio de computación en la nube
3. Cliente para la conexión remota a equipos (Cliente de Escritorio Remoto - RDP)
4. Cliente para la conexión por SSH a equipos

## Procedimiento

Realizar el siguiente conjunto de actividades para el desarrollo del laboratorio.

> __Nota:__ Recuerde documentar por medio de pantallazos la ejecución de las diferentes actividades con el fin de realizar un documento que quede como evidencia del trabajo en equipo. Este documento es el que deberá ser cargado en el espacio de Google Classroom provisto para ello.

### Integración de aplicaciones

En este laboratorio se espera poder realizar la integración de una aplicación __dummy__ para ser autenticada con el directorio de su Tenant de Azure

#### Preparación del ambiente

Ejecute los siguientes pasos en su equipo o en alguna máquina virtual para el despliegue de las aplicaciones.

1. Vaya a al sitio web de [ngrok](https://ngrok.com/)
2. Realice la creación de una cuenta en el [sitio web](https://dashboard.ngrok.com/signup)
3. Descargue el ejecutable de la aplicación. Descomprima el archivo en una carpeta de fácil acceso por terminal (ej. c:\temp).
4. Abra una línea de comando y ejecute los siguientes comandos:

```powershell
cd c:\temp
.\ngrok.exe config add-authtoken "token_entregado_en_sitio_web"
```

5. 

#### Integración vía SAML

   1. Verifique que tenga acceso al sitio web de la aplicación [Test Service Provider](https://sptest.iamshowcase.com/)
   2. En la página web diríjase a la sección de [instrucciones](https://sptest.iamshowcase.com/instructions)
   3. Descargue el archivo que se le indica en el botón de [Download Metadata](https://sptest.iamshowcase.com/testsp_metadata.xml); guarde ese archivo ya que se usará mas adelante
   4. Ejecute los pasos indicados en alguna de las siguientes guías:
      1. [Guía de referencia 1](https://boxyhq.com/docs/jackson/sso-providers/azure) de los pasos a seguir en la configuración de una aplicación empresarial vía __SAML__
      2. [Guía de referencia 2](https://docs.digicert.com/en/trust-lifecycle-manager/how-to-guides/configure-a-profile-to-authenticate-requests-via-saml-2-0-using-microsoft-azure-ad-saml-idp/create-saml-idp-applications-in-azure-ad-portal.html).

   > __Nota__: Recuerde hacer uso del archivo de metadatos que descargo en los pasos anteriores. En la __guía de referencia 2__ no ejecutar los pasos descritos del 15 al 19.

   5. Una vez cargue el archivo de metadatos, en la ventana que se indica a continuación, busque la sección de __Relay State (Optional)__ y coloque el siguiente valor: __https://sptest.iamshowcase.com/protected?color=pink__. Recuerde luego de ingresar el parámetro de guardar la configuración.
    ![SAMLConfig1](./Images/SAMLConfig1.png)
   6. Guarde el archivo de metadatos que descargo del portal de Azure y realice la carga en la página de la aplicación de prueba [aquí](https://sptest.iamshowcase.com/instructions#spinit)
    ![SAMLConfig2](./Images/SAMLConfig2.png)
   7. Luego de cargar todo, haga click en la opción que dice __Protected Page__. Aquí la aplicación va a solicitar que se autentique con algún usuario que esté creado en su directorio.
    ![SAMLConfig3](./Images/SAMLConfig3.png)
   8. Si todo esta correcto, usted será redirigido a una página que contiene toda la información del usuario
    ![SAMLConfig4](./Images/SAMLConfig4.png)

### Implementación de redes virtuales

En este laboratorio de realizará la implementación de las definiciones de redes virtuales (VNET's o VCN's), se realizarán interconexiones entre ellas y finalmente se hará prueba de una topología de Hub and Spoke. Ejecute los pasos indicados en las siguientes guías:

   1. [Implementación de redes virtuales en Azure](./AZ-104-Labs/Instructions/Labs/LAB_04-Implement_Virtual_Networking.md)
   2. [Interconexión entre redes virtuales en Azure](./AZ-104-Labs/Instructions/Labs/LAB_05-Implement_Intersite_Connectivity.md)
   3. [Configuración avanzada del tráfico de red en Azure](./AZ-104-Labs/Instructions/Labs/LAB_06-Implement_Network_Traffic_Management.md)