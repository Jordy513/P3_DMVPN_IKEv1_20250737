# VPN Hub-and-Spoke — DMVPN Fase 2 con IKEv1

### Jordy Jose Rosario Ortiz · Matrícula: 2025-0737

**Seguridad de Redes 2026-C-2 · ITLA**

---

## 📋 Tabla de Contenido

1. [Objetivo del Laboratorio](#1-objetivo-del-laboratorio)
2. [Marco Teórico](#2-marco-teórico)
   - [¿Qué es DMVPN?](#21-qué-es-dmvpn)
   - [Los tres componentes de DMVPN](#22-los-tres-componentes-de-dmvpn)
   - [Las Fases de DMVPN](#23-las-fases-de-dmvpn)
   - [DMVPN Fase 2 vs. Hub-and-Spoke clásico](#24-dmvpn-fase-2-vs-hub-and-spoke-clásico)
   - [Parámetros Criptográficos Utilizados](#25-parámetros-criptográficos-utilizados)
3. [Documentación de la Red](#3-documentación-de-la-red)
   - [Topología](#31-topología)
   - [Tabla de Dispositivos y Direccionamiento IP](#32-tabla-de-dispositivos-y-direccionamiento-ip)
4. [Scripts de Configuración](#4-scripts-de-configuración)
   - [Router R1 — HUB](#41-router-r1--hub)
   - [Router R2 — SPOKE 1](#42-router-r2--spoke-1)
   - [Router R3 — SPOKE 2](#43-router-r3--spoke-2)
   - [Configuración de PCs (Hosts)](#44-configuración-de-pcs-hosts)
5. [Verificación del Túnel](#5-verificación-del-túnel)
6. [Capturas de Pantalla](#6-capturas-de-pantalla)
7. [Consideraciones de Seguridad](#7-consideraciones-de-seguridad)
8. [Video Demostrativo](#8-video-demostrativo)
9. [Referencias](#9-referencias)

---

## 1. Objetivo del Laboratorio

El objetivo de este laboratorio es **implementar y verificar una VPN Hub-and-Spoke punto a multipunto utilizando DMVPN Fase 2 con IKEv1** sobre infraestructura Cisco IOS. A través de esta práctica se busca demostrar:

* La configuración de una interfaz mGRE (Multipoint GRE) en el Hub (R1) y en los Spokes (R2, R3), permitiendo que un único router Hub maneje múltiples túneles dinámicos sin necesidad de una interfaz de túnel por cada Spoke.
* El funcionamiento de NHRP (Next Hop Resolution Protocol) como protocolo de registro y resolución de direcciones, permitiendo que los Spokes se registren ante el Hub y, en Fase 2, descubran las IPs públicas de los demás Spokes para establecer túneles directos Spoke-to-Spoke.
* La protección criptográfica del túnel DMVPN mediante IPSec con IKEv1, manteniendo el soporte de multicast necesario para el enrutamiento dinámico.
* La propagación de rutas mediante EIGRP sobre la nube DMVPN, con la configuración específica de `no ip split-horizon` necesaria en el Hub para que las rutas se propaguen correctamente entre Spokes.
* La verificación de comunicación Spoke-to-Spoke directa (sin pasar por el Hub), demostrando la ventaja central de DMVPN Fase 2 frente al modelo Hub-and-Spoke tradicional.

---

## 2. Marco Teórico

### 2.1 ¿Qué es DMVPN?

DMVPN (Dynamic Multipoint VPN) es una solución de Cisco que permite construir una topología VPN escalable donde múltiples sitios remotos (Spokes) se conectan a un sitio central (Hub) mediante un único túnel lógico, y donde los Spokes pueden además establecer túneles dinámicos directos entre sí sin pasar por el Hub.

A diferencia del modelo Hub-and-Spoke clásico — donde cada par Hub-Spoke requiere su propia interfaz de túnel y, si se desea comunicación directa entre Spokes, ese tráfico debe obligatoriamente pasar por el Hub (hairpinning) — DMVPN resuelve ambos problemas combinando tres tecnologías.

### 2.2 Los tres componentes de DMVPN

* **mGRE (Multipoint GRE):** Extensión de GRE que permite que una sola interfaz de túnel tenga múltiples destinos simultáneos. El Hub usa `tunnel mode gre multipoint` en lugar de declarar un `tunnel destination` fijo — esto es lo que permite que un solo Tunnel0 en el Hub sirva a N Spokes sin crear N interfaces.
* **NHRP (Next Hop Resolution Protocol):** Protocolo que resuelve la dirección IP pública real (NBMA) de un peer a partir de su dirección IP dentro del túnel. Los Spokes se registran ante el Hub (que actúa como NHS — Next Hop Server) anunciando su IP del túnel y su IP pública. Cuando un Spoke necesita alcanzar a otro Spoke, consulta vía NHRP al Hub para obtener su IP pública y así construir un túnel directo.

  > **Requisito crítico para que NHRP se dispare:** el protocolo de enrutamiento del Hub debe anunciar el next-hop **original** de cada Spoke (no reemplazarlo por sí mismo). EIGRP hace "next-hop-self" por defecto en cualquier interfaz — si no se desactiva con `no ip next-hop-self eigrp 1` en el Tunnel0 del Hub, todos los Spokes ven al Hub como next-hop para cualquier ruta remota, el tráfico de datos siempre viaja por el Hub, y la resolución NHRP entre Spokes nunca se dispara — el túnel se comporta como Fase 1 aunque la interfaz sea mGRE.

* **IPSec (en este laboratorio, con IKEv1):** Protege el tráfico de la nube mGRE con cifrado, integridad y autenticación. Se aplica mediante `tunnel protection ipsec profile` sobre la interfaz Tunnel0 — igual que en el laboratorio de GRE over IPSec, pero ahora en modo multipunto.

### 2.3 Las Fases de DMVPN

| Fase | Hub | Spokes | Comunicación Spoke-Spoke |
|---|---|---|---|
| **Fase 1** | mGRE | GRE punto a punto (point-to-point) | ❌ Siempre pasa por el Hub |
| **Fase 2** (este lab) | mGRE | mGRE | ✅ Directa, una vez resuelta vía NHRP |
| **Fase 3** | mGRE + NHRP Redirect | mGRE + NHRP Shortcut | ✅ Directa, con redirección optimizada por el Hub |

En **Fase 2**, tanto el Hub como los Spokes usan interfaces mGRE. Cuando dos Spokes necesitan comunicarse, el Spoke origen consulta al Hub vía NHRP, obtiene la IP pública del Spoke destino, y construye un túnel GRE dinámico directo con él — sin que el tráfico de datos pase por el Hub. El Hub solo participa en el registro NHRP, no en el reenvío del tráfico de datos entre Spokes.

### 2.4 DMVPN Fase 2 vs. Hub-and-Spoke clásico

| Característica | Hub-and-Spoke clásico | DMVPN Fase 2 |
|---|---|---|
| **Interfaces de túnel en el Hub** | Una por cada Spoke (Tunnel1, Tunnel2, ...) | Una sola interfaz mGRE para todos los Spokes |
| **Comunicación Spoke-Spoke** | Siempre vía Hub (doble salto) | Directa, tras resolución NHRP |
| **Escalabilidad al agregar un sitio** | Requiere nueva interfaz de túnel en el Hub | Solo requiere configurar el nuevo Spoke |
| **Latencia Spoke-Spoke** | Mayor (2 saltos cifrados) | Menor (1 salto directo tras la resolución) |
| **Protocolo de resolución** | Ninguno — rutas estáticas | NHRP |
| **Complejidad de configuración del Hub** | Crece linealmente con cada Spoke | Constante, independiente del número de Spokes |

### 2.5 Parámetros Criptográficos Utilizados

| Parámetro | Valor configurado | Propósito |
|---|---|---|
| **Cifrado IKE (Fase 1 ISAKMP)** | AES-256 | Cifrado simétrico del canal IKE |
| **Hash IKE (Fase 1 ISAKMP)** | SHA-256 | Integridad de mensajes IKE |
| **Autenticación** | Pre-Shared Key (PSK) | Autenticación mutua entre Hub y Spokes |
| **Grupo Diffie-Hellman** | Grupo 14 (2048-bit MODP) | Intercambio seguro de clave simétrica |
| **Cifrado ESP (Fase 2 IPSec)** | AES-256 | Cifrado del tráfico de la nube DMVPN |
| **Autenticación ESP (Fase 2 IPSec)** | SHA-256 HMAC | Integridad del tráfico de la nube DMVPN |
| **Modo IPSec** | Transport | mGRE ya encapsula — IPSec solo cifra el payload |
| **Modo del túnel** | `gre multipoint` | Permite múltiples destinos desde una sola interfaz |
| **NHRP Network ID** | 1 | Identificador lógico de la nube DMVPN |
| **NHRP Authentication** | `dmvpnkey` | Autenticación de los mensajes de registro NHRP |
| **Subred del túnel DMVPN** | 10.25.37.0/24 | Espacio de direccionamiento de la nube mGRE |
| **Enrutamiento dinámico** | EIGRP AS 1 | Propagación de rutas LAN entre Hub y Spokes |

---

## 3. Documentación de la Red

### 3.1 Topología

La topología consta de un Hub (R1) y dos Spokes (R2 y R3), todos conectados a una nube común que representa Internet (`192.168.1.0/24`). Cada router tiene su propia subred LAN /26 derivada de la matrícula `20250737`. La nube DMVPN usa la subred `10.25.37.0/24` para las interfaces Tunnel0.

```
                                    [ INTERNET / NET ]
                                    192.168.1.0/24
                                          │
                  ┌───────────────────────┼───────────────────────┐
                  │ e0/0: 192.168.1.10    │ e0/0: 192.168.1.20    │ e0/0: 192.168.1.30
          ┌───────┴───────┐       ┌───────┴───────┐       ┌───────┴───────┐
          │  Router R1    │       │  Router R2    │       │  Router R3    │
          │     HUB       │═══════│   SPOKE 1     │       │   SPOKE 2     │
          │  Tunnel0:     │  mGRE │  Tunnel0:     │       │  Tunnel0:     │
          │  10.25.37.1   │  NHRP │  10.25.37.2   │       │  10.25.37.3   │
          └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
                  │                       ║                       ║
                  │            ╚═══════════════════════════════════╝
                  │              Túnel Spoke-Spoke directo
                  │              (negociado dinámicamente vía NHRP)
                  │ e0/1: 20.25.37.1/26  │ e0/1: 20.25.37.65/26  │ e0/1: 20.25.37.129/26
                  │                       │                       │
          ┌───────┴───────┐       ┌───────┴───────┐       ┌───────┴───────┐
          │      SW1      │       │      SW2      │       │      SW3      │
          └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
                  │                       │                       │
              ┌───┴──┐               ┌───┴──┐                ┌───┴──┐
              │ PC1  │               │ PC2  │                │ PC3  │
              │  .2  │               │ .66  │                │ .130 │
              └──────┘               └──────┘                └──────┘
           20.25.37.0/26          20.25.37.64/26          20.25.37.128/26

  ════════════════════════════════════════════════════════════════════════
  Flujo DMVPN Fase 2 — comunicación Spoke-to-Spoke directa:
    1. PC2 (Spoke1, 20.25.37.66) hace ping a PC3 (Spoke2, 20.25.37.130).
    2. R2 (Spoke1) no conoce la IP pública de R3 — consulta vía NHRP
       al Hub (R1), que actúa como NHS (Next Hop Server).
    3. El Hub responde con la IP pública real de R3 (192.168.1.30).
    4. R2 construye dinámicamente un túnel GRE directo hacia R3
       usando esa IP — sin que el tráfico de datos pase por R1.
    5. IPSec (IKEv1) cifra el tráfico de ese túnel directo.
    6. PC3 recibe el ping. El Hub NO participó en el reenvío de datos.
  ════════════════════════════════════════════════════════════════════════
```

### 3.2 Tabla de Dispositivos y Direccionamiento IP

| Dispositivo | Tipo / Modelo | Interfaz | Dirección IP | Máscara | Rol |
|---|---|---|---|---|---|
| **NET** | Switch/Cloud simulado | — | 192.168.1.0/24 | /24 | Simulación de Internet (NBMA) |
| **R1** | Cisco IOS (Router) | e0/0 | 192.168.1.10 | /24 | WAN — IP pública del Hub |
| | | e0/1 | 20.25.37.1 | /26 | Gateway LAN — Hub |
| | | Tunnel0 | 10.25.37.1 | /24 | mGRE — Hub / NHS |
| **R2** | Cisco IOS (Router) | e0/0 | 192.168.1.20 | /24 | WAN — IP pública Spoke 1 |
| | | e0/1 | 20.25.37.65 | /26 | Gateway LAN — Spoke 1 |
| | | Tunnel0 | 10.25.37.2 | /24 | mGRE — Spoke 1 |
| **R3** | Cisco IOS (Router) | e0/0 | 192.168.1.30 | /24 | WAN — IP pública Spoke 2 |
| | | e0/1 | 20.25.37.129 | /26 | Gateway LAN — Spoke 2 |
| | | Tunnel0 | 10.25.37.3 | /24 | mGRE — Spoke 2 |
| **SW1** | Cisco IOS (Switch L2) | — | — | — | Conmutación LAN Hub |
| **SW2** | Cisco IOS (Switch L2) | — | — | — | Conmutación LAN Spoke 1 |
| **SW3** | Cisco IOS (Switch L2) | — | — | — | Conmutación LAN Spoke 2 |
| **PC1** | Host Linux / VPC | eth0 | 20.25.37.2 | /26 | Host Hub |
| **PC2** | Host Linux / VPC | eth0 | 20.25.37.66 | /26 | Host Spoke 1 |
| **PC3** | Host Linux / VPC | eth0 | 20.25.37.130 | /26 | Host Spoke 2 |

---

## 4. Scripts de Configuración

### 4.1 Router R1 — HUB

```cisco
! ══════════════════════════════════════════════════════
! R1 — HUB | DMVPN Fase 2 con IKEv1
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R1

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-NET
 ip address 192.168.1.10 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-HUB
 ip address 20.25.37.1 255.255.255.192
 no shutdown

! ─── PASO 1: ISAKMP Policy — IKE Fase 1 ───────────────
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! ─── PASO 2: PSK válida para CUALQUIER Spoke ──────────
! En DMVPN se usa una wildcard PSK (0.0.0.0) porque el
! Hub no conoce de antemano las IPs de todos los Spokes.
crypto isakmp key DmvpnITLA2026! address 0.0.0.0 0.0.0.0

! ─── PASO 3: Transform Set en modo TRANSPORTE ─────────
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

! ─── PASO 4: IPSec Profile ─────────────────────────────
crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_DMVPN

! ─── PASO 5: Interfaz mGRE del Hub ─────────────────────
interface Tunnel0
 description DMVPN-HUB-Fase2
 ip address 10.25.37.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint        ! mGRE — sin destination fijo
 tunnel key 20250737                ! Identificador del túnel (matrícula)
 ip nhrp network-id 1
 ip nhrp authentication dmvpnkey
 ip nhrp map multicast dynamic      ! Acepta registros dinámicos de Spokes
 tunnel protection ipsec profile DMVPN_PROFILE
 no shutdown

! ─── PASO 6: EIGRP sobre la nube DMVPN ─────────────────
! no ip split-horizon es OBLIGATORIO en el Hub: sin esto,
! las rutas aprendidas de un Spoke no se reenvían a otro
! Spoke por la misma interfaz (regla normal de split-horizon).
!
! no ip next-hop-self es IGUAL DE OBLIGATORIO en Fase 2:
! sin esto, EIGRP reemplaza el next-hop real de cada Spoke
! por la IP del propio Hub al anunciar la ruta a los demás
! Spokes. Si eso pasa, el tráfico SIEMPRE viaja por el Hub
! (comportamiento de Fase 1) y NHRP nunca se dispara, porque
! los Spokes nunca ven el next-hop real del otro Spoke.
interface Tunnel0
 no ip split-horizon eigrp 1
 no ip next-hop-self eigrp 1

router eigrp 1
 network 20.25.37.0 0.0.0.63       ! LAN local del Hub
 network 10.25.37.0 0.0.0.255      ! Subred de la nube DMVPN
 no auto-summary
```

### 4.2 Router R2 — SPOKE 1

```cisco
! ══════════════════════════════════════════════════════
! R2 — SPOKE 1 | DMVPN Fase 2 con IKEv1
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R2

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-NET
 ip address 192.168.1.20 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-Spoke1
 ip address 20.25.37.65 255.255.255.192
 no shutdown

! ─── PASO 1: ISAKMP Policy — IKE Fase 1 ───────────────
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! ─── PASO 2: PSK — apunta específicamente al Hub ──────
crypto isakmp key DmvpnITLA2026! address 192.168.1.10

! ─── PASO 3: Transform Set en modo TRANSPORTE ─────────
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

! ─── PASO 4: IPSec Profile ─────────────────────────────
crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_DMVPN

! ─── PASO 5: Interfaz mGRE del Spoke ───────────────────
! Igual que en el Hub, el Spoke también usa mGRE en
! Fase 2 — esto es lo que habilita la comunicación
! Spoke-to-Spoke directa.
interface Tunnel0
 description DMVPN-Spoke1-Fase2
 ip address 10.25.37.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 20250737
 ip nhrp network-id 1
 ip nhrp authentication dmvpnkey
 ip nhrp nhs 10.25.37.1                    ! Dirección del Hub en el túnel
 ip nhrp map 10.25.37.1 192.168.1.10       ! Mapeo estático: IP-túnel del Hub -> IP pública
 ip nhrp map multicast 192.168.1.10        ! Para tráfico multicast (EIGRP) hacia el Hub
 tunnel protection ipsec profile DMVPN_PROFILE
 no shutdown

! ─── PASO 6: EIGRP sobre la nube DMVPN ─────────────────
router eigrp 1
 network 20.25.37.64 0.0.0.63      ! LAN local de Spoke 1
 network 10.25.37.0 0.0.0.255      ! Subred de la nube DMVPN
 no auto-summary
```

### 4.3 Router R3 — SPOKE 2

```cisco
! ══════════════════════════════════════════════════════
! R3 — SPOKE 2 | DMVPN Fase 2 con IKEv1
! Jordy Rosario — 20250737 | Seguridad de Redes 2026-C-2
! ══════════════════════════════════════════════════════

hostname R3

! ─── Interfaces físicas ────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-NET
 ip address 192.168.1.30 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-Spoke2
 ip address 20.25.37.129 255.255.255.192
 no shutdown

! ─── PASO 1: ISAKMP Policy — IKE Fase 1 ───────────────
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! ─── PASO 2: PSK — apunta específicamente al Hub ──────
crypto isakmp key DmvpnITLA2026! address 192.168.1.10

! ─── PASO 3: Transform Set en modo TRANSPORTE ─────────
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

! ─── PASO 4: IPSec Profile ─────────────────────────────
crypto ipsec profile DMVPN_PROFILE
 set transform-set TS_DMVPN

! ─── PASO 5: Interfaz mGRE del Spoke ───────────────────
interface Tunnel0
 description DMVPN-Spoke2-Fase2
 ip address 10.25.37.3 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 20250737
 ip nhrp network-id 1
 ip nhrp authentication dmvpnkey
 ip nhrp nhs 10.25.37.1
 ip nhrp map 10.25.37.1 192.168.1.10
 ip nhrp map multicast 192.168.1.10
 tunnel protection ipsec profile DMVPN_PROFILE
 no shutdown

! ─── PASO 6: EIGRP sobre la nube DMVPN ─────────────────
router eigrp 1
 network 20.25.37.128 0.0.0.63     ! LAN local de Spoke 2
 network 10.25.37.0 0.0.0.255      ! Subred de la nube DMVPN
 no auto-summary
```

### 4.4 Configuración de PCs (Hosts)

**PC1 — LAN del Hub (`20.25.37.0/26`)**

```bash
ip 20.25.37.2 255.255.255.192 20.25.37.1
```

**PC2 — LAN de Spoke 1 (`20.25.37.64/26`)**

```bash
ip 20.25.37.66 255.255.255.192 20.25.37.65
```

**PC3 — LAN de Spoke 2 (`20.25.37.128/26`)**

```bash
ip 20.25.37.130 255.255.255.192 20.25.37.129
```

> **Nota:** Si los hosts son VPCs de PNETLab se usa el comando `ip` como se muestra. Si son máquinas Linux se configura con `ip addr add` e `ip route add default via`.

---

## 5. Verificación del Túnel

### 5.1 Verificar el registro NHRP en el Hub

Una vez que los Spokes levantan su interfaz Tunnel0, deben registrarse automáticamente ante el Hub. Verificar desde **R1**:

```cisco
R1# show dmvpn
```

*Salida esperada:*

```
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        T1 - Route Installed, T2 - Nexthop-override
        C - CTS Capable
        # Ent --> Number of NHRP entries with same NBMA peer
        NHS Status: E --> Expecting Replies, R --> Responding, W --> Waiting
        UpDn Time --> Up or Down Time for a Tunnel

==========================================================================
Interface: Tunnel0, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 192.168.1.20         10.25.37.2    UP 00:02:15     D
     1 192.168.1.30         10.25.37.3    UP 00:01:48     D
```

> Ambos Spokes aparecen registrados con `Attrb: D` (Dynamic) y estado `UP`. Esto confirma que el registro NHRP funcionó correctamente.

---

### 5.2 Verificar la caché NHRP en un Spoke

> **Aclaración de direcciones:** `show ip nhrp` solo resuelve direcciones **dentro del espacio del túnel** (`10.25.37.0/24`). La columna `NBMA address` muestra la IP pública/WAN real (`192.168.1.0/24`) asociada a esa IP de túnel. Las LANs (`20.25.37.x`) **nunca aparecen aquí** — esas rutas las aprende EIGRP por separado (sección 5.5), no NHRP.

```cisco
R2# show ip nhrp
```

*Salida esperada antes de comunicación Spoke-Spoke (solo conoce al Hub — entrada estática):*

```
10.25.37.1/32 via 10.25.37.1
   Tunnel0 created 00:03:49, never expire
   Type: static, Flags: used
   NBMA address: 192.168.1.10
```

> `Type: static` y `never expire` son correctos aquí — esta entrada la creó manualmente el comando `ip nhrp map 10.25.37.1 192.168.1.10` en la configuración de R2, por lo que no caduca.

*Salida esperada después de hacer ping hacia el otro Spoke (PC2 → PC3) — aparece una segunda entrada dinámica:*

```
10.25.37.1/32 via 10.25.37.1
   Tunnel0 created 00:03:49, never expire
   Type: static, Flags: used
   NBMA address: 192.168.1.10
10.25.37.3/32 via 10.25.37.3
   Tunnel0 created 00:00:05, expire 01:59:55
   Type: dynamic, Flags: router used
   NBMA address: 192.168.1.30
```

> La segunda entrada — `10.25.37.3` (IP **de túnel** de R3) resuelta hacia `192.168.1.30` (IP **pública/NBMA** de R3) — aparece **solo después** de generar tráfico hacia Spoke2. Tiene `Type: dynamic` y sí expira (`expire 01:59:55`). Esta es la prueba de que NHRP resolvió la IP pública de R3 dinámicamente y se construyó el túnel directo Spoke-to-Spoke, sin pasar por el Hub.
>
> Si tu output solo muestra la primera entrada (estática hacia el Hub) **incluso después de hacer ping hacia la LAN del otro Spoke**, revisa lo siguiente antes de seguir:
>
> 1. Corre `show ip route eigrp` en el Spoke y mira el next-hop de la ruta hacia la LAN remota. Si dice `via 10.25.37.1` (la IP del Hub) en vez de `via 10.25.37.3` (la IP real del otro Spoke), el problema es que falta `no ip next-hop-self eigrp 1` en el `Tunnel0` del Hub — sin ese comando, EIGRP oculta el next-hop real de cada Spoke y el tráfico de datos nunca sale del camino Hub-Spoke, por lo que NHRP nunca se dispara.
> 2. Confirma con `show ip eigrp neighbors` que el Spoke sí tiene adyacencia con el Hub.
> 3. Una vez corregido el next-hop-self en el Hub, vuelve a correr el ping y el `show ip nhrp` — la entrada dinámica debería aparecer en segundos.

---

### 5.3 Verificar el estado de IKE Fase 1 en cada Spoke

```cisco
R1# show crypto isakmp sa
```

*Salida esperada — con ambos Spokes registrados:*

```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
192.168.1.10    192.168.1.20    QM_IDLE        1001    ACTIVE
192.168.1.10    192.168.1.30    QM_IDLE        1002    ACTIVE
```

---

### 5.4 Verificar las IPSec SAs activas

```cisco
R1# show crypto ipsec sa
```

> Debe mostrar dos SAs activas — una por cada Spoke conectado al Hub, ambas con `mode transport`.

---

### 5.5 Verificar la tabla de enrutamiento EIGRP

```cisco
R2# show ip route eigrp
```

*Salida esperada en Spoke 1 — debe ver las LANs del Hub y de Spoke 2:*

```
      20.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
D        20.25.37.0/26 [90/XXXXXX] via 10.25.37.1, 00:05:12, Tunnel0
D        20.25.37.128/26 [90/XXXXXX] via 10.25.37.3, 00:01:30, Tunnel0
```

> Nótese que la ruta hacia `20.25.37.128/26` (LAN de Spoke 2) apunta directamente a `10.25.37.3` — esto confirma que, tras la resolución NHRP, el tráfico ya no necesita pasar por el Hub para llegar a otro Spoke.

---

### 5.6 Verificar comunicación Spoke-to-Spoke directa

Ejecutar desde **PC2** (Spoke 1) hacia **PC3** (Spoke 2):

```bash
PC2> ping 20.25.37.130
```

*Resultado esperado:*

```
84 bytes from 20.25.37.130 icmp_seq=1 ttl=62 time=X ms
84 bytes from 20.25.37.130 icmp_seq=2 ttl=62 time=X ms
84 bytes from 20.25.37.130 icmp_seq=3 ttl=62 time=X ms
```

Para confirmar que el tráfico viaja directo y no por el Hub, repetir `show ip nhrp` en R2 inmediatamente después — debe mostrar la entrada dinámica hacia `10.25.37.3` con `Type: dynamic`.

---

### 5.7 Tabla de comandos de verificación

| Comando | Qué muestra |
|---|---|
| `show dmvpn` | Vista general de la nube DMVPN: peers registrados, estado, tipo (S/D). |
| `show ip nhrp` | Caché NHRP — entradas estáticas (Hub) y dinámicas (Spoke-Spoke resueltas). |
| `show ip nhrp nhs` | Estado del registro ante el NHS (Hub) desde un Spoke. |
| `show crypto isakmp sa` | SAs de Fase 1 — una por cada peer (Hub-Spoke1, Hub-Spoke2, y Spoke-Spoke si aplica). |
| `show crypto ipsec sa` | SAs de Fase 2 — contadores de paquetes por túnel. |
| `show ip route eigrp` | Rutas aprendidas dinámicamente — confirma next-hop directo entre Spokes. |
| `show ip eigrp neighbors` | Vecinos EIGRP formados sobre la nube Tunnel0. |
| `debug nhrp` | Debug del proceso de resolución NHRP en tiempo real. |

---

## 6. Capturas de Pantalla

A continuación se detalla el índice de evidencias correspondientes a las fases de configuración, verificación y funcionamiento de la VPN DMVPN Fase 2, alojadas en la carpeta [`screenshots/`](screenshots/README.md):

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, los tres routers, switches y PCs encendidos. |
| 2 | [`02_config_hub_tunnel.png`](screenshots/02_config_hub_tunnel.png) | Consola de R1 mostrando la interfaz `Tunnel0` con `tunnel mode gre multipoint`, `ip nhrp network-id` y `tunnel protection ipsec profile`. |
| 3 | [`03_config_hub_eigrp_splithorizon.png`](screenshots/03_config_hub_eigrp_splithorizon.png) | Consola de R1 mostrando `no ip split-horizon eigrp 1` aplicado en `Tunnel0` — paso crítico para que las rutas se propaguen entre Spokes. |
| 4 | [`04_config_spoke1_nhrp.png`](screenshots/04_config_spoke1_nhrp.png) | Consola de R2 mostrando `ip nhrp nhs`, `ip nhrp map` apuntando al Hub y el resto de la configuración de la interfaz mGRE. |
| 5 | [`05_config_spoke2_completa.png`](screenshots/05_config_spoke2_completa.png) | Configuración completa de R3 (Spoke 2) mostrando la simetría con R2. |
| 6 | [`06_show_dmvpn_hub.png`](screenshots/06_show_dmvpn_hub.png) | Salida de `show dmvpn` en R1 mostrando ambos Spokes registrados con estado `UP` y `Attrb: D`. |
| 7 | [`07_isakmp_sa_dos_peers.png`](screenshots/07_isakmp_sa_dos_peers.png) | Salida de `show crypto isakmp sa` en R1 mostrando dos SAs activas en `QM_IDLE` — una por cada Spoke. |
| 8 | [`08_ping_spoke_a_spoke.png`](screenshots/08_ping_spoke_a_spoke.png) | Ping exitoso desde PC2 (`20.25.37.66`) hacia PC3 (`20.25.37.130`) — comunicación directa entre Spokes. |
| 9 | [`09_nhrp_dinamico_post_ping.png`](screenshots/09_nhrp_dinamico_post_ping.png) | Salida de `show ip nhrp` en R2 mostrando la entrada dinámica hacia `10.25.37.3` creada después del ping — evidencia de la resolución NHRP Spoke-to-Spoke. |
| 10 | [`10_eigrp_route_nexthop_directo.png`](screenshots/10_eigrp_route_nexthop_directo.png) | Salida de `show ip route eigrp` en R2 mostrando la ruta hacia la LAN de Spoke 2 con next-hop directo `10.25.37.3` (no vía el Hub). |

---

## 7. Consideraciones de Seguridad

### 7.1 Riesgo de la PSK wildcard en el Hub

En este laboratorio, el Hub usa una PSK wildcard (`crypto isakmp key DmvpnITLA2026! address 0.0.0.0 0.0.0.0`) porque no se conoce de antemano la IP de cada Spoke que pueda unirse. Esto es **necesario para la escalabilidad de DMVPN**, pero tiene una implicación de seguridad importante: cualquier dispositivo que conozca la PSK puede intentar registrarse como Spoke.

En producción, esto se mitiga con:

* Certificados digitales (PKI) en lugar de PSK — cada Spoke se autentica con su propio certificado.
* ACLs en la interfaz WAN del Hub limitando qué rangos de IP pueden iniciar negociación IKE.
* Migración a IKEv2 con autenticación EAP para Spokes dinámicos.

### 7.2 Hardening recomendado

```cisco
! Limitar el número de registros NHRP simultáneos por Hub (anti-DoS)
interface Tunnel0
 ip nhrp max-send 100 every 10

! Forzar Perfect Forward Secrecy
crypto ipsec profile DMVPN_PROFILE
 set pfs group14

! Restringir el acceso WAN del Hub solo a los Spokes conocidos
ip access-list extended ACL_WAN_DMVPN
 permit udp 192.168.1.0 0.0.0.255 host 192.168.1.10 eq 500
 permit esp 192.168.1.0 0.0.0.255 host 192.168.1.10
 deny ip any any log
```

### 7.3 Limitación de IKEv1 en DMVPN

Este laboratorio usa IKEv1 por requerimiento del curso, pero en producción se recomienda **DMVPN con IKEv2 (también llamado FlexVPN)**, que ofrece:

* Negociación más rápida con muchos Spokes simultáneos uniéndose.
* Mejor resistencia a ataques de DoS durante el registro masivo de Spokes.
* Soporte de autenticación EAP para Spokes con identidades dinámicas.

---

## 8. Video Demostrativo

🎥 **[Ver demostración en YouTube](https://youtu.be/n4JQ1k3DX7Q)**

**Duración:** 6:49

**Contenido del video:**

* ✅ Topología funcional en PNETLab con nombre completo `Jordy Rosario — 20250737` visible.
* ✅ Reloj del sistema operativo visible evidenciando fecha y hora actual.
* ✅ Rostro y voz del autor realizando la explicación técnica del laboratorio.
* ✅ Demostración de los scripts de configuración aplicados en R1 (Hub), R2 y R3 (Spokes).
* ✅ Verificación de `show dmvpn` mostrando ambos Spokes registrados.
* ✅ Verificación de `show crypto isakmp sa` mostrando las SAs activas.
* ✅ Ping desde PC2 hacia PC3 (Spoke-to-Spoke) y verificación de `show ip nhrp` mostrando la entrada dinámica resultante.
* ✅ Verificación de `show ip route eigrp` mostrando el next-hop directo entre Spokes.

---

## 9. Referencias

* Cisco Systems. (2024). *Dynamic Multipoint VPN (DMVPN) Design Guide*.
* Cisco Systems. (2024). *Cisco IOS Security Configuration Guide: Secure Connectivity — DMVPN*.
* Luciani, J. et al. (1998). *RFC 2332 — NBMA Next Hop Resolution Protocol (NHRP)*. IETF.
* Harkins, D. & Carrel, D. (1998). *RFC 2409 — The Internet Key Exchange (IKEv1)*. IETF.
* Hanks, S. et al. (1994). *RFC 1701 — Generic Routing Encapsulation (GRE)*. IETF.
* Doraswamy, N. & Harkins, D. (2003). *IPSec: The New Security Standard for the Internet, Intranets, and Virtual Private Networks (2nd Ed.)*. Prentice Hall.
