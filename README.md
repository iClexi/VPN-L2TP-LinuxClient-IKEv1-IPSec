# VPN L2TP/IPSec IKEv1 Client-to-Site con Cliente Linux

<p align="center">
  <img src="https://img.shields.io/badge/Plataforma-GNS3-blue?style=for-the-badge" />
  <img src="https://img.shields.io/badge/VPN-L2TP%2FIPSec-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/IKE-IKEv1-red?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Cliente-Kali%20Linux-557C94?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Modelo-Client--to--Site-success?style=for-the-badge" />
</p>

<p align="center">
  <b>VPN Client-to-Site punto a multipunto usando L2TP sobre IPSec con IKEv1</b>
</p>

---

## Datos del proyecto

| Campo | Detalle |
|---|---|
| Autor | Michael David Robles Fermín |
| Matrícula | 2025-0845 |
| Asignatura | Seguridad de Redes |
| Práctica | VPN Client-to-Site L2TP/IPSec IKEv1 |
| Servidor VPN | R1-L2TP-SERVER |
| Cliente VPN | Kali Linux |
| Repositorio | https://github.com/iClexi/VPN-L2TP-LinuxClient-IKEv1-IPSec |
| Video | https://youtu.be/Fu7Mby9g2_E |

---

## Documentación técnica

La documentación técnica profesional está en:

[`docs/Documentacion Tecnica Profesional.pdf`](docs/Documentacion%20Tecnica%20Profesional.pdf)

También se incluye la versión editable:

[`docs/Documentacion Tecnica Profesional.docx`](docs/Documentacion%20Tecnica%20Profesional.docx)

---

## Descripción general

Esta práctica implementa una VPN **Client-to-Site punto a multipunto** utilizando **L2TP sobre IPSec con IKEv1**. El router R1 funciona como servidor VPN y Kali Linux funciona como cliente remoto. El objetivo es que Kali, estando fuera de la red interna, pueda autenticarse por VPN y acceder a la LAN `192.168.45.0/24`.

L2TP crea el túnel lógico del cliente remoto, PPP autentica el usuario y crea la interfaz `ppp0`, mientras que IPSec protege el tráfico L2TP mediante IKEv1.

---

## Topología

<p align="center">
  <img src="images/01_topologia_completa.png" alt="Topología VPN L2TP IPSec IKEv1" width="900">
</p>

La topología usa a Kali como cliente externo, el ISP como red pública simulada, R1 como servidor VPN y una PC interna detrás del switch SW1. La prueba final consiste en que Kali llegue a `192.168.45.10`, que está dentro de la LAN privada.

### Conexiones principales

| Desde | Interfaz | Hacia | Interfaz |
|---|---:|---|---:|
| Kali Linux | eth0 | ISP | Gi0/1 |
| ISP | Gi0/0 | R1-L2TP-SERVER | Gi0/0 |
| R1-L2TP-SERVER | Gi0/1 | SW1 | Gi0/0 |
| SW1 | Gi0/1 | PC1 | eth0 |

---

## Direccionamiento

| Equipo | Interfaz | IP | Función |
|---|---:|---:|---|
| Kali Linux | eth0 | 20.25.8.50/30 | Cliente externo |
| ISP | Gi0/1 | 20.25.8.49/30 | Gateway de Kali |
| ISP | Gi0/0 | 20.25.8.45/30 | Gateway WAN de R1 |
| R1-L2TP-SERVER | Gi0/0 | 20.25.8.46/30 | WAN del servidor VPN |
| R1-L2TP-SERVER | Gi0/1 | 192.168.45.1/24 | Gateway de la LAN interna |
| PC1 | eth0 | 192.168.45.10/24 | Host interno |
| Kali Linux | ppp0 | 192.168.84.101 | IP recibida por VPN |

---

## Parámetros usados

| Parámetro | Valor |
|---|---|
| Tipo de VPN | Client-to-Site |
| Modelo | Punto a multipunto |
| Protocolo de túnel | L2TP |
| Protección | IPSec |
| Negociación | IKEv1 |
| Autenticación IPSec | Pre-shared key |
| PSK | ITLA20250845 |
| Usuario VPN | michael |
| Contraseña VPN | L2TP20250845 |
| Pool VPN | 192.168.84.100 - 192.168.84.150 |
| IP recibida por Kali | 192.168.84.101 |
| Transform-set | L2TP-3DES |
| Modo IPSec | Transport |
| Crypto map | L2TP-MAP |
| Dynamic map | L2TP-DYNAMIC |

