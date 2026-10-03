## 1. Topología

### pfSense (firewall)

Crear la VM via Virutal Manger, escoger OS FreeBDS 14.x. con 8GB de disco, 2GB de ram y 2 hilos. Todo siguiente durante la instalación. Esta finaliza cuando nos da escoger entre varias opciones numeradas.

### WAN

**Switch**: switch nativo de GNS3
**Atacante**: [docker](https://www.kali.org/docs/containers/using-kali-docker-images/) de kali linux con `nmap`, `nikto`, `curl`, podemos preparar un contenedor con estos paquetes con un `Dockerfile`:
```dockerfile
FROM kalilinux/kali-rolling
RUN apt update && apt install -y nmap nikto curl iproute2 iputils-ping && apt clean
```
Para crear el contenedor:
```shell
docker build -f <nombre_del_Dockerfile> -t <nombre_del_contenedor> .
```

### LAN

**Switch**: switch nativo de GNS3
**Clinete Web**: [docker](https://hub.docker.com/r/gns3/webterm) que inclueye navegador web

### DMZ (Opcional)

**Switch**: switch nativo de GNS3
**Servidor**: [docker](https://hub.docker.com/_/debian?tag=trixie) de debian con `nginx`, `openssh-server`, `curl`, podemos preparar un contenedor con estos paquetes con un `Dockerfile`:
```dockerfile
FROM debian:trixie-backports
ENV DEBIAN_FRONTEND=noninteractive
RUN apt update && apt install -y nginx openssh-server curl iproute2 iputils-ping && apt clean
# Inicia SSH en segundo plano y Nginx en primer plano para mantener vivo el contenedor
CMD service ssh start && nginx -g "daemon off;"
```
Para crear el contenedor:
```shell
docker build -f <nombre_del_Dockerfile> -t <nombre_del_contenedor> .
```

---

## 2. Configuración de los dispositivos

Asignación de interfaces, instalación de paquetes por CLI, optimización del kernel y configuración del motor de detección Suricata en un entorno virtualizado.

---

### 1. Creación del Template en GNS3

1. **Imagen base:** Utilizar el disco virtual base `.qcow2`.
2. En GNS3, ir a **Edit** > **Preferences** > **QEMU VMs** > **New**.
3. **Nombre:** `pfSense-2.7.2`.
4. **Recursos de hardware virtual:**
   * **RAM:** `2048 MB` (mínimo recomendado para compilación de reglas en memoria).
   * **vCPUs:** `2`.
5. **Configuración de adaptadores de red:**
   * Configurar un mínimo de `3` adaptadores tipo `Intel Gigabit Ethernet (e1000)` o `VirtIO`:
     - Adaptador 0 (`em0`): Interfaz **WAN**.
     - Adaptador 1 (`em1`): Interfaz **LAN**.
     - Adaptador 2 (`em2`): Interfaz **DMZ / OPT1** (para servidores web internos).
6. **Tipo de consola:** `telnet` o `VNC`.

> NOTA: telnet puede generar errores

---

### 2. Asignación de Interfaces y Conectividad Inicial (CLI)

Al iniciar pfSense por primera vez, el asistente de consola solicita la configuración de VLANs e interfaces físicas.

#### Asignación de puertos físicos
```text
Should VLANs be set up now [y|n]? n
Enter the WAN interface name: em0
Enter the LAN interface name: em1
Enter the Optional 1 interface name: em2 (o presionar Enter para omitir)
Do you want to proceed [y|n]? y
```

#### Conectividad temporal para descarga de paquetes
Para instalar Suricata y descargar sus firmas sin depender de una red interna con gateway:
1. Conectar temporalmente un nodo **NAT** de GNS3 a la interfaz WAN (`em0`) a través de un switch.
2. En el menú principal de pfSense, ingresar a la **opción 2) Set interface(s) IP address**:
   * Seleccionar interfaz `1` (WAN).
   * Activar DHCP IPv4: `y`.
   * Desactivar IPv6: `n`.
3. Validar conectividad a Internet desde la shell de pfSense:
   ```sh
   ping -c 8.8.8.8
   ```

---

### 3. Instalación de Suricata vía Shell (CLI)

En lugar de utilizar el gestor de paquetes de la GUI web, el paquete se puede instalar directamente a través del gestor de paquetes nativo de FreeBSD:

1. En el menú de consola de pfSense, seleccionar la **opción 8) Shell**.
2. Actualizar los repositorios e instalar el metapaquete oficial de Suricata:
   ```sh
   pkg update
   pkg install pfSense-pkg-suricata
   ```
3. Salir de la shell:
   ```sh
   exit
   ```
 
#### Ingreso a la GUI Web
Desde el navegador del **Cliente LAN** (`192.168.1.10`):
* URL: `https://192.168.1.1`
* Credenciales por defecto: Usuario `admin`, contraseña `pfsense`.

#### Habilitar y descargar firmas ETOpen
1. Ir a **Services** > **Suricata** > pestaña **Global Settings**.
2. Marcar la casilla **Install ETOpen Rules**.
3. *Rules Update Setting* -> Update Interval -> 12 HOURS
4. Guardar cambios (**Save**).
5. Ir a la pestaña **Updates** y hacer clic en **Update** (o **Force**).
6. Verificar que el resultado indique `Result: success` y las firmas queden registradas con sus hashes MD5.

---

### 4. Fijar Direccionamiento Estático del Laboratorio

Una vez descargado el paquete y listas las dependencias, se retira el nodo NAT de GNS3 y se fijan los direccionamientos definitivos de la topología:
* **pfSense WAN (`em0`):** `203.0.113.1/24`
* **pfSense LAN (`em1`):** `192.168.1.1/24`
* **Atacante:** `203.0.113.10/24` (conectado al switch WAN)
* **Cliente LAN:** `192.168.1.X/24` (conectado al switch LAN)

#### Configurar IP estática en la WAN por consola:
1. En el menú de pfSense, seleccionar **opción 2) Set interface(s) IP address**.
2. Seleccionar interfaz `1` (WAN).
3. Configurar IPv4 por DHCP: `n`.
4. Nueva dirección IPv4: `203.0.113.1`.
5. Máscara de subred (CIDR): `24`.
6. Upstream Gateway: Presionar **Enter** (dejar vacío).
7. IPv6: `n` y presionar **Enter**.
8. Revertir protocolo a HTTP: `n`.

