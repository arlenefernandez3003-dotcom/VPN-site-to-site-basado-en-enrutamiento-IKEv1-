# 🔐 IPSec IKEv1 — VPN Site-to-Site Basada en Enrutamiento

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![IPSec](https://img.shields.io/badge/Protocolo-IPSec%20IKEv1-blue?style=for-the-badge&logo=cisco)
![Type](https://img.shields.io/badge/Tipo-Site--to--Site-brightgreen?style=for-the-badge)
![Mode](https://img.shields.io/badge/Modo-Route--Based-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Plataforma-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Verificación del Túnel](#6-verificación-del-túnel)
7. [Capturas de Pantalla](#7-capturas-de-pantalla)
8. [Video Demostrativo](#8-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Site-to-Site basada en enrutamiento (route-based)** utilizando **IPSec con IKEv1** en routers Cisco IOS dentro de PNetLab. La práctica cubre:

- Negociación del canal seguro IKEv1 en dos fases (ISAKMP y Quick Mode), idéntico proceso criptográfico que en el modelo basado en políticas.
- Creación de una **interfaz de túnel virtual (Virtual-Tunnel Interface)** que encapsula automáticamente todo el tráfico IP enrutado hacia ella, sin depender de una ACL de "tráfico interesante".
- Protección del tráfico entre `202.50.73.128/25` (Site A) y `202.50.73.0/25` (Site B) a través del segmento `192.168.19.0/24` simulado por la nube NAT de PNetLab.
- Validación del túnel mediante comandos `show`, estado de la interfaz Tunnel y pruebas de conectividad.

### ¿Por qué Route-Based?

A diferencia del modelo policy-based, aquí el tráfico a cifrar se determina por la **tabla de enrutamiento**: cualquier paquete con ruta hacia la interfaz `Tunnel0` se cifra automáticamente. Esto permite usar protocolos de enrutamiento dinámico sobre el túnel y escalar con mayor facilidad a topologías más complejas.

```
PC1 → R1 → [Ruta hacia 202.50.73.0/25 apunta a Tunnel0] → Cifra → ISP → R2 → Descifra → PC2
```

---

## 2. Topología

```
                              [ INTERNET / ISP ]
                               192.168.19.0/24
                              (Cloud/NAT: 192.168.19.2)
                                      │
                  ┌───────────────────┴───────────────────┐
                  │ e0/0: 192.168.19.5                     │ e0/0: 192.168.19.6
          ┌───────┴────────┐                      ┌────────┴───────┐
          │   R1 (Peer A)  │◄══ Tunnel0 (VTI) ════►  R2 (Peer B)  │
          │                │   IPSec IKEv1         │               │
          └───────┬────────┘    Route-Based        └────────┬──────┘
                  │ e0/1: 202.50.73.129/25                  │ e0/1: 202.50.73.1/25
                  │                                         │
          ┌───────┴────────┐                      ┌─────────┴──────┐
          │     SW1        │                      │      SW2       │
          └───────┬────────┘                      └────────┬───────┘
                  │                                        │
          ┌───────┴────────┐                      ┌────────┴───────┐
          │      PC1       │                      │      PC2       │
          │ 202.50.73.130  │                      │  202.50.73.2   │
          └────────────────┘                      └────────────────┘
           ◄── SITE A ──►                          ◄── SITE B ──►
           202.50.73.128/25                        202.50.73.0/25
```

> El túnel lógico `Tunnel0` se levanta entre `192.168.19.5` (R1) y `192.168.19.6` (R2), usando una red interna `/30` (`10.0.0.0/30`) como direccionamiento del túnel.

**Flujo de establecimiento:**
1. R1 tiene una ruta estática hacia `202.50.73.0/25` apuntando a `Tunnel0`.
2. Al recibir tráfico de PC1 hacia PC2, R1 lo encamina por `Tunnel0`.
3. El `tunnel protection ipsec profile` cifra el paquete automáticamente con IKEv1 + IPSec.
4. El paquete cifrado viaja por el segmento ISP hasta R2, que lo descifra y entrega a PC2.

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo | Interfaz   | Dirección IP        | Máscara | Gateway        | Rol                          |
|-------------|------------|----------------------|---------|-----------------|-------------------------------|
| Cloud (NAT) | —          | 192.168.19.2         | /24     | —               | Gateway Cloud NAT (PNetLab)   |
| **R1**      | **e0/0**   | **192.168.19.5**     | **/24** | 192.168.19.2    | WAN Peer A → Cloud            |
| **R1**      | **e0/1**   | **202.50.73.129**    | **/25** | —               | Gateway LAN Site A            |
| **R1**      | **Tunnel0**| **10.0.0.1**         | **/30** | —               | Túnel VTI hacia R2            |
| **R2**      | **e0/0**   | **192.168.19.6**     | **/24** | 192.168.19.2    | WAN Peer B → Cloud            |
| **R2**      | **e0/1**   | **202.50.73.1**      | **/25** | —               | Gateway LAN Site B            |
| **R2**      | **Tunnel0**| **10.0.0.2**         | **/30** | —               | Túnel VTI hacia R1            |
| SW1         | —          | —                    | —       | —               | Capa 2 Site A                 |
| SW2         | —          | —                    | —       | —               | Capa 2 Site B                 |
| PC1         | eth0       | 202.50.73.130        | /25     | 202.50.73.129   | Host Site A                   |
| PC2         | eth0       | 202.50.73.2          | /25     | 202.50.73.1     | Host Site B                   |

### Tabla de Subredes

| Subred              | Rango Utilizable               | Broadcast        | Uso                  |
|---------------------|----------------------------------|------------------|-----------------------|
| `192.168.19.0/24`   | 192.168.19.1 – 192.168.19.254   | 192.168.19.255   | Segmento WAN/Cloud    |
| `202.50.73.0/25`    | 202.50.73.1 – 202.50.73.126     | 202.50.73.127    | LAN Site B            |
| `202.50.73.128/25`  | 202.50.73.129 – 202.50.73.254   | 202.50.73.255    | LAN Site A            |
| `10.0.0.0/30`       | 10.0.0.1 – 10.0.0.2             | 10.0.0.3         | Interfaz Tunnel (VTI) |

---

## 4. Parámetros Configurados

### IKE Fase 1 — ISAKMP Policy

| Parámetro           | Valor               | Descripción                                             |
|----------------------|---------------------|-----------------------------------------------------------|
| Número de política  | `10`                | Prioridad de la política ISAKMP                          |
| Cifrado             | AES-256             | Cifrado simétrico del canal IKE                          |
| Hash / Integridad   | SHA-256             | Verificación de integridad de mensajes IKE               |
| Autenticación       | Pre-Shared Key      | Clave compartida entre ambos peers                       |
| Grupo DH            | Group 14 (2048-bit) | Intercambio Diffie-Hellman para derivar clave de sesión  |
| Lifetime SA         | 86400 s (24 h)      | Duración del canal IKE antes de renegociar                |
| Pre-Shared Key      | `ITLA2025Arlene`    | Clave idéntica en R1 y R2                                 |

### IKE Fase 2 — IPSec / Profile

| Parámetro            | Valor                  | Descripción                                          |
|------------------------|-------------------------|--------------------------------------------------------|
| Transform Set         | `TS_AES256_SHA256`     | Cifrado ESP-AES-256 + ESP-SHA256-HMAC                 |
| Modo                  | Tunnel                 | Encapsulamiento del paquete IP completo               |
| IPSec Profile         | `IPSEC_PROFILE_VTI`    | Vincula el transform set a la interfaz Tunnel          |
| Aplicación            | `tunnel protection ipsec profile` | Se aplica directamente sobre `Tunnel0`      |
| Lifetime SA           | 3600 s (1 h)            | Duración del túnel de datos antes de renegociar        |

### Interfaz Tunnel (VTI)

| Parámetro            | R1            | R2            |
|------------------------|---------------|---------------|
| IP Tunnel              | 10.0.0.1/30   | 10.0.0.2/30   |
| Tunnel Source          | Ethernet0/0   | Ethernet0/0   |
| Tunnel Destination     | 192.168.19.6  | 192.168.19.5  |
| Tunnel Mode            | ipsec ipv4    | ipsec ipv4    |

> A diferencia del modelo policy-based, **no se usa ACL** para definir tráfico interesante — el enrutamiento hacia `Tunnel0` cumple esa función.

---

## 5. Scripts de Configuración

### R1 — Site A

```cisco
! ══════════════════════════════════════════════════════════════
!  R1 — Site A | IPSec IKEv1 Route-Based VPN (VTI)
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R1

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Cloud
 ip address 192.168.19.5 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 202.50.73.129 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: ISAKMP Policy (IKE Fase 1) ──────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2025Arlene address 192.168.19.6

! ── Paso 2: Transform Set (IKE Fase 2) ──────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 3: IPSec Profile (sustituye al Crypto Map) ─────────
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set security-association lifetime seconds 3600

! ── Paso 4: Interfaz de Túnel Virtual (VTI) ──────────────────
interface Tunnel0
 description Tunel-VPN-hacia-SiteB-R2
 ip address 10.0.0.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.6
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE_VTI

! ── Paso 5: Ruta estática hacia LAN remota vía Tunnel0 ───────
ip route 202.50.73.0 255.255.255.128 Tunnel0
```

### R2 — Site B

```cisco
! ══════════════════════════════════════════════════════════════
!  R2 — Site B | IPSec IKEv1 Route-Based VPN (VTI)
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R2

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Cloud
 ip address 192.168.19.6 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 202.50.73.1 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: ISAKMP Policy (IKE Fase 1) ──────────────────────
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key ITLA2025Arlene address 192.168.19.5

! ── Paso 2: Transform Set (IKE Fase 2) ──────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 3: IPSec Profile (sustituye al Crypto Map) ─────────
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set security-association lifetime seconds 3600

! ── Paso 4: Interfaz de Túnel Virtual (VTI) ──────────────────
interface Tunnel0
 description Tunel-VPN-hacia-SiteA-R1
 ip address 10.0.0.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.5
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE_VTI

! ── Paso 5: Ruta estática hacia LAN remota vía Tunnel0 ───────
ip route 202.50.73.128 255.255.255.128 Tunnel0
```

### Hosts (VPCs en PNetLab)

```bash
# PC1 — Site A
ip 202.50.73.130 255.255.255.128 202.50.73.129

# PC2 — Site B
ip 202.50.73.2 255.255.255.128 202.50.73.1
```

---

## 6. Verificación del Túnel

### Estado de la interfaz Tunnel

```cisco
R1# show interface tunnel 0
```

Salida esperada:

```
Tunnel0 is up, line protocol is up
  Internet address is 10.0.0.1/30
  Tunnel source 192.168.19.5, destination 192.168.19.6
  Tunnel protocol/transport IPSEC/IP
```

> `line protocol is up` confirma que el túnel IPSec está correctamente cifrando y descifrando el tráfico de keepalive.

---

### Estado IKE Fase 1 — ISAKMP SA

```cisco
R1# show crypto isakmp sa
```

Salida esperada:

```
IPv4 Crypto ISAKMP SA
dst              src              state       conn-id  status
192.168.19.6     192.168.19.5     QM_IDLE     1001     ACTIVE
```

---

### Estado IPSec Fase 2 — IPSec SA

```cisco
R1# show crypto ipsec sa
```

Salida esperada (fragmento):

```
interface: Tunnel0
   Crypto map tag: Tunnel0-head-0, local addr 192.168.19.5

   #pkts encaps: 18, #pkts encrypt: 18, #pkts digest: 18
   #pkts decaps: 18, #pkts decrypt: 18, #pkts verify: 18
```

> Nótese que ahora el Crypto Map se genera automáticamente y se asocia a `Tunnel0`, no a la interfaz física.

---

### Verificar la tabla de enrutamiento

```cisco
R1# show ip route static
```

Salida esperada:

```
S    202.50.73.0/25 is directly connected, Tunnel0
```

---

### Conectividad extremo a extremo

```bash
# Desde PC1 hacia PC2 (con túnel activo)
PC1> ping 202.50.73.2

# Desde R1 — ping al peer del túnel
R1# ping 10.0.0.2 source 10.0.0.1
```

Resultado esperado:

```
!!!!!!!!!!
Success rate is 100 percent (10/10)
```

---

### Tabla de Comandos de Verificación

| Comando | Qué verifica |
|---|---|
| `show interface tunnel 0` | Estado de la VTI (up/up indica túnel operativo). |
| `show crypto isakmp sa` | Estado de IKE Fase 1. Debe mostrar `QM_IDLE`. |
| `show crypto ipsec sa` | SAs de Fase 2 asociadas a `Tunnel0` con contadores de paquetes. |
| `show ip route static` | Confirma que la ruta hacia la LAN remota apunta a `Tunnel0`. |
| `show crypto session` | Resumen rápido del estado de la sesión IPSec. |

---

## 7. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.png) | Topología en PNetLab con nombre y matrícula visibles, todos los nodos encendidos. |
| 2 | [Config R1 – Tunnel VTI](evidencias/2.png) | Consola R1: configuración de `Tunnel0` con `tunnel protection ipsec profile`. |
| 3 | [Estado del túnel](evidencias/3.png) | Salida de `show interface tunnel 0` y `show crypto isakmp sa` en estado `up/up` y `QM_IDLE`. |
| 4 | [Ping exitoso](evidencias/4.png) | Ping exitoso de PC1 (`202.50.73.130`) a PC2 (`202.50.73.2`) con túnel activo. |

---

## 8. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](#)**

---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 02: IPSec IKEv1 Route-Based VPN*

</div>
