# Guía de laboratorio – Despliegue de arquitectura Hub & Spoke en Azure

- [Guía de laboratorio – Despliegue de arquitectura Hub \& Spoke en Azure](#guía-de-laboratorio--despliegue-de-arquitectura-hub--spoke-en-azure)
  - [Objetivo del laboratorio](#objetivo-del-laboratorio)
    - [Arquitectura de referencia](#arquitectura-de-referencia)
      - [Diagrama (arquitectura objetivo)](#diagrama-arquitectura-objetivo)
    - [Requisitos previos](#requisitos-previos)
    - [Componentes del laboratorio](#componentes-del-laboratorio)
    - [Pasos de creación de componenetes en Azure](#pasos-de-creación-de-componenetes-en-azure)
      - [Paso 1 – Creación del Resource Group](#paso-1--creación-del-resource-group)
      - [Paso 2 – Creación de la VNet Hub (2 subredes)](#paso-2--creación-de-la-vnet-hub-2-subredes)
      - [Paso 3 – Creación de las VNets Spoke (1 subred cada una)](#paso-3--creación-de-las-vnets-spoke-1-subred-cada-una)
      - [Paso 4 – Despliegue de máquinas virtuales Linux](#paso-4--despliegue-de-máquinas-virtuales-linux)
        - [VM Firewall en Hub (dual NIC)](#vm-firewall-en-hub-dual-nic)
      - [Paso 5 – Configuración de VNet Peering (Hub ↔ Spokes)](#paso-5--configuración-de-vnet-peering-hub--spokes)
      - [Paso 6 – Configuración de NSG por subred](#paso-6--configuración-de-nsg-por-subred)
        - [NSG para Subnet-Back (Backend / Egress)](#nsg-para-subnet-back-backend--egress)
        - [NSG para el resto de subredes (Subnet-Front y Spokes)](#nsg-para-el-resto-de-subredes-subnet-front-y-spokes)
      - [Paso 7 – Configuración paso a paso del Firewall Linux (NVA) usando FWBuilder (iptables)](#paso-7--configuración-paso-a-paso-del-firewall-linux-nva-usando-fwbuilder-iptables)
      - [Paso 8 – Enrutamiento (UDR) para forzar tráfico a través del Firewall](#paso-8--enrutamiento-udr-para-forzar-tráfico-a-través-del-firewall)
    - [Validación avanzada (Network Watcher + Linux)](#validación-avanzada-network-watcher--linux)
      - [Validación funcional desde las VMs](#validación-funcional-desde-las-vms)
      - [Revisar “Effective routes” (rutas efectivas) en la NIC](#revisar-effective-routes-rutas-efectivas-en-la-nic)
      - [Network Watcher – Next hop (diagnóstico de enrutamiento)](#network-watcher--next-hop-diagnóstico-de-enrutamiento)
      - [Network Watcher – Connection troubleshoot (diagnóstico de conectividad)](#network-watcher--connection-troubleshoot-diagnóstico-de-conectividad)
      - [Validación en el firewall (captura y counters)](#validación-en-el-firewall-captura-y-counters)
    - [Resultados esperados](#resultados-esperados)

## Objetivo del laboratorio

El objetivo de este laboratorio es que el participante diseñe y despliegue paso a paso una arquitectura de red **Hub & Spoke** en Microsoft Azure, incorporando una **máquina virtual Linux actuando como firewall / router (NVA)** configurada mediante FWBuilder (políticas compiladas a iptables) en la VNet Hub y con NSG por subred aplicando mínimo privilegio. Al finalizar el laboratorio, el estudiante será capaz de:

- Crear **3 redes virtuales** (1 Hub y 2 Spoke).
- Desplegar **una VM Linux por VNet** con el **tamaño mínimo** (ej.: Standard_B1ls, sujeto a disponibilidad regional/suscripción).
- Implementar una VM en el Hub con **dos interfaces de red (dual-homed)**, una por subred del Hub.
- Configurar **VNet Peering** (Hub ↔ Spokes) con tráfico reenviado (forwarded).
- Forzar Spoke-to-Spoke y salida a Internet a través del firewall (UDR).
- Aplicar NSG por subred según requerimientos:
  - Subnet-Back: todo outbound a Internet y solo SSH inbound.
  - Todas las demás subredes: permitir inbound/outbound únicamente hacia rangos privados 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16.

- Ejecutar **validación avanzada** con **Azure Network Watcher** (Next hop, Effective routes, Connection troubleshoot).

### Arquitectura de referencia

Despliegue de una arquitectura Hub & Spoke en Microsoft Azure con Firewall Linux (NVA) y validación avanzada.

Se toma como referencia la [topología de referencia Hub-spoke](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke)

#### Diagrama (arquitectura objetivo)

```shell
                        (Internet)
                             ^
                             |
                      [Azure SNAT/Outbound]
                             |
                    +----------------------+
                    |  Hub-VNet 10.0.0.0/16|
                    |                      |
 Spoke1 Peering     |     Subnet-Back      |     Spoke2 Peering
+----------------+  |     10.0.1.0/24      |  +----------------+
| Spoke1-VNet    |  |     (WAN/EGRESS)     |  | Spoke2-VNet    |
| 10.1.0.0/16    |  |     [NIC2 - back]    |  | 10.2.0.0/16    |
| WorkloadSubnet |  |         |            |  | WorkloadSubnet |
| 10.1.0.0/24    |--+      +--------+      +--| 10.2.0.0/24    |
|  [VM-Spoke1]   |  |     | FW-Linux |     |  |   [VM-Spoke2]  |
+----------------+  |     |   (NVA)  |     |  +----------------+
                    |      +--------+      |
                    |         |            |
                    |    [NIC1 - front]    |
                    |     Subnet-Front     |
                    |     10.0.0.0/24      |
                    +----------------------+

Rutas (UDR) en Spokes:
- 0.0.0.0/0  -> Next hop = VirtualAppliance (IP de NIC1 en Subnet-Front)
- Spoke1: 10.2.0.0/16 -> Next hop = VirtualAppliance (IP de NIC1)
- Spoke2: 10.1.0.0/16 -> Next hop = VirtualAppliance (IP de NIC1)

```

> **Nota:** La VM firewall es un “router” multi-interfaz. El IP forwarding debe estar habilitado tanto en Azure (NICs) como en el sistema operativo Linux para que reenvíe tráfico.

### Requisitos previos

- Suscripción activa de Azure.
- Permisos Owner o Contributor.
- Azure CLI o Azure Cloud Shell.
- Conocimientos básicos de redes IP, rutas, Linux.

### Componentes del laboratorio

| Componente | Nombre | Dirección/Detalle |
| ---- | ---- | ---- |
| Resource Group | RG-Networking | eastus |
| VNet Hub | Hub-VNet | 10.0.0.0/16 |
| Subnet Hub Front | Subnet-Front | 10.0.0.0/24 |
| Subnet Hub Back | Subnet-Back | 10.0.1.0/24 |
| VNet Spoke 1 | Spoke1-VNet | 10.1.0.0/16 |
| Subnet Spoke 1 | WorkloadSubnet | 10.1.0.0/24 |
| VNet Spoke 2 | Spoke2-VNet | 10.2.0.0/16 |
| Subnet Spoke 2 | WorkloadSubnet | 10.2.0.0/24 |
| VM Firewall | FW-Linux | 2 NICs (Front + Back) |
| VM Spoke 1 | VM-Spoke1 | 1 NIC |
| VM Spoke 2 | VM-Spoke2 | 1 NIC |
| Tamaño VM | Standard_B1ls | mínimo típico (puede variar por suscripción/región) |

### Pasos de creación de componenetes en Azure

#### Paso 1 – Creación del Resource Group

En el la consola CLI en el portal de Azure

```shell
az group create \
  --name RG-Networking \
  --location eastus
```

#### Paso 2 – Creación de la VNet Hub (2 subredes)

En el la consola CLI en el portal de Azure

Creeación de la VNET

```shell
az network vnet create \
  --resource-group RG-Networking \
  --name Hub-VNet \
  --address-prefix 10.0.0.0/16
```

Creación de las SUBNET's en HUB

- Subnet-Front (tránsito Spokes)

```shell
az network vnet subnet create \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --name Subnet-Front \
  --address-prefix 10.0.0.0/24
```

- Subnet-Back (egress/Internet)

```shell
az network vnet subnet create \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --name Subnet-Back \
  --address-prefix 10.0.1.0/24
```

#### Paso 3 – Creación de las VNets Spoke (1 subred cada una)

En el la consola CLI en el portal de Azure

- Spoke 1

```shell
az network vnet create \
  --resource-group RG-Networking \
  --name Spoke1-VNet \
  --address-prefix 10.1.0.0/16 \
  --subnet-name WorkloadSubnet \
  --subnet-prefix 10.1.0.0/24
```

- Spoke 2

```shell
az network vnet create \
  --resource-group RG-Networking \
  --name Spoke2-VNet \
  --address-prefix 10.2.0.0/16 \
  --subnet-name WorkloadSubnet \
  --subnet-prefix 10.2.0.0/24
```

#### Paso 4 – Despliegue de máquinas virtuales Linux

Se utilizará el tamaño *Standard_B1ls* (uno de los más pequeños permitidos en Azure). Se puede seleccionar cualquier otro tipo de tamaño de máquina, este es sólo un valor de referencia.

##### VM Firewall en Hub (dual NIC)

En el la consola CLI en el portal de Azure

- Crear NIC Front (Subnet-Front):

```shell
az network nic create \
  --resource-group RG-Networking \
  --name fw-nic-front \
  --vnet-name Hub-VNet \
  --subnet Subnet-Front
```

- Crear NIC Back (Subnet-Back):

```shell
az network nic create \
  --resource-group RG-Networking \
  --name fw-nic-back \
  --vnet-name Hub-VNet \
  --subnet Subnet-Back
```

- **Importante** - Habilitar IP forwarding en ambas NIC (Azure): En una NVA con múltiples NIC, el IP forwarding debe habilitarse en cada interfaz que reciba tráfico para reenviar.

```shell
az network nic update \
  --resource-group RG-Networking \
  --name fw-nic-front \
  --ip-forwarding true

az network nic update \
  --resource-group RG-Networking \
  --name fw-nic-back \
  --ip-forwarding true
```

- Crear VM Firewall (2 NICs):

**Recuerde** reemplazar el valor de <CONTRASEÑA_SEGURA> por algo asignado por usted

```shell
az vm create \
  --resource-group RG-Networking \
  --name FW-Linux \
  --image Ubuntu2204 \
  --size Standard_B1ls \
  --nics fw-nic-front fw-nic-back \
  --admin-username azureuser \
  --authentication-type password \
  --admin-password <CONTRASEÑA_SEGURA>
```

 **Importante:** en Azure, una NIC será primaria (típicamente la primera creada/asignada). En este laboratorio, la NIC asociada a Subnet-Back se usa como salida natural (default route) hacia Internet.

- Crear VM Spoke 1

**Recuerde** reemplazar el valor de <CONTRASEÑA_SEGURA> por algo asignado por usted

```shell
az vm create \
  --resource-group RG-Networking \
  --name VM-Spoke1 \
  --image Ubuntu2204 \
  --size Standard_B1ls \
  --vnet-name Spoke1-VNet \
  --subnet WorkloadSubnet \
  --admin-username azureuser \
  --authentication-type password \
  --admin-password <CONTRASEÑA_SEGURA>
```

- Crear VM Spoke 2

**Recuerde** reemplazar el valor de <CONTRASEÑA_SEGURA> por algo asignado por usted

```shell
az vm create \
  --resource-group RG-Networking \
  --name VM-Spoke2 \
  --image Ubuntu2204 \
  --size Standard_B1ls \
  --vnet-name Spoke2-VNet \
  --subnet WorkloadSubnet \
  --admin-username azureuser \
  --authentication-type password \
  --admin-password <CONTRASEÑA_SEGURA>
```

#### Paso 5 – Configuración de VNet Peering (Hub ↔ Spokes)

Para el uso de equipos tipo NVAs, se requiere permitir forwarded traffic (tráfico reenviado) para que el Hub pueda actuar como tránsito.

En el la consola CLI en el portal de Azure

- Peering Spoke1 → Hub

```shell
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
  --allow-forwarded-t
```

- Peering Spoke2 → Hub

```shell
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

#### Paso 6 – Configuración de NSG por subred

##### NSG para Subnet-Back (Backend / Egress)

Objetivo:

1. Permitir todo el tráfico saliente a Internet.
2. Permitir solo SSH entrante (22/tcp) hacia el firewall Linux.
3. Bloquear cualquier otro tráfico entrante

En el la consola CLI en el portal de Azure

- Crear NSG para Subnet-Back

```shell
az network nsg create \
  --resource-group RG-Networking \
  --name NSG-Subnet-Back
```

- Crear reglas del NSG para Subnet-Back

```shell
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
```

- Asociar NSG a Subnet-Back

```shell
az network vnet subnet update \
  --resource-group RG-Networking \
  --vnet-name Hub-VNet \
  --name Subnet-Back \
  --network-security-group NSG-Subnet-Back
```

##### NSG para el resto de subredes (Subnet-Front y Spokes)

Aplica a:

- Subnet-Front (Hub)
- WorkloadSubnet (Spoke1)
- WorkloadSubnet (Spoke2)

**Objetivo:**

- Permitir tráfico entrante y saliente únicamente desde/hacia:
  - 10.0.0.0/8
  - 172.16.0.0/12
  - 192.168.0.0/16
- Bloquear cualquier otro tráfico.

En el la consola CLI en el portal de Azure

- Crear NSG general para subredes internas

```shell
az network nsg create \
  --resource-group RG-Networking \
  --name NSG-Internal-Networks
```

- Crear reglas para NSG general de subredes internas

```shell
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
```

- Asociar NSG a las subredes

```shell
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

- Validación de NSG

Verifica reglas efectivas en una tarjeta de red (NIC) de alguna de las VM's

```shell
az network nic list-effective-nsg \
  --resource-group RG-Networking \
  --name <NIC_NAME> \
  -o table
```

> *Nota:* Reemplazar el valor de <NIC_NAME> por el nombre de alguna interfaz de red que posea alguna de las máquinas virtuales creadas.

#### Paso 7 – Configuración paso a paso del Firewall Linux (NVA) usando FWBuilder (iptables)

- Obtener IP de la NIC Front (será el “next hop” de los Spokes)

En el la consola CLI en el portal de Azure

```shell
FW_FRONT_IP=$(az network nic show \
  --resource-group RG-Networking \
  --name fw-nic-front \
  --query "ipConfigurations[0].privateIPAddress" -o tsv)

echo $FW_FRONT_IP
```

> **Importante:** Guardar estee valor de la dirección IP, ya que va a ser usado en las rutas UDR de los spokes mas adelante.

- Verificar si la VM tiene IP pública asociada

Primero, identifica si alguna de las NIC del firewall tiene una Public IP asociada (normalmente la NIC de Subnet-Back).

```shell
az network nic show \
  --resource-group RG-Networking \
  --name fw-nic-back \
  --query "ipConfigurations[0].publicIpAddress.id" \
  -o tsv
```

> **NOTA:**
>
> 1. Si el comando retorna un ID, la NIC ya tiene una IP pública asociada.
> 2. Si no retorna nada, debes crear y asociar una IP pública (siguiente paso).

- Crear una IP pública (si no existe)

Con el siguiente comando se crea una dirección IP pública en el la consola CLI en el portal de Azure

```shell
az network public-ip create \
  --resource-group RG-Networking \
  --name FW-Public-IP \
  --sku Standard \
  --allocation-method Static
```

- Asociar la IP pública a la NIC del firewall

Asocia la IP pública a la NIC Back del firewall `fw-nic-back`.

```shell
az network nic ip-config update \
  --resource-group RG-Networking \
  --nic-name fw-nic-back \
  --name ipconfig1 \
  --public-ip-address FW-Public-IP
```

> Buenas prácticas:
>
> - Usar IP pública solo para administración.
> - Restringir el acceso con NSG (en este laboratorio, solo SSH entrante).

- Obtener la dirección IP pública asignada

En el la consola CLI en el portal de Azure

```shell
az network public-ip show \
  --resource-group RG-Networking \
  --name FW-Public-IP \
  --query "ipAddress" \
  -o tsv
```

El comando anterior debe retornar algún valor parecido al siguiente

```shell
52.170.25.10

```

- Conectar a la máquina por la Interfaz pública
  
Conectarse con la ayuda del cliente SSH MobaXterm a la máquina de firewall por medio de la dirección IP pública obtenida en el paso anterior, y con el usuario y contraseña definidos al momento de la creación de la máquina.

La aplicación de MobaXterm la puede descargar [aquí](https://mobaxterm.mobatek.net/download.html)

> Para saber como configurar una conexión por SSH puede verificar alguno de los siguientes enlaces:
>
> - [Domina MobaXterm](https://youtu.be/a6D4ElCbYII?si=tcuVHQVIonbALJ0k)
> - [Guía MobaXterm](https://www.redeszone.net/analisis/software/mobaxterm-terminal-windows/)
> - [MobaXterm documentation](https://mobaxterm.mobatek.net/documentation.html)

- Identificar interfaces (Front/Back)

En la conexión SSH de la máquina ejecute los siguientes comandos

```shell
ip -br a
ip route
```

> **Nota:** Apunta los nombres (por ejemplo eth0 y eth1) y cuál tiene la ruta por defecto (default via ...).

- Habilitar reenvío de IP en el sistema (Linux)

El reenvío IP suele estar deshabilitado por hardening; para que un host actúe como router/firewall debe habilitarse. El benchmark CIS lo menciona como parámetro crítico (y advierte que es requerido en sistemas que actúan como router).

En la conexión SSH de la máquina ejecute los siguientes comandos

```shell
sudo tee /etc/sysctl.d/99-nva-forwarding.conf >/dev/null <<'EOF'
net.ipv4.ip_forward=1
# Evita problemas con rutas asimétricas típicas en NVAs
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
EOF

sudo sysctl --system

```

- Instalar y habilitar iptables y fwbuilder

En benchmarks CIS modernos, iptables se considera deprecado en distribuciones recientes y se recomienda nftables para nuevos scripts/configuraciones.

En la conexión SSH de la máquina ejecute los siguientes comandos

```shell
sudo apt-get update
sudo apt-get install -y fwbuilder
```

- Crear política de firewall + NAT (nftables)

> Objetivo:
>
> - Permitir forwarding Front → Back (salida a Internet).
> - Permitir retorno Back → Front solo si es established/related.
> - Permitir forwarding Spoke1 ↔ Spoke2 (Front ↔ Front) porque ambos llegan por el Hub.
> - Hacer SNAT/Masquerade por la interfaz Back para que los spokes tengan salida a Internet.
> iptables soporta NAT con postrouting (masquerade/SNAT).

1. Ejecute la aplicación, para ellos en la conexión SSH que tiene con MobaXterm ejecute el siguiente comando

    ```shell
    fwbuilder
    ```

    > **Nota:** Se abre una nueva ventana del entorno gráfico de FWBuilder, no se debe cerrar o manipular la conexión SSH.

2. Ejecute las siguientes acciones en el entorno gráfico de FWBuilder
   1. Crear objetos de red:
        - NET-Hub-Front = 10.0.0.0/24
        - NET-Hub-Back = 10.0.1.0/24
        - NET-Spoke1 = 10.1.0.0/16
        - NET-Spoke2 = 10.2.0.0/16
   2. Crear el objeto firewall FW-Linux (plataforma iptables) y agregar interfaces con nombres reales (eth0, eth1).
   3. Reglas recomendadas en FWBuilder (Policy + NAT)
      - Policy (filtro):
        1) Allow Established, Related (Forward)
        2) Spokes → Internet (Forward)
           1) Source: NET-Spoke1, NET-Spoke2
           2) Destination: Any
           3) Services (lab): DNS, HTTP, HTTPS, ICMP
           4) Inbound iface: `Front`
           5) Outbound iface: `Back`
        3) Spoke1 → Spoke2 (Forward) (ICMP/SSH según pruebas)
        4) Spoke2 → Spoke1 (Forward)
        5) SSH de administración hacia el firewall (Input)
           1) Source: Any
           2) Destination: Any
           3) Services (lab): SSH
           4) Inbound iface: Any
           5) Outbound iface: Any
      - NAT (Source NAT / Masquerade):
        - Original Src: NET-Spoke1, NET-Spoke2
        - Original Dst: Any
        - Translated Src: Interfaz Back (egress) para aplicar masquerade.
      > **NOTA:** Valide las reglas de firewall indicadas en el desarrollo del laboratorio
  
   4. Compilar y desplegar el script .fw usando la interfaz gráfica. Verificar la ruta de almacenamiento del archivo.
   5. Cargar el archivo, En la conexión SSH de la máquina ejecute los siguientes comandos:

      ```shell
      sudo mkdir -p /etc/firewall
      sudo cp <RUTA_ARCHIVO>/FW-Linux.fw /etc/firewall/FW-Linux.fw
      sudo chmod +x /etc/firewall/FW-Linux.fw
      
      # Ejecutar
      sudo /etc/firewall/FW-Linux.fw
      ```

   6. Verificación rápida (iptables):

En la conexión SSH de la máquina ejecute los siguientes comandos
      ```shell
      sudo iptables -S
      sudo iptables -t nat -S
      ```

> **NOTA:** Si usted tiene dudas del uso de FWBuilder, puede consultar los siguientes enlaces o realice una busqueda en Internet para encontrar la guía que mas se acomode a su experiencia.
>
> - [Aprendiendo Ciberseguridad - Charla 135](https://youtu.be/-JBP5CYNZCU?si=yzWVg4z_EQG7qeEj)
> - [Aprendiendo Ciberseguridad - Charla 136](https://youtu.be/f3oHKzg2OKE?si=SACfLKpmidgDI9GU)
> - [Aprendiendo Ciberseguridad - Charla 137](https://youtu.be/L9Go0shlnd4?si=VH5_vWoY2oWF4boX)
> - [Aprendiendo Ciberseguridad - Charla 138](https://youtu.be/tB2m-wnxJZI?si=3W4tfc45kdxT728r)
> - [Getting Started With Firewall Builder](https://www.howtoforge.com/getting-started-with-firewall-builder)
> - [Enredando Con Redes - FWBuilder](https://enredandoconredes.com/2013/12/16/fwbuilder/)

- Checklist rápido del firewall

Al finalizar toda la configuración del firewall ejecute los siguientes comandos en la conexión SSH para validar la presencia de reglas

```shell
# 1) forwarding activo
sysctl net.ipv4.ip_forward

# 2) reglas nftables cargadas
sudo iptables -S
sudo iptables -t nat -S

```

#### Paso 8 – Enrutamiento (UDR) para forzar tráfico a través del Firewall

- Crear tabla de rutas por Spoke

Se recomienda una tabla por spoke (mejor aislamiento/operación).

En el la consola CLI en el portal de Azure

```shell
az network route-table create \
  --name RT-Spoke1 \
  --resource-group RG-Networking

az network route-table create \
  --name RT-Spoke2 \
  --resource-group RG-Networking
```

- Rutas para Spoke1

(a) Internet por Firewall:

```shell
az network route-table route create \
  --resource-group RG-Networking \
  --route-table-name RT-Spoke1 \
  --name DefaultToFirewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_FRONT_IP
```

(b) Spoke2 por Firewall (Spoke-to-Spoke):

```shell
az network route-table route create \
  --resource-group RG-Networking \
  --route-table-name RT-Spoke1 \
  --name ToSpoke2 \
  --address-prefix 10.2.0.0/16 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FW_FRONT_IP
```

- Rutas para Spoke2

Configuración de rutas

```shell
az network route-table route create \
  --resource-group RG-Networking \
  --route-table-name RT-Spoke2 \
  --name DefaultToFirewall \
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
```

- Asociar route tables a subredes WorkloadSubnet

```shell
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

### Validación avanzada (Network Watcher + Linux)

#### Validación funcional desde las VMs

En el la consola CLI en el portal de Azure

En VM-Spoke1:

```shell
# Ping a VM-Spoke2 (reemplaza por su IP privada)
ping -c 4 <IP_PRIVADA_VM_SPOKE2>

# Salida a internet (DNS + HTTP)
nslookup www.microsoft.com
curl -I https://www.microsoft.com
```

En VM-Spoke2:

```shell
ping -c 4 <IP_PRIVADA_VM_SPOKE1>
curl -I https://www.microsoft.com
```

#### Revisar “Effective routes” (rutas efectivas) en la NIC

Microsoft Learn recomienda revisar rutas efectivas para diagnosticar problemas: las rutas efectivas combinan rutas del sistema, UDRs y rutas propagadas.

En el la consola CLI en el portal de Azure (ejemplo sobre NIC de VM-Spoke1):

```shell
# Obtener NIC ID de VM-Spoke1
NIC1_ID=$(az vm show -g RG-Networking -n VM-Spoke1 --query "networkProfile.networkInterfaces[0].id" -o tsv)

# Ver rutas efectivas
az network nic show-effective-route-table --ids $NIC1_ID -o table
```

Verifica que exista:

- 0.0.0.0/0 con next hop VirtualAppliance (IP del firewall).
- Prefijo del otro Spoke (10.2.0.0/16) con next hop VirtualAppliance.

#### Network Watcher – Next hop (diagnóstico de enrutamiento)

Next hop te muestra el tipo de salto (Internet/VirtualAppliance/etc.), la IP y el route table id, útil para validar que una UDR está dirigiendo tráfico como esperas.
Ejemplo: En el la consola CLI en el portal de Azure se valida desde VM-Spoke1 hacia un IP destino en Internet:

```shell
az network watcher show-next-hop \
  --resource-group RG-Networking \
  --vm VM-Spoke1 \
  --nic 0 \
  --source-ip 10.1.0.4 \
  --dest-ip 13.107.21.200 \
  -o table
```

> *Nota:* oficial muestra cómo usar Next hop para detectar rutas personalizadas que causan problemas.

#### Network Watcher – Connection troubleshoot (diagnóstico de conectividad)

Usa Connection troubleshoot para validar:

- Bloqueos por NSG.
- Rutas/Udrs.
- Fallas DNS.

La herramienta ayuda a detectar problemas de rutas, NSG y otras causas de “unreachable”.

#### Validación en el firewall (captura y counters)

En FW-Linux:

```shell
# Ver sesiones NAT/conntrack (si está instalado)
sudo apt-get install -y conntrack
sudo conntrack -L | head

# Captura rápida (ajusta interfaces)
sudo tcpdump -ni any '(icmp or tcp port 443)' -c 50
```

### Resultados esperados

Al finalizar el laboratorio:

- Las VMs de Spoke1 y Spoke2 se comunican sin peering directo (Spoke-to-Spoke) y el tránsito pasa por FW-Linux.
- Las VMs de los Spokes salen a Internet a través del firewall Linux (NAT/masquerade).
- Las herramientas de Network Watcher confirman:
  - UDR aplicada (Effective routes).
  - Next hop = VirtualAppliance (cuando corresponda).
  - Conectividad “Reachable” con Connection troubleshoot.