---

### 5. Desactivación de Hardware Offloading (GUI)

#### Desactivar Hardware Offloading (Obligatorio para Suricata)
En entornos virtualizados, el offloading por hardware corrompe o trunca los paquetes analizados por el motor, impidiendo la coincidencia de firmas.

1. Ir a **System** > **Advanced** > pestaña **Networking**.
2. En la sección **Network Interfaces**, marcar:
   * [x] **Disable hardware checksum offload**
   * [x] **Disable hardware TCP segmentation offload** (TSO)
   * [x] **Disable hardware large receive offload** (LRO)
3. Guardar cambios con **Save**.
4. Reiniciar pfSense para aplicar los cambios en el kernel (**Diagnostics** > **Reboot** o consola opción **5**).

---

## 3. Configuración del Firewall y Reglas

### 1. Configuración de NAT (Port Forwarding)

Para publicar el servicio web de la DMZ hacia el exterior sin exponer directamente la red interna:

* **Ruta de configuración:** `Firewall → NAT → Port Forward → Add`
* **Parámetros configurados:**
  * **Interface:** `WAN`
  * **Address Family:** `IPv4`
  * **Protocol:** `TCP`
  * **Destination:** `WAN address`
  * **Destination Port Range:** `8080` a `8080` (Custom: 8080)
  * **Redirect Target IP:** `172.16.1.10`
  * **Redirect Target Port:** `80` (HTTP)
  * **Description:** `NAT WAN:8080 a DMZ Web:80`
  * **Filter rule association:** `Add associated filter rule` (crea la regla de paso en la interfaz WAN en forma automática).

---

### 2. Reglas de Filtrado por Interfaz

Las políticas siguen el principio de **Mínimo Privilegio** (*Least Privilege*) y la política base **Deny All** (bloqueo implícito al final de la evaluación). El orden de evaluación es estrictamente de arriba hacia abajo (primera coincidencia).

#### A. Interfaz WAN (`Firewall → Rules → WAN`)

| Orden |  Acción   |     Protocolo     | Origen | Puerto Orig. |    Destino    | Puerto Dest. | Log | Descripción                                                                            |
| :---: | :-------: | :---------------: | :----: | :----------: | :-----------: | :----------: | :-: | :------------------------------------------------------------------------------------- |
| **1** | **Pass**  | IPv4 ICMP *(any)* |  `*`   |     `*`      | `WAN address` |     `*`      | No  | *Permitir ping desde WAN para diagnóstico en laboratorio (deshabilitar tras pruebas).* |
| **2** | **Pass**  |     IPv4 TCP      |  `*`   |     `*`      | `172.16.1.10` | `80 (HTTP)`  | No  | *Regla asociada al Port Forwarding desde el puerto WAN 8080.*                          |
| **-** | **Block** |        `*`        |  `*`   |     `*`      |      `*`      |     `*`      | Sí  | *Regla implícita por defecto (Deny All).*                                              |