---

## Funcionamiento de la VPN

R1 actúa como concentrador VPN. Primero autentica al cliente mediante IKEv1 e IPSec usando la clave precompartida `ITLA20250845`. Después L2TP crea el túnel lógico y PPP autentica al usuario `michael`. Al conectarse, Kali recibe una IP del pool `192.168.84.100 - 192.168.84.150` y se crea la interfaz `ppp0`.

La práctica es punto a multipunto porque R1 no está limitado a un solo cliente. El servidor usa un pool de direcciones, VPDN, Virtual-Template y dynamic crypto map, permitiendo aceptar múltiples clientes remotos autorizados.

---

## Configuración principal en R1

La configuración completa está en [`configs/R1-L2TP-SERVER.cfg`](configs/R1-L2TP-SERVER.cfg). La parte más importante del servidor VPN incluye AAA, VPDN, Virtual-Template, IKEv1, IPSec y el crypto map aplicado en la WAN.

```cisco
aaa new-model

aaa authentication ppp L2TP-AUTH local
aaa authorization network L2TP-AUTH local
username michael password 0 L2TP20250845

ip local pool L2TP-POOL 192.168.84.100 192.168.84.150

vpdn enable
vpdn-group L2TP-GROUP
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

interface Virtual-Template1
 description CLIENTES-L2TP-IPSEC
 ip unnumbered GigabitEthernet0/1
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2 L2TP-AUTH
 ppp ipcp dns 8.8.8.8
 ip tcp adjust-mss 1360

crypto isakmp policy 10
 encr 3des
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key ITLA20250845 address 0.0.0.0 0.0.0.0
crypto isakmp nat keepalive 20

crypto ipsec transform-set L2TP-3DES esp-3des esp-sha-hmac
 mode transport

crypto dynamic-map L2TP-DYNAMIC 10
 set transform-set L2TP-3DES
 set security-association lifetime seconds 3600

crypto map L2TP-MAP 10 ipsec-isakmp dynamic L2TP-DYNAMIC

interface GigabitEthernet0/0
 crypto map L2TP-MAP
```

AAA y el usuario local permiten autenticar al cliente L2TP mediante PPP. El pool entrega direcciones a los clientes remotos. VPDN habilita L2TP, y la Virtual-Template crea dinámicamente interfaces Virtual-Access para cada cliente conectado.

---

## Configuración principal en Kali Linux

La configuración completa está en [`configs/KALI-LINUX-CLIENT.cfg`](configs/KALI-LINUX-CLIENT.cfg). En Kali se usan tres componentes: strongSwan para IPSec/IKEv1, xl2tpd para L2TP y PPP para la autenticación del usuario.

```bash
sudo ipsec up L2TP-IPSEC-R1
sudo systemctl restart xl2tpd
echo "c R1-L2TP" | sudo tee /var/run/xl2tpd/l2tp-control
ip addr show ppp0
sudo ip route replace 192.168.45.0/24 dev ppp0
ping -c 4 192.168.45.10
```

Cuando la conexión sube, Kali recibe una interfaz `ppp0` con IP del pool VPN. Esa interfaz se usa para llegar a la red interna `192.168.45.0/24`.

---

## Verificación técnica integrada

### Estado general de R1

<p align="center">
  <img src="images/02_r1_show_ip_interface_brief.png" alt="show ip interface brief en R1" width="900">
</p>

El comando `show ip interface brief` confirma que R1 tiene activas la WAN `20.25.8.46`, la LAN `192.168.45.1` y las interfaces Virtual-Access creadas por la conexión L2TP.

### Negociación IKEv1

<p align="center">
  <img src="images/03_r1_show_crypto_isakmp_sa.png" alt="show crypto isakmp sa en R1" width="900">
</p>

La salida muestra el estado `QM_IDLE`, lo que confirma que IKEv1 negoció correctamente entre Kali y R1.

### Protección IPSec

<p align="center">
  <img src="images/04_r1_show_crypto_ipsec_sa_parte1.png" alt="show crypto ipsec sa parte 1" width="900">
</p>

La salida muestra paquetes encapsulados y desencapsulados. Esto demuestra que IPSec está protegiendo tráfico entre `20.25.8.46` y `20.25.8.50`.

<p align="center">
  <img src="images/05_r1_show_crypto_ipsec_sa_parte2.png" alt="show crypto ipsec sa parte 2" width="900">
</p>