#### B. Interfaz LAN (`Firewall → Rules → LAN`)

Se retiraron/deshabilitaron las reglas permisivas por defecto (`Default allow LAN to any rule` IPv4/IPv6) y se aplicaron accesos específicos:

| Orden |  Acción   |  Protocolo   |    Origen     | Puerto Orig. |    Destino    | Puerto Dest.  |  Log   | Descripción                                                          |
| :---: | :-------: | :----------: | :-----------: | :----------: | :-----------: | :-----------: | :----: | :------------------------------------------------------------------- |
| **1** | **Pass**  |     `*`      |      `*`      |     `*`      | `LAN Address` | `443, 80, 22` |   No   | *Anti-Lockout Rule (evita perder acceso administrativo al pfSense).* |
| **2** | **Pass**  |   IPv4 TCP   | `LAN subnets` |     `*`      | `172.16.1.10` |  `80 (HTTP)`  |   No   | *Acceso web al servidor de la DMZ.*                                  |
| **3** | **Pass**  |   IPv4 TCP   | `LAN subnets` |     `*`      | `172.16.1.10` |  `22 (SSH)`   |   No   | *Acceso SSH administrativo al servidor DMZ.*                         |
| **4** | **Block** |   IPv4 `*`   | `LAN subnets` |     `*`      | `DMZ subnets` |      `*`      | **Sí** | *Bloquear el resto de tráfico no autorizado hacia la DMZ.*           |
| **5** | **Pass**  |   IPv4 TCP   | `LAN subnets` |     `*`      |      `*`      |  `80 - 443`   |   No   | *Salida web LAN a Internet (HTTP/HTTPS).*                            |
| **6** | **Pass**  | IPv4 TCP/UDP | `LAN subnets` |     `*`      |      `*`      |  `53 (DNS)`   |   No   | *Consultas DNS de clientes LAN a Internet/pfSense.*                  |
| **-** | **Block** |     `*`      |      `*`      |     `*`      |      `*`      |      `*`      |   Sí   | *Regla implícita por defecto (Deny All).*                            |

*Nota:* Las reglas originales `Default allow LAN to any rule` (IPv4 e IPv6) quedan deshabilitadas (icono en gris).

#### C. Interfaz DMZ (`Firewall → Rules → DMZ`)

Garantiza el aislamiento de la zona semiconfiable para evitar movimiento lateral hacia la LAN en caso de que el servidor web sea vulnerado:

| Orden |  Acción   |     Protocolo     |    Origen     | Puerto Orig. |    Destino    | Puerto Dest.  |  Log   | Descripción                                                        |
| :---: | :-------: | :---------------: | :-----------: | :----------: | :-----------: | :-----------: | :----: | :----------------------------------------------------------------- |
| **1** | **Pass**  | IPv4 ICMP *(any)* | `DMZ subnets` |     `*`      | `DMZ address` |      `*`      |   No   | *Permitir ping de diagnóstico a la gateway (172.16.1.1).*          |
| **2** | **Block** |     IPv4 `*`      | `DMZ subnets` |     `*`      | `LAN subnets` |      `*`      | **Sí** | *Aislamiento estricto: evita movimiento lateral de DMZ hacia LAN.* |
| **3** | **Pass**  |   IPv4 TCP/UDP    | `DMZ subnets` |     `*`      |      `*`      |  `53 (DNS)`   |   No   | *Resolución de nombres para el servidor (actualizaciones).*        |
| **4** | **Pass**  |   IPv4 TCP/UDP    | `DMZ subnets` |     `*`      |      `*`      |  `80 (HTTP)`  |   No   | *Salida HTTP a repositorios de software.*                          |
| **5** | **Pass**  |   IPv4 TCP/UDP    | `DMZ subnets` |     `*`      |      `*`      | `443 (HTTPS)` |   No   | *Salida HTTPS a repositorios de software.*                         |
| **-** | **Block** |        `*`        |      `*`      |     `*`      |      `*`      |      `*`      |   Sí   | *Regla implícita por defecto (Deny All).*                          |

---

### 3. Matriz de Flujos y Verificación de Conectividad

| Origen           | Destino          | Puerto / Servicio | Resultado Esperado                         | Prueba Realizada                  |
| :--------------- | :--------------- | :---------------- | :----------------------------------------- | :-------------------------------- |
| `lab-atacante-1` | `203.0.113.1`    | TCP 8080          | **PERMITIDO** (traducido a 172.16.1.10:80) | `curl http://203.0.113.1:8080`    |
| `lab-atacante-1` | `203.0.113.1`    | TCP 22 / Otros    | **BLOQUEADO** (Deny All implícito)         | `nmap -p 22 203.0.113.1`          |
| `gns3-webterm-1` | `172.16.1.10`    | TCP 80, 22        | **PERMITIDO**                              | Navegador HTTP y conexión SSH     |
| `gns3-webterm-1` | `172.16.1.10`    | ICMP (Ping)       | **BLOQUEADO** (Regla 4 LAN)                | `ping 172.16.1.10`                |
| `lab-servidor-1` | `192.168.1.0/24` | Any               | **BLOQUEADO** (Regla 2 DMZ)                | `ping 192.168.1.1` (log generado) |
| `lab-servidor-1` | `172.16.1.1`     | ICMP (Ping)       | **PERMITIDO** (Regla 1 DMZ)                | `ping 172.16.1.1`                 |

--- 

## 4. Configuración de Suricata y Reglas

### Creación de la Interfaz de Monitoreo

1. En **Services** > **Suricata**, ingresar a la pestaña **Interfaces** y pulsar **+ Add**.
2. **General Settings:**
   * **Interface:** `WAN` (`em0`).
   * **Description:** `WAN Interface`.
   * **Send Alerts to System Log:** Marcado.
   * **Block Offenders:** Desmarcado inicialmente.
1. Guardar cambios con **Save**.

---

### Selección de Categorías de Reglas

1. Editar la interfaz WAN (icono de lápiz) y abrir la pestaña **WAN Categories**.
2. Seleccionar las categorías requeridas para el laboratorio:
   * `emerging-scan.rules`
   * `emerging-web_server.rules`
   * `emerging-web_specific_apps.rules`
   * `emerging-exploit.rules`
   * `emerging-dos.rules`
1. Guardar cambios con **Save**.

---

### Reglas Personalizadas

1. En la configuración de la interfaz WAN, ir a la pestaña **WAN Rules**.
2. En el menú desplegable **Category**, seleccionar `custom.rules`.
3. En el cuadro de texto inferior, insertar la regla personalizada para escaneo rápido SYN:
```text
alert tcp any any -> $HOME_NET any (msg:"LAB Nmap SYN scan rapido"; flags:S; threshold:type both, track by_src, count 20, seconds 3; classtype:attempted-recon; sid:1000001; rev:1;)
```
4. Guardar cambios con **Save**.

---

### Verificación de Pass List e Inicio del Servicio

1. En **Services** > **Suricata** > pestaña **Pass Lists**, verificar que la lista contenga las direcciones locales (LAN y WAN) para evitar auto-bloqueos.
2. En **Services** > **Suricata** > pestaña **Interfaces**, verificar que la interfaz WAN tenga asignada la Pass List en su configuración.
3. En la tabla de interfaces, pulsar el botón verde **Start** (icono de Play) en la fila de la interfaz WAN.
4. Confirmar que el estado cambie a **Running** (icono de visto verde).

## 5. Pruebas del Firewall 

### Primera prueba

```shell
nmap -p 22 203.0.113.1
```

![](attachments/Pasted%20image%2020261003160023.png)

![](attachments/Pasted%20image%2020261003160042.png)

### Segunda Prueba

```shell
ping 192.168.1.1
```

![](attachments/Pasted%20image%2020261003160349.png)

![](attachments/Pasted%20image%2020261003160436.png)

## 6. Pruebas de Suricata 

### Primera prueba

```shell
nmap -sS -p- -T4 203.0.113.1
```

![](attachments/Pasted%20image%2020261003141345.png)

Interfaz WAN
![](attachments/Pasted%20image%2020261003141603.png)

### Segunda prueba

```shell
nmap -sV -p 8080 --script http-enum 203.0.113.1
```

![](attachments/Pasted%20image%2020261003143806.png)

Interfaz WAN
![](attachments/Pasted%20image%2020261003143829.png)

Interfaz DMZ
![](attachments/Pasted%20image%2020261003143853.png)

### Tercera prueba