Aquí se observan asociaciones IPSec entrantes y salientes en estado activo, usando ESP con 3DES y SHA-HMAC en modo transport.

<p align="center">
  <img src="images/06_r1_show_crypto_ipsec_sa_parte3.png" alt="show crypto ipsec sa parte 3" width="900">
</p>

Esta continuación confirma que existen SAs activas para proteger el tráfico L2TP sobre IPSec.

### Túnel y sesión L2TP

<p align="center">
  <img src="images/07_r1_show_vpdn_tunnel.png" alt="show vpdn tunnel en R1" width="900">
</p>

`show vpdn tunnel` muestra el túnel L2TP establecido con el cliente Kali desde `20.25.8.50`.

<p align="center">
  <img src="images/08_r1_show_vpdn_session.png" alt="show vpdn session en R1" width="900">
</p>

`show vpdn session` confirma la sesión L2TP activa del usuario `michael`, asociada a una interfaz Virtual-Access.

### Ruta y conectividad desde Kali

<p align="center">
  <img src="images/09_kali_ip_route.png" alt="ip route en Kali" width="900">
</p>

La tabla de rutas de Kali muestra la ruta hacia `192.168.45.0/24` por `ppp0`. Esto significa que el tráfico hacia la LAN interna viaja por la VPN.

<p align="center">
  <img src="images/10_kali_ping_pc_lan.png" alt="ping desde Kali hacia PC LAN" width="900">
</p>

El ping desde Kali hacia `192.168.45.10` confirma que el cliente remoto accede correctamente a la PC interna mediante L2TP/IPSec.

---

## Comandos de verificación

En R1:

```cisco
show crypto isakmp sa
show crypto ipsec sa
show vpdn tunnel
show vpdn session
show ip interface brief
```

En Kali:

```bash
sudo ipsec statusall
ip addr show ppp0
ip route
ping -c 4 192.168.45.1
ping -c 4 192.168.45.10
```

---

## Archivos de configuración

Las configuraciones completas están en la carpeta [`configs/`](configs/) con extensión `.cfg`.

| Archivo | Descripción |
|---|---|
| [`configs/R1-L2TP-SERVER.cfg`](configs/R1-L2TP-SERVER.cfg) | Configuración completa del servidor VPN |
| [`configs/ISP.cfg`](configs/ISP.cfg) | Configuración del router ISP |
| [`configs/SW1.cfg`](configs/SW1.cfg) | Configuración del switch LAN |
| [`configs/PC1-LAN.cfg`](configs/PC1-LAN.cfg) | Configuración de la PC interna |
| [`configs/KALI-LINUX-CLIENT.cfg`](configs/KALI-LINUX-CLIENT.cfg) | Configuración del cliente Kali Linux |

---

## Estructura del repositorio

```text
VPN-L2TP-LinuxClient-IKEv1-IPSec/
|
|-- README.md
|-- MichaelRobles_2025-0845_Links_P1.txt
|
|-- configs/
|   |-- ISP.cfg
|   |-- KALI-LINUX-CLIENT.cfg
|   |-- PC1-LAN.cfg
|   |-- R1-L2TP-SERVER.cfg
|   |-- SW1.cfg
|
|-- docs/
|   |-- Documentacion Tecnica Profesional.pdf
|   |-- Documentacion Tecnica Profesional.docx
|
|-- images/
    |-- 01_topologia_completa.png
    |-- 02_r1_show_ip_interface_brief.png
    |-- 03_r1_show_crypto_isakmp_sa.png
    |-- 04_r1_show_crypto_ipsec_sa_parte1.png
    |-- 05_r1_show_crypto_ipsec_sa_parte2.png
    |-- 06_r1_show_crypto_ipsec_sa_parte3.png
    |-- 07_r1_show_vpdn_tunnel.png
    |-- 08_r1_show_vpdn_session.png
    |-- 09_kali_ip_route.png
    |-- 10_kali_ping_pc_lan.png
```

---

## Conclusión

La práctica demuestra que un cliente Linux externo puede conectarse de forma segura a una LAN privada mediante una VPN **L2TP sobre IPSec con IKEv1**. R1 funciona como servidor VPN, autentica al usuario, asigna una IP desde el pool, crea la sesión L2TP y permite que Kali acceda a la red interna `192.168.45.0/24`.

Aunque se muestra un solo cliente en la demostración, la configuración es punto a multipunto porque el servidor usa un pool de direcciones, VPDN, Virtual-Template y dynamic crypto map para aceptar múltiples clientes remotos autorizados.