```shell
nikto -h http://203.0.113.1:8080
```

![](attachments/Pasted%20image%2020261003142615.png)

Interfaz DMZ
![](attachments/Pasted%20image%2020261003142729.png)

Interfaz WAN
![](attachments/Pasted%20image%2020261003142825.png)

### Cuarta prueba

```shell
# Simulación de herramienta SQLMap y payload SQLi
curl -A "sqlmap/1.0" "http://203.0.113.1:8080/?id=1'%20OR%20'1'='1"
# Intento de Path/Directory Traversal
curl "http://203.0.113.1:8080/../../etc/passwd"
```

![](attachments/Pasted%20image%2020261003143516.png)

Interfaz DMZ
![](attachments/Pasted%20image%2020261003143528.png)

Interfaz WAN
![](attachments/Pasted%20image%2020261003143624.png)


## Modo IPS

Al enviar cualquier ataque de los antriormente probados desde la WAN el modo IPS bloque la IP del atacante.

```shell
nmap -sS -p- -T4 203.0.113.1
```

![](attachments/Pasted%20image%2020261003182854.png)

![](attachments/Pasted%20image%2020261003182906.png)

```shell
curl -A "sqlmap/1.0" "http://203.0.113.1:8080/?id=1'%20OR%20'1'='1"
```

![](attachments/Pasted%20image%2020261003182614.png)

![](attachments/Pasted%20image%2020261003182644.png)

## Conclusiones

- El firewall aplica eficazmente el principio de mínimo privilegio descartando conexiones a puertos no autorizados, pero es incapaz por sí solo de inspeccionar el contenido de los paquetes. En el puerto `8080` (publicado por port forwarding), el firewall valida que el handshake TCP sea legítimo y lo redirige al servidor DMZ. Es exclusivamente Suricata el que identifica que dentro de esa sesión TCP permitida viajan cadenas de inyección SQL, directory traversal (`../../etc/passwd`) o agentes de escaneo maliciosos (`sqlmap`, `nikto`).

- Desplegar una instancia en WAN y otra en DMZ proporciona una correlación completa ya que en WAN, se captura el intento de reconocimiento temprano y los escaneos de puertos contra la IP perimetral (`203.0.113.1`) antes de que intervenga la traducción de direcciones y en DMZ, se audita el tráfico que efectivamente cruzó el perímetro hacia la IP interna del servidor (`172.16.1.10`), confirmando si una amenaza web alcanzó su objetivo.

- El aislamiento estricto de la zona DMZ (`Block DMZ net -> LAN net`) reduce drásticamente el radio de impacto de una brecha. Aun en el escenario hipotético de que el atacante logre comprometer por completo el contenedor web explotando una falla web, el firewall impide que pivote hacia la red confiable (`192.168.1.0/24`) para acceder a la administración del pfSense o a los equipos de los usuarios.

- Durante las pruebas en modo IDS, el sistema generó alertas detalladas en los logs, pero el servidor Nginx continuó respondiendo a cada una de las solicitudes de Nikto y cURL. En un entorno de producción, un IDS pasivo únicamente notifica el incidente a posteriori; sin intervención manual o una transición a IPS (Inline / Block Offenders), el atacante continúa recopilando información y explotando vulnerabilidades sin fricción. La transición a IPS con Block Offenders convirtió la defensa en una acción proactiva, Suricata coordinó dinámicamente con pf para registrar la IP hostil (203.0.113.10) en tablas de descarte temporal, cortando sesiones TCP y forzando timeouts en herramientas automatizadas. No obstante, el bloqueo por IP en modo Legacy conlleva el riesgo de generar falsos positivos que afecten a usuarios legítimos. Por ello, su viabilidad en producción exige un afinamiento riguroso de firmas y el uso estricto de Pass Lists para proteger la operatividad y la administración.

- La limitación a categorías esenciales de ET Open (`scan`, `web_server`, etc.) y el diseño de la regla personalizada para escaneo SYN demostraron que es viable obtener alta visibilidad de amenazas sin agotar la memoria RAM (1 GB en QEMU) ni degradar el enrutamiento. Además, la creación previa de la Pass List resulta crítica antes de activar el bloqueo preventivo, asegurando que un análisis de reconocimiento no aísle la administración legítima.

## Notas

pfSense bloquea por defecto los paquetes ICMP de la red WAN

Empezar con:

![](attachments/Pasted%20image%2020261003004500.png)

por default se configurar WAN y LAN como DHCP y server DHCP.
